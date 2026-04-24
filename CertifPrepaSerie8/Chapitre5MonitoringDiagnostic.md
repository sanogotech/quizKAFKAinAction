
# 📊 Chapitre 5 : Monitoring, CLI et JMX – Outils avancés du quotidien  
*(Version ultra-détaillée – 5 fois plus de contenu que la précédente)*

---

### 1. Les trois piliers du monitoring Kafka

Un bon monitoring repose sur :

- **Métriques JMX** : indicateurs internes (débit, latence, état des partitions, etc.)
- **Logs** : événements systèmes, erreurs, changements de leadership
- **Commandes CLI** : diagnostics interactifs, réparations, reprises

📌 **Objectif** : détecter les anomalies avant qu’elles n’impactent les clients (producteurs/consommateurs).

💡 **Astuce pro** :  
Centralisez les métriques dans Prometheus + Grafana, et les logs dans l’ELK stack (Elasticsearch, Logstash, Kibana) ou Loki.

---

### 2. Métriques JMX – Les yeux dans le moteur

#### 2.1. Activer JMX sur les brokers et clients

**Pour un broker Kafka** (fichier `kafka-server-start.sh` ou `KAFKA_JMX_OPTS`) :
```bash
export KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"
```

**Pour un client (producteur/consommateur)** : ajoutez les mêmes options JVM.

💡 **Astuce pro** :  
En production, activez l’authentification et SSL pour JMX (`-Dcom.sun.management.jmxremote.authenticate=true` + fichiers de mots de passe).

#### 2.2. Métriques brokers essentielles

| Domaine | MBean | Description | Seuil d’alerte |
|---------|-------|-------------|----------------|
| **Réplication** | `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions` | Partitions dont le nombre de réplicas disponibles est inférieur au facteur de réplication | > 0 |
| **ISR** | `kafka.controller:type=ControllerStats,name=ShrinkingISRRate` | Taux de sortie de l’ISR (par seconde) | > 0,1 |
| **Leadership** | `kafka.controller:type=ControllerStats,name=LeaderElectionRate` | Élections de leader par seconde | > 0,1 (sauf après panne) |
| **File d’attente** | `kafka.network:type=RequestMetrics,name=RequestQueueSize` | Taille de la file des requêtes réseau | > 0 (signification surcharge) |
| **Latence** | `kafka.network:type=RequestMetrics,name=LocalTimeMs` | Temps passé dans le broker (hors I/O disque) | > 100 ms |
| **Disque** | `kafka.log:type=LogFlushStats,name=FlushTimeMs` | Temps des fsync disque | > 1 s |
| **Débit** | `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec` | Octets entrants par seconde | Variation brutale |
| **Controller actif** | `kafka.controller:type=KafkaController,name=ActiveControllerCount` | 1 si ce broker est le contrôleur | Doit être 1 sur exactement un broker |

💡 **Astuce pro** :  
- Utilisez `jconsole` ou `jvisualvm` pour explorer les MBeans de façon interactive.  
- Pour une collecte automatisée, le **Prometheus JMX Exporter** est la référence.

#### 2.3. Métriques producteurs

| MBean | Description | Seuil |
|-------|-------------|-------|
| `kafka.producer:type=producer-metrics,client-id=...` `request-latency-avg` | Latence moyenne des requêtes | > 100 ms |
| `kafka.producer:type=producer-metrics` `record-error-rate` | Taux d’erreurs (par seconde) | > 0 |
| `kafka.producer:type=producer-metrics` `buffer-available-bytes` | Tampon libre | < 10% du total |
| `kafka.producer:type=producer-topic-metrics` `byte-rate` | Débit par topic | Suivre l’évolution |

💡 **Astuce pro** :  
- `record-error-rate` > 0 est critique ; inspectez les logs du producteur.  
- `buffer-available-bytes` bas → augmentez `buffer.memory`.

#### 2.4. Métriques consommateurs

| MBean | Description | Seuil |
|-------|-------------|-------|
| `kafka.consumer:type=consumer-fetch-manager-metrics,name=records-lag-max` | Lag maximum parmi toutes les partitions | > 10 000 |
| `kafka.consumer:type=consumer-coordinator-metrics,name=assigned-partitions` | Nombre de partitions assignées | Variation brutale (rebalance) |
| `kafka.consumer:type=consumer-metrics` `records-consumed-rate` | Taux de consommation | Comparer au débit du topic |

💡 **Astuce pro** :  
- Le lag est la métrique la plus importante pour les applications de streaming.  
- Utilisez `kafka-consumer-groups` (CLI) pour une vue rapide.

---

### 3. Commandes CLI – La boîte à outils indispensable

Toutes les commandes ci-dessous utilisent l’option `--bootstrap-server` (obligatoire en KRaft). Remplacez `localhost:9092` par vos brokers.

#### 3.1. Gestion des topics

```bash
# Lister les topics
kafka-topics --bootstrap-server localhost:9092 --list

# Décrire un topic (partitions, réplicas, ISR, leader)
kafka-topics --bootstrap-server localhost:9092 --describe --topic mon-topic

# Décrire tous les topics (utile pour chercher les déséquilibres)
kafka-topics --bootstrap-server localhost:9092 --describe --under-replicated-partitions

# Créer un topic (3 partitions, réplication 3)
kafka-topics --bootstrap-server localhost:9092 --create --topic nouveau --partitions 3 --replication-factor 3

# Modifier le nombre de partitions (uniquement augmentation)
kafka-topics --bootstrap-server localhost:9092 --alter --topic mon-topic --partitions 10

# Supprimer un topic (si delete.topic.enable=true)
kafka-topics --bootstrap-server localhost:9092 --delete --topic mon-topic
```

💡 **Astuce pro** :  
- Vérifiez toujours `under-replicated-partitions` après la création.  
- Une augmentation de partitions ne répartit pas les anciennes données ; utilisez `kafka-reassign-partitions` pour équilibrer.

#### 3.2. Groupes de consommateurs et lag

```bash
# Lister les groupes
kafka-consumer-groups --bootstrap-server localhost:9092 --list

# Voir le lag d’un groupe (avec détails par partition)
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group mon-groupe

# Version avec plus de détails (membres, affectation)
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group mon-groupe --members --verbose

# Réinitialiser les offsets au début (ex: rejeu)
kafka-consumer-groups --bootstrap-server localhost:9092 --group mon-groupe --reset-offsets --to-earliest --topic mon-topic --execute

# Réinitialiser à un timestamp spécifique (millisecondes depuis epoch)
kafka-consumer-groups --bootstrap-server localhost:9092 --group mon-groupe --reset-offsets --to-datetime 2025-01-01T00:00:00.000Z --topic mon-topic --execute

# Supprimer un groupe (libère les offsets)
kafka-consumer-groups --bootstrap-server localhost:9092 --delete --group mon-groupe
```

💡 **Astuce pro** :  
- Le lag est la colonne `LAG` dans la sortie. Un lag qui augmente = consommateur trop lent.  
- `--reset-offsets` nécessite que le groupe soit `inactif` (aucun consommateur connecté).  
- Pour un rejeu sélectif, utilisez `kafka-console-consumer` avec `--offset` puis repositionnez manuellement.

#### 3.3. Producteurs et consommateurs de test

```bash
# Producteur rapide (stdin vers topic)
kafka-console-producer --bootstrap-server localhost:9092 --topic test-topic --property "key.separator=-" --property "parse.key=true"

# Consommateur en temps réel (affichage immédiat)
kafka-console-consumer --bootstrap-server localhost:9092 --topic test-topic --from-beginning

# Consommateur avec affichage des clés et en-têtes
kafka-console-consumer --bootstrap-server localhost:9092 --topic test-topic --property print.key=true --property print.headers=true

# Performance – producteur
kafka-producer-perf-test --topic perf --num-records 1000000 --record-size 100 --throughput 10000 --producer-props bootstrap.servers=localhost:9092 acks=1

# Performance – consommateur
kafka-consumer-perf-test --topic perf --messages 1000000 --broker-list localhost:9092 --timeout 60000
```

💡 **Astuce pro** :  
- Le producteur de test (`kafka-producer-perf-test`) accepte toutes les configurations producteur via `--producer-props`.  
- Pour mesurer la latence end‑to‑end, utilisez `kafka-verifiable-producer` et `kafka-verifiable-consumer`.

#### 3.4. Configuration et métadonnées

```bash
# Modifier une configuration dynamique (ex: rétention d’un topic)
kafka-configs --bootstrap-server localhost:9092 --alter --entity-type topics --entity-name mon-topic --add-config retention.ms=86400000

# Supprimer une configuration (revient à la valeur par défaut)
kafka-configs --bootstrap-server localhost:9092 --alter --entity-type topics --entity-name mon-topic --delete-config retention.ms

# Voir toutes les configurations (y compris celles par défaut)
kafka-configs --bootstrap-server localhost:9092 --describe --entity-type topics --entity-name mon-topic --all

# Décrire les configurations broker
kafka-configs --bootstrap-server localhost:9092 --describe --entity-type brokers --entity-default
```

💡 **Astuce pro** :  
- Les modifications via `kafka-configs` sont dynamiques (pas besoin de redémarrer).  
- Utilisez `--all` avec prudence car la sortie est très volumineuse.

#### 3.5. Outils de diagnostique avancé

```bash
# Inspecter un segment de log (affiche les offsets, messages)
kafka-dump-log --files /var/lib/kafka/data/topic-0/00000000000000000000.log

# Version détaillée avec every message
kafka-dump-log --files /var/lib/kafka/data/topic-0/00000000000000000000.log --deep-iteration

# Vérifier la réplication (divergences entre réplicas)
kafka-replica-verification --bootstrap-server localhost:9092 --report-interval-ms 5000

# Réassignation des partitions – générer un plan d’équilibrage
kafka-reassign-partitions --bootstrap-server localhost:9092 --topics-to-move-json-file topics.json --generate --broker-list "0,1,2"

# Exécuter le plan
kafka-reassign-partitions --bootstrap-server localhost:9092 --reassignment-json-file reassign.json --execute

# Vérifier la progression
kafka-reassign-partitions --bootstrap-server localhost:9092 --reassignment-json-file reassign.json --verify

# Élection manuelle du leader préféré (rééquilibrage des leaders)
kafka-leader-election --bootstrap-server localhost:9092 --election-type preferred --all-topic-partitions

# Lister les features (KRaft)
kafka-features --bootstrap-server localhost:9092 list
```

💡 **Astuce pro** :  
- `kafka-replica-verification` est très utile pour détecter des divergences silencieuses (corruption).  
- La réassignation peut prendre du temps ; utilisez `--verify` pour la progression.  
- Prévoyez des **throttles** (`--throttle`) pour éviter de saturer le réseau.

---

### 4. CLI avancée – Scénarios de résolution de problèmes

#### 4.1. Un consommateur est bloqué, lag énorme

```bash
# Voir le lag actuel
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group mon-groupe

# Si un consommateur est mort, on peut le supprimer du groupe
kafka-consumer-groups --bootstrap-server localhost:9092 --delete --group mon-groupe

# Redémarrer le consommateur avec les offsets précédents (perte possible)
# Ou réinitialiser les offsets pour rejouer
kafka-consumer-groups --bootstrap-server localhost:9092 --group mon-groupe --reset-offsets --to-earliest --topic mon-topic --execute
```

#### 4.2. Un topic a des partitions sous‑répliquées

```bash
kafka-topics --bootstrap-server localhost:9092 --describe --under-replicated-partitions
# Cela montre les partitions où ISR < replication.factor

# Vérifier les brokers morts
kafka-broker-api-versions --bootstrap-server localhost:9092 | grep -v "SUCCESS"

# Forcer la réplication (réassignation) ou attendre le retour du broker
```

#### 4.3. Un producteur reçoit `NotLeaderForPartition`

```bash
# Chercher le véritable leader
kafka-topics --bootstrap-server localhost:9092 --describe --topic mon-topic

# Si le leader est absent, provoquer une élection
kafka-leader-election --bootstrap-server localhost:9092 --election-type preferred --topic mon-topic --partition 0
```

💡 **Astuce pro** :  
- Pour les scénarios complexes, le **Kafka Exporter** (Prometheus) couplé à Grafana donne une vision temps réel.

---

### 5. Logs – Que surveiller et comment les interpréter

#### 5.1. Logs broker – emplacements et niveaux

- Fichier : `logs/server.log` (par défaut).  
- Niveau par défaut : `INFO`. Pour le debug: `DEBUG` dans `log4j.properties`.

**Messages importants** :
- `[Controller id=X]` : actions du contrôleur (élections, réassignations).  
- `[ReplicaFetcherThread]` : réplication, retards.  
- `[SocketServer]` : connexions acceptées/refusées.  
- `[KafkaApis]` : requêtes API (produce, fetch, metadata).

📌 **Exemple de log de basculement** :
```
INFO [Controller 1]: Starting preferred replica leader election for partitions [topic-0, topic-1]
INFO [Controller 1]: New leader for partition topic-0 is 2 (old leader 1)
```

💡 **Astuce pro** :  
- Augmentez le niveau de log pour `kafka.controller` et `kafka.server.ReplicaManager` lors d’incidents.  
- Utilisez `tail -f` sur `server.log` en production avec des filtres (grep).

#### 5.2. Logs client (producteur / consommateur)

Les clients Kafka (applications) produisent aussi des logs précieux. Activez le niveau `DEBUG` pour `org.apache.kafka.clients`.

📌 **Exemple de log de rebalance** :
```
DEBUG [Consumer clientId=app-1, groupId=mon-groupe] Assignor leader: group assignment ...
```

💡 **Astuce pro** :  
- Les logs clients sont souvent désactivés ; activez‑les temporairement pour résoudre des problèmes de rebalance ou de transaction.

---

### 6. Tests – TopologyTestDriver, integration tests, harness

#### 6.1. TopologyTestDriver (Kafka Streams)

Permet de tester la topologie sans cluster réel, en mémoire.

```java
import org.apache.kafka.streams.TopologyTestDriver;
import org.apache.kafka.streams.TestInputTopic;
import org.apache.kafka.streams.TestOutputTopic;
import org.apache.kafka.streams.StreamsBuilder;
import org.junit.jupiter.api.Test;

public class MyTopologyTest {

    @Test
    void testTopology() {
        StreamsBuilder builder = new StreamsBuilder();
        // ... construire votre topologie
        Topology topology = builder.build();
        Properties config = new Properties();
        config.put(StreamsConfig.APPLICATION_ID_CONFIG, "test");
        config.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "dummy");
        TopologyTestDriver driver = new TopologyTestDriver(topology, config);

        TestInputTopic<String, String> input = driver.createInputTopic("input-topic", Serdes.String().serializer(), Serdes.String().serializer());
        TestOutputTopic<String, String> output = driver.createOutputTopic("output-topic", Serdes.String().deserializer(), Serdes.String().deserializer());

        input.pipeInput("key", "value");
        assertThat(output.readValue()).isEqualTo("expected");
        driver.close();
    }
}
```

💡 **Astuce pro** :  
- Le `TopologyTestDriver` simule le temps via `advanceWallClockTime()` et `advanceStreamTime()`.  
- Il ne supporte pas les interactions externes (appels à des bases de données) sans moquerie.

#### 6.2. Embedded Kafka – cluster local pour tests d’intégration

Utiliser `kafka-clients` et le **Embedded Kafka** (bibliothèque `kafka-junit` ou `spring-kafka-test`).

```java
@ClassRule
public static EmbeddedKafkaBroker embeddedKafka = new EmbeddedKafkaBroker(1, true, "input-topic");

@Test
void testProduceAndConsume() {
    Map<String, Object> configs = new HashMap<>();
    configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, embeddedKafka.getBrokersAsString());
    // ... envoyer un message, consommer, vérifier
}
```

💡 **Astuce pro** :  
- Les clusters embarqués sont lents, donc préférez `TopologyTestDriver` pour les tests unitaires.  
- Pour les tests end‑to‑end, montez un vrai broker via Docker (Testcontainers).

#### 6.3. Testcontainers Kafka (moderne)

```java
@Testcontainers
public class KafkaIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:latest"));

    @Test
    void testWithRealBroker() {
        String bootstrapServers = kafka.getBootstrapServers();
        // ... produire, consommer
    }
}
```

💡 **Astuce pro** :  
- Testcontainers démarre un vrai broker, idéal pour tester exactement les configurations.  
- Pensez à limiter les ressources (mémoire, CPU) pour éviter les lenteurs en CI.

---

### 7. Tableau de bord (Grafana + Prometheus) – configuration minimale

1. **Exporter JMX** (`jmx_exporter`) lancé sur chaque broker.
2. **Prometheus** : scrap les métriques.
3. **Grafana** : visualisation avec dashboard (ex: `https://grafana.com/grafana/dashboards/10459`).

Exemple de job `prometheus.yml` :  
```yaml
scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets: ['localhost:9999']
    metrics_path: '/metrics'
```

💡 **Astuce pro** :  
- Les dashboards recommandés : `Kafka Exporter Dashboard` (ID 14751) et `Kafka Consumer Lag` (ID 13853).  
- Associez des alertes Prometheus (ex: `UnderReplicatedPartitions > 0`).

---

### 8. Tableau récapitulatif – commandes de debug rapide

| Problème suspecté | Commande CLUE |
|------------------|---------------|
| Consommateur lent | `kafka-consumer-groups --describe --group <g>` → voir `LAG` |
| Partition sous‑répliquée | `kafka-topics --describe --under-replicated-partitions` |
| Déséquilibre des leaders | `kafka-leader-election --election-type preferred --all-topic-partitions` |
| Vérifier l’état des logs | `kafka-dump-log --files <log>` |
| Réassignation | `kafka-reassign-partitions --generate` puis `--execute` |
| Config modifiée récemment | `kafka-configs --describe --entity-type topics --entity-name <t>` |
| Brokers en vie | `kafka-broker-api-versions --bootstrap-server localhost:9092` |

---

## Conclusion du chapitre 5 – points clés

- **JMX** : activez‑le sur brokers et clients, centralisez dans Prometheus.
- **CLI** : maîtrisez `kafka-topics`, `kafka-consumer-groups`, `kafka-reassign-partitions`.
- **Logs** : surveillez les erreurs et les changements de leadership.
- **Tests** : `TopologyTestDriver` pour les streams, `Testcontainers` pour l’intégration.
- **Alerting** : lag, sous‑réplication, taux d’erreur.

Avec ces cinq chapitres, vous disposez d’une couverture exhaustive de l’écosystème Kafka pour réussir la certification Confluent Developer. 🚀
