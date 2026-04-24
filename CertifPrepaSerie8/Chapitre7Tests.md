
## 🧪 Chapitre 7 : Tests – Unitaires, intégration, performance, résilience  
*(Version ultra-détaillée – 5 fois plus de contenu que la précédente)*

---

### 1. Pourquoi tester Kafka ?

Les applications Kafka sont des systèmes **distribués**, **asynchrones** et souvent **stateful**. Les tests doivent couvrir :

- La logique métier des streams
- La gestion des états (state stores)
- Les comportements de reprise (failover)
- Les performances sous charge
- Les interactions avec Connect, Schema Registry, etc.

📌 **Objectif** : détecter les erreurs de sérialisation, les problèmes de fenêtrage, les fuites d’état, les rebalances intempestives ou les corruptions de données.

💡 **Astuce pro** :  
Adoptez une pyramide de tests :  
- **Unitaires** (nombreux, rapides) : `TopologyTestDriver`, mocks.  
- **Intégration** (moins nombreux) : Embedded Kafka ou Testcontainers.  
- **End‑to‑end** (rares) : cluster réel, environnements de staging.

---

### 2. Tests unitaires avec TopologyTestDriver (Kafka Streams)

#### 2.1. Principes

`TopologyTestDriver` exécute une topologie **en mémoire** sans broker réel. Il simule le temps, les fenêtres, les state stores, mais pas les appels externes (ex: bases de données, API). Idéal pour tester :

- Les transformations (map, filter, flatMap)
- Les agrégations fenêtrées
- Les jointures KStream‑KTable, KStream‑KStream
- Les suppressions (`suppress`)

#### 2.2. Configuration et mise en œuvre

```java
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.TopologyTestDriver;
import org.apache.kafka.streams.TestInputTopic;
import org.apache.kafka.streams.TestOutputTopic;
import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.KTable;
import org.apache.kafka.streams.kstream.Materialized;
import org.apache.kafka.streams.state.KeyValueStore;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.Properties;
import static org.assertj.core.api.Assertions.assertThat;

class MyStreamsTest {

    private TopologyTestDriver testDriver;
    private TestInputTopic<String, String> inputTopic;
    private TestOutputTopic<String, String> outputTopic;

    @BeforeEach
    void setUp() {
        StreamsBuilder builder = new StreamsBuilder();
        
        // Topologie simple : majuscule sur les valeurs
        KStream<String, String> source = builder.stream("input");
        source.mapValues(String::toUpperCase).to("output");
        
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "test-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "dummy");
        
        testDriver = new TopologyTestDriver(builder.build(), props);
        
        inputTopic = testDriver.createInputTopic("input", 
            Serdes.String().serializer(), 
            Serdes.String().serializer());
        outputTopic = testDriver.createOutputTopic("output",
            Serdes.String().deserializer(),
            Serdes.String().deserializer());
    }

    @AfterEach
    void tearDown() {
        testDriver.close();
    }

    @Test
    void testUpperCase() {
        inputTopic.pipeInput("key1", "hello");
        assertThat(outputTopic.readValue()).isEqualTo("HELLO");
        assertThat(outputTopic.isEmpty()).isTrue();
    }
}
```

💡 **Astuce pro** :  
- Utilisez `pipeInput()` avec une horloge simulée : `inputTopic.pipeInput("key", "value", 1_000L)` (timestamps en millisecondes).  
- Pour les fenêtres, avancez le temps avec `testDriver.advanceWallClockTime(Duration.ofMinutes(1))`.  
- Vérifiez l’état des state stores via `testDriver.getKeyValueStore("store-name")`.

#### 2.3. Tester les fenêtres et le temps

```java
@Test
void testHoppingWindow() {
    // Envoyer deux messages à 0ms et 5000ms
    inputTopic.pipeInput("k", "A", 0L);
    inputTopic.pipeInput("k", "B", 5_000L);
    
    // Avancer le temps pour déclencher l'émission
    testDriver.advanceWallClockTime(Duration.ofSeconds(10));
    
    // Vérifier les résultats de la fenêtre (ex: agrégation)
    var result = outputTopic.readKeyValuesToList();
    assertThat(result).hasSize(...);
}
```

💡 **Astuce pro** :  
- `advanceWallClockTime` ne déclenche pas directement les fenêtres ; il faut aussi que le `commit.interval` soit écoulé (simulé).  
- Utilisez `testDriver.advanceWallClockTime` **avant** de lire le topic de sortie.

#### 2.4. Tester les state stores (interactive queries)

```java
@Test
void testStateStore() {
    // Après avoir alimenté la topologie
    KeyValueStore<String, Long> store = testDriver.getKeyValueStore("count-store");
    Long count = store.get("key1");
    assertThat(count).isEqualTo(3L);
}
```

💡 **Astuce pro** :  
- Le nom du state store doit correspondre à celui passé dans `Materialized.as("store-name")`.  
- Les stores sont réinitialisés entre chaque test (nouvelle instance de `TopologyTestDriver`).

---

### 3. Tests d’intégration avec Embedded Kafka

#### 3.1. Avantages et limites

Embedded Kafka démarre un vrai broker (ou plusieurs) dans la JVM, ce qui permet de tester :
- Les interactions producteur‑consommateur réelles
- Les commits d’offsets
- Les rebalances
- Les transactions

**Limites** : plus lent que `TopologyTestDriver` (quelques secondes par test).

#### 3.2. Utiliser `kafka-junit` (déprécié, mais simple)

```java
import kafka.junit.jupiter.KafkaTest;
import kafka.junit.jupiter.KafkaCluster;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@KafkaTest
class IntegrationTest {

    @KafkaCluster
    private KafkaCluster cluster;

    private KafkaProducer<String, String> producer;
    private KafkaConsumer<String, String> consumer;

    @BeforeEach
    void setUp() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, cluster.bootstrapServers());
        producer = new KafkaProducer<>(props);
        consumer = new KafkaConsumer<>(props);
    }

    @Test
    void testProduceConsume() {
        producer.send(new ProducerRecord<>("topic", "key", "value"));
        producer.flush();
        consumer.subscribe(List.of("topic"));
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(5));
        assertThat(records).hasSize(1);
    }
}
```

💡 **Astuce pro** :  
- Préférez `clustertest` ou plutôt **Testcontainers** (standard moderne).  
- L’Embedded Kafka ne supporte pas toutes les propriétés des vrais brokers (certaines configs avancées peuvent échouer).

#### 3.3. Testcontainers Kafka (recommandé)

```java
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

@Testcontainers
class KafkaContainerTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

    @Test
    void testWithRealBroker() {
        String bootstrapServers = kafka.getBootstrapServers();
        // utiliser le producteur/consommateur normal
    }
}
```

💡 **Astuce pro** :  
- Testcontainers peut être lent au premier démarrage (téléchargement d’image).  
- Pour accélérer, configurez un réseau partagé et un cache.  
- Exemple avec plusieurs brokers et KRaft : utilisez `KafkaContainer` avec `KRAFT`.

---

### 4. Tests de performance (load testing)

#### 4.1. Outils intégrés à Kafka

- `kafka-producer-perf-test` : débit producteur.
- `kafka-consumer-perf-test` : débit consommateur.

Exemple pour mesurer le débit d’un pipeline de streams (indirect) :
```bash
# Produire 5 millions de messages
kafka-producer-perf-test --topic input-topic --num-records 5000000 --record-size 1024 --throughput 50000 --producer-props bootstrap.servers=localhost:9092

# Consommer via l’application de stream (à lancer à côté), puis mesurer le débit de sortie
kafka-consumer-perf-test --topic output-topic --messages 5000000 --broker-list localhost:9092
```

💡 **Astuce pro** :  
- Lancez le producteur de test et l’application Streams sur des machines différentes pour éviter les interférences.  
- Utilisez `--throughput -1` pour un débit maximal (pas de throttling).

#### 4.2. Apache JMeter ou Gatling

Créez un script JMeter avec :
- `Kafka Producer Sampler` (plugin) ou un `Java Request` encapsulant un producteur.
- Mesurez la latence moyenne, le taux d’erreur, le débit.

📌 **Exemple de configuration JMeter** :
- Thread group : 10 threads, ramp-up 1 min, loop count ∞.
- Sampler : envoi de messages avec clé aléatoire.
- Ajoutez un listener `Aggregate Graph`.

💡 **Astuce pro** :  
- Pour les tests de charge sur Kafka Streams, injectez des données avec un producteur et utilisez `kafka-consumer-groups` pour surveiller le lag.

#### 4.3. Simuler des pannes (Chaos Engineering)

```bash
# Tuer un broker aléatoire
kill -9 $(pgrep -f "kafka.Kafka" | head -1)

# Simuler une latence réseau (avec tc sur Linux)
sudo tc qdisc add dev eth0 root netem delay 100ms

# Couper le réseau partiellement (iptables)
sudo iptables -A INPUT -p tcp --dport 9092 -j DROP
```

💡 **Astuce pro** :  
- Utilisez **Chaos Mesh** ou **Gremlin** pour automatiser les scénarios de panne.  
- Vérifiez que l’application de stream reprend sans perte de données (transaction log et state store).

---

### 5. Tests de résilience – exactly‑once et reprises

#### 5.1. Simulation d’un crash de consommateur

Dans un test unitaire avec `TopologyTestDriver`, impossible (pas de consommation réelle). Donc avec `Testcontainers` :

```java
@Test
void testConsumerCrashAndRestore() {
    // Démarrer l’application (ex: Streams, ou un consommateur simple)
    // Créer un thread pour l’exécuter
    // Tuer le thread (ou stopper le Streams)
    // Redémarrer avec le même application.id et vérifier les offsets
}
```

💡 **Astuce pro** :  
- Pour les streams, activez `processing.guarantee=exactly_once_v2` et vérifiez qu’aucun message n’est traité deux fois après redémarrage.  
- Utilisez `kafka-consumer-groups --describe` pour observer les offsets.

#### 5.2. Tests de reprise sur panne d’un state store

Impossible avec `TopologyTestDriver`. Avec Embeddable (ou Testcontainers) :
- Lancer l’application de stream.
- Forcer l’arrêt d’un pod (via docker stop).
- Redémarrer et vérifier que les agrégations reprennent au bon offset.

📌 **Vérification** : les résultats après redémarrage doivent être identiques à ceux sans panne.

💡 **Astuce pro** :  
- Activez les réplicas en attente (`num.standby.replicas=1`) pour réduire le temps de récupération.  
- Mesurez le temps de récupération (restauration du state store) via les logs.

---

### 6. Tests de Connecteurs (Source/Sink)

#### 6.1. Test d’un source connector sans système externe

Utiliser **`Mockito`** pour mocker `SourceContext` ou les `offset` storage.

```java
@Test
void testSourceTaskPoll() {
    MySourceTask task = new MySourceTask();
    Map<String, String> config = Map.of("my.config", "value");
    task.initialize(new MockSourceContext());
    task.start(config);
    
    List<SourceRecord> records = task.poll();
    assertThat(records).hasSize(1);
}
```

💡 **Astuce pro** :  
- Pour les connecteurs JDBC, utilisez une base de données H2 en mémoire (fichier `jdbc:h2:mem:test`).  
- Pour Debezium, déployez un conteneur MySQL/PostgreSQL aux côtés de Kafka (Testcontainers).

#### 6.2. Test d’un sink connector avec Embedded Kafka

```java
@Test
void testSinkTaskPut() throws Exception {
    // Lancer un cluster embeddé
    EmbeddedKafkaBroker broker = new EmbeddedKafkaBroker(1);
    broker.afterPropertiesSet();
    
    // Créer un topic
    broker.createTopic("sink-input");
    
    // Configurer la tâche sink
    MySinkTask task = new MySinkTask();
    task.start(configMap);
    
    // Produire un message dans le topic
    try (KafkaProducer<String, String> producer = ...) {
        producer.send(new ProducerRecord<>("sink-input", "key", "value")).get();
    }
    
    // Exécuter la tâche (généralement le framework appelle put())
    List<SinkRecord> records = ...; // à simuler ou récupérer via un consommateur mocké
    task.put(records);
    
    // Vérifier que le système externe (mocké) a reçu la donnée
}
```

💡 **Astuce pro** :  
- Pour les sink, utilisez un **mock** du système externe (ex: `MockElasticsearchClient`).  
- Testez l’écriture par lot et les erreurs (DLQ) avec des enregistrements invalides.

---

### 7. Tests end‑to‑end (E2E) – environnement de staging

L’objectif est de valider l’ensemble de la pipeline : producteur → Kafka → Streams → Connecteur sink → base cible.

📌 **Exemple de pipeline** :  
- Source : API REST → producteur.  
- Transformations : Streams avec fenêtrage.  
- Sink : JDBC dans PostgreSQL.  

**Démarche** :  
1. Démarrer un cluster Kafka complet (KRaft ou ZooKeeper) via Docker Compose.  
2. Y ajouter les connecteurs nécessaires (Debezium, JDBC).  
3. Lancer l’application de stream.  
4. Injecter des données de test (via un script Python ou `kafka-console-producer`).  
5. Vérifier le résultat dans PostgreSQL (via des requêtes SQL).  

💡 **Astuce pro** :  
- Automatisez les tests E2E dans la CI (GitLab CI, GitHub Actions) avec des conteneurs Docker temporaires.  
- Utilisez des jeux de données déterministes pour comparer les résultats attendus.  
- Mesurez le temps de bout en bout (latence).

---

### 8. Tableau récapitulatif des outils de test

| Type | Outil | Vitesse | Fidélité | Cas d’usage |
|------|-------|---------|----------|-------------|
| Unitaire | `TopologyTestDriver` | Très rapide | Haute (sauf I/O externe) | Logique de transformation, fenêtres, state stores |
| Unitaire (clients) | `Mockito` + producteur mocké | Rapide | Moyenne (pas de vraie réseau) | Tests de callbacks, sérialisation |
| Intégration | Testcontainers Kafka | Lente (démarrage) | Haute (vrai broker) | Transactions, exactly‑once, rebalances |
| Intégration | Embedded Kafka | Moyenne | Haute (mais certaines limites) | Scénarios simples avec peu de partitions |
| Performance | `kafka-producer-perf-test` | – | Réelle | Benchmarking, capacité |
| Résilience | Chaos Mesh, Gremlin | – | Réelle | Pannes de brokers, réseau, disque |

---

### 9. Bonnes pratiques générales

- **Ne pas tester les performances avec `TopologyTestDriver`** (pas de vrais I/O).  
- **Isoler les tests unitaires** : un test par fonction de la topologie.  
- **Utiliser des IDs fixes** dans les tests (`application.id = "test-app"`) pour éviter les conflits entre tests.  
- **Nettoyer les topics internes** après les tests d’intégration (`embeddedKafka.clear()`).  
- **Versionner les jeux de données de test** (fichiers JSON, Avro) dans le dépôt.  
- **Tester les évolutions de schéma** (Schema Registry) avec des clients mockés.

📌 **Exemple de test d’évolution de schéma** :  
```java
// Envoyer un message avec un nouveau champ optionnel
// Vérifier que l’application de stream ne crash pas (tolérance)
```

💡 **Astuce pro** :  
- Pour les tests de charge, surveillez le lag, l’utilisation CPU/mémoire, et l’espace disque.  
- Ajoutez un **test de brouillard** (`fog test`) : réduisez le débit aléatoirement pour vérifier le backpressure.

---

## Conclusion du chapitre 7 – points clés

- **`TopologyTestDriver`** est votre meilleur ami pour les tests unitaires de Streams (rapide, fiable).  
- **Testcontainers** est la solution moderne pour les tests d’intégration (real brokers, connecteurs).  
- **Les tests de performance** nécessitent des outils spécifiques (`perf-test`, JMeter).  
- **Les tests de résilience** simulent des pannes (kill broker, latence réseau) pour vérifier exactly‑once.  
- **Ne négligez pas les tests de connecteurs** (source/sink) avec des mocks ou des conteneurs.

Avec ce chapitre, vous disposez d’une boîte à outils complète pour tester toutes les facettes d’un système Kafka, de l’unité à l’échelle de la production. 🚀
