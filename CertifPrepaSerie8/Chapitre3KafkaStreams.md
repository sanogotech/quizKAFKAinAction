# 🔁 Chapitre 3 : Kafka Streams – KStream, KTable, State Stores, Windowing, et Intégration  
*(Version ultra-détaillée – 5 fois plus de contenu que la précédente)*

---

### 1. Principes fondamentaux de Kafka Streams

Kafka Streams est une bibliothèque Java (pas de serveur séparé) pour construire des applications de traitement de flux **élastiques**, **fault‑tolerant**, et **exactly‑once**, directement sur Kafka.

#### 1.1. Topologie de traitement – un graphe de processeurs

Une application Kafka Streams définit une **topologie** (DAG) de nœuds de traitement :
- **Source** : un ou plusieurs topics d’entrée (KStream ou KTable).
- **Processeurs** : opérations stateless (map, filter, flatMap) ou stateful (aggregate, join, window).
- **Sink** : écriture dans un ou plusieurs topics de sortie.

📌 **Exemple de topologie simple** :  
```java
StreamsBuilder builder = new StreamsBuilder();
KStream<String, String> source = builder.stream("input-topic");
KStream<String, String> processed = source.mapValues(v -> v.toUpperCase());
processed.to("output-topic");
```

💡 **Astuce pro** :  
- Utilisez `TopologyTestDriver` pour tester la topologie hors ligne.  
- Visualisez la topologie avec `Topology.describe()`.

#### 1.2. Threading et parallélisme

- Chaque instance de l’application crée un nombre configurable de threads (`num.stream.threads`).
- Chaque thread exécute un sous‑ensemble de **tasks** (une tâche par partition des topics d’entrée).
- Le nombre maximal de tâches est le nombre de partitions du topic d’entrée (ou le max si plusieurs topics).

📌 **Exemple** : topic d’entrée avec 6 partitions, `num.stream.threads=3` → chaque thread traite 2 partitions (2 tâches).

💡 **Astuce pro** :  
- `num.stream.threads` ne doit généralement pas dépasser le nombre de cœurs CPU disponibles.  
- Surveillez `kafka.streams:type=stream-thread-metrics` pour le taux d’occupation des threads.

---

### 2. KStream vs KTable – Modélisation des données

#### 2.1. KStream – Flux d’événements

Un **KStream** représente un flux immuable d’enregistrements. Chaque enregistrement est indépendant ; il n’y a pas de notion de mise à jour d’une clé.

📌 **Exemple** : clics utilisateur, logs, relevés capteurs.  
Opérations typiques : `map`, `filter`, `flatMap`, `branch`, `merge`.

💡 **Astuce pro** :  
- Les opérations sur KStream sont **stateless** par défaut, sauf si vous utilisez des états (agrégations fenêtrées).  
- L’ordre entre les partitions n’est pas garanti, mais l’ordre au sein d’une partition l’est.

#### 2.2. KTable – Vue mutable (table)

Une **KTable** représente une vue actualisable des dernières valeurs par clé. Elle est construite à partir d’un topic **changelog** (compacted) ou dérivée d’un KStream.

📌 **Exemple** : dernière position GPS d’un véhicule, solde de compte bancaire.  
Opérations : `mapValues`, `filter`, `join` avec KStream, `toStream`.

💡 **Astuce pro** :  
- Une KTable est **matérialisée** dans un state store local (RocksDB).  
- Les mises à jour successives de la même clé remplacent la valeur précédente.

#### 2.3. GlobalKTable – Table globalement répliquée

Une **GlobalKTable** est une KTable dont l’intégralité des données est chargée sur **chaque instance** de l’application. Elle permet des jointures sans repartitionnement.

📌 **Exemple** : table de référence (codes postaux, taux de change) de petite taille (quelques milliers d’enregistrements).

💡 **Astuce pro** :  
- Évitez les GlobalKTables volumineuses (plus de quelques centaines de Mo) car elles augmentent la mémoire de chaque instance.  
- Le topic sous‑jacent doit être compacté (`cleanup.policy=compact`).

---

### 3. State Stores – Moteur de gestion d’état

#### 3.1. Types de state stores

| Type | Persistance | Cas d’usage |
|------|-------------|--------------|
| `KeyValueStore` | RocksDB (par défaut) | Aggrégations simples, réductions, joints de tables |
| `WindowStore` | RocksDB avec index temporel | Fenêtres tumbling, hopping |
| `SessionStore` | RocksDB pour sessions | Fenêtres par session |
| `InMemoryKeyValueStore` | Mémoire volatile (tests) | Tests unitaires, petites données |

📌 **Configuration** :  
```java
StoreBuilder<KeyValueStore<String, Long>> storeBuilder =
    Stores.keyValueStoreBuilder(Stores.persistentKeyValueStore("my-store"),
                                Serdes.String(), Serdes.Long());
builder.addStateStore(storeBuilder);
```

💡 **Astuce pro** :  
- Pour les très gros états (plus de mémoire RAM), utilisez RocksDB avec un cache bien dimensionné (`rocksdb.config.setter`).  
- Utilisez `InMemoryKeyValueStore` uniquement pour les tests ou les très petits états volatiles.

#### 3.2. Changelog topics – tolérance aux pannes

Chaque state store a un **topic de changelog** interne, partitionné de la même manière que le stream/topic source. En cas de panne, l’instance redémarre et **rejoue** le changelog pour restaurer l’état.

📌 **Paramètres influents** :  
- `commit.interval.ms` : définit la fréquence de prise de snapshot dans le changelog. Plus faible, moins de perte possible, mais plus de charge réseau.  
- `num.standby.replicas` : nombre de répliques en attente qui consomment le changelog pour réduire le temps de récupération.

💡 **Astuce pro** :  
- Ne désactivez jamais le changelog (impossible).  
- Prenez garde à la rétention du changlog (`log.retention.ms`) : si trop courte, la restauration peut échouer.

#### 3.3. Restoration et standby replicas

- **Restauration normale** : une tâche lit le changelog depuis le début pour reconstruire l’état local.  
- **Standby replicas** : d’autres instances du même groupe maintiennent une copie de l’état en mémoire ; en cas de panne, la bascule est quasi immédiate.

📌 **Configuration** :  
```java
props.put(StreamsConfig.NUM_STANDBY_REPLICAS_CONFIG, 1);
```

💡 **Astuce pro** :  
- Les standby replicas consomment de la mémoire et du CPU, mais améliorent la résilience.  
- Pour les applications critiques, utilisez `num.standby.replicas = 1` au minimum.

---

### 4. Windowing – Découpage temporel des agrégations

#### 4.1. Tumbling Window (fenêtre fixe sans chevauchement)

Les fenêtres sont contiguës, de taille fixe, alignées sur l’horloge (par ex. de 0h00 à 0h05). Chaque événement appartient à exactement une fenêtre.

📌 **Exemple** : ventes par heure.  
```java
KTable<Windowed<String>, Long> count = stream
    .groupByKey()
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .count();
```

💡 **Astuce pro** :  
- L’alignement est sur l’époque Unix (début 1970). Pour un décalage, utilisez `TimeWindows.of(...).advanceBy(...)`.

#### 4.2. Hopping Window (fenêtres chevauchantes)

Fenêtres de taille fixe, qui avancent d’un pas plus petit que la taille. Un même événement peut appartenir à plusieurs fenêtres.

📌 **Exemple** : moyenne glissante des 15 dernières minutes, mise à jour toutes les 5 minutes.  
```java
.windowedBy(TimeWindows.of(Duration.ofMinutes(15)).advanceBy(Duration.ofMinutes(5)))
```

💡 **Astuce pro** :  
- Le paramètre `advanceBy` doit être un sous‑multiple de la taille pour éviter des trous.  
- Utile pour les dashboards temps réel.

#### 4.3. Session Window (fenêtres par inactivité)

Les fenêtres se ferment après une période d’inactivité (gap). Chaque événement étend la session.

📌 **Exemple** : session utilisateur (ensemble des clics tant qu’il n’y a pas de trou > 30 minutes).  
```java
.windowedBy(SessionWindows.with(Duration.ofMinutes(30)))
```

💡 **Astuce pro** :  
- Les fenêtres de session nécessitent un state store spécifique (`SessionStore`).  
- La sortie est émise sous forme de `SessionWindowed` (avec début et fin).

#### 4.4. Grace period et gestion du retard

Les événements qui arrivent après la fin de la fenêtre peuvent être intégrés si le `grace period` le permet. Passé ce délai, ils sont ignorés.

```java
.windowedBy(TimeWindows.of(Duration.ofMinutes(5)).grace(Duration.ofSeconds(30)))
```

💡 **Astuce pro** :  
- Un grace period trop court exclut les événements en retard (par ex. réseaux lents).  
- Un grace period trop long retarde la publication des résultats et augmente l’état.  
- Règle empirique : grace period = taille de la fenêtre × 0,2 à 0,5.

---

### 5. Jointures entre streams et tables

#### 5.1. Stream‑Stream join (KStream-KStream)

Jointure basée sur le temps : les deux flux sont fenêtrés. Seuls les enregistrements dont les timestamps sont proches (dans une fenêtre) sont joints.

📌 **Exemple** : joindre des clics avec des impressions dans une fenêtre de 10 secondes.  
```java
stream1.join(stream2,
    (value1, value2) -> value1 + "_" + value2,
    JoinWindows.of(Duration.ofSeconds(10))
);
```

💡 **Astuce pro** :  
- Les deux streams doivent être co‑partitionnés (même nombre de partitions et même clé).  
- Utilisez `.through()` si nécessaire pour repartitionner.

#### 5.2. Stream‑Table join (KStream-KTable)

Jointure d’un flux avec une table : à chaque événement du flux, on cherche la valeur courante dans la table (au moment du traitement). Très utile pour l’enrichissement.

📌 **Exemple** : flux de commandes avec table des clients.  
```java
KStream<String, Order> orderStream = ...;
KTable<String, Customer> customerTable = ...;
KStream<String, EnrichedOrder> enriched = orderStream.join(customerTable,
    (order, customer) -> new EnrichedOrder(order, customer));
```

💡 **Astuce pro** :  
- La table doit être une KTable (ou GlobalKTable).  
- La jointure ne nécessite pas de fenêtre ; elle utilise la dernière valeur de la table.

#### 5.3. Table‑Table join (KTable-KTable)

Jointure deux tables pour synchroniser des changements. Le résultat est une nouvelle table.

📌 **Exemple** : joindre des stocks et des prix.  
```java
KTable<String, Stock> stockTable = ...;
KTable<String, Price> priceTable = ...;
KTable<String, StockWithPrice> joined = stockTable.join(priceTable,
    (stock, price) -> new StockWithPrice(stock, price));
```

💡 **Astuce pro** :  
- Les enregistrements de part et d’autre sont mis à jour indépendamment ; le résultat est recalculé à chaque changement.

---

### 6. Configuration avancée et exactly‑once

#### 6.1. Paramètres clés de Kafka Streams

| Paramètre | Défaut | Rôle |
|-----------|--------|------|
| `application.id` | (obligatoire) | Identifiant unique de l’application, utilisé pour les groupes de consommateurs et le nom des topics internes |
| `bootstrap.servers` | – | Liste des brokers |
| `processing.guarantee` | `at_least_once` | `exactly_once_v2` pour EOS |
| `commit.interval.ms` | 30000 (30s) | Fréquence de commit des offsets et des changelog |
| `num.stream.threads` | 1 | Nombre de threads de traitement |
| `cache.max.bytes.buffering` | 10 MB | Taille du cache pour les opérations stateful (réduit les émissions) |
| `state.dir` | `/tmp/kafka-streams` | Répertoire local pour les state stores |

💡 **Astuce pro** :  
- `application.id` doit être unique par application.  
- Pour les environnements de test, évitez d’utiliser le même `application.id` sur plusieurs instances.

#### 6.2. Debugging et monitoring d’une application Streams

| Commande / Métrique | Utilité |
|---------------------|---------|
| `kafka-streams-application-reset` | Réinitialiser les offsets et effacer les state stores (rejeu) |
| `kafka-topics --list | grep -E "-changelog|-repartition"` | Lister les topics internes |
| `kafka-consumer-groups --describe --group <application-id>` | Voir les retards des topics sources |
| `kafka.streams:type=stream-thread-metrics` `process-latency-avg` | Latence moyenne de traitement |

💡 **Astuce pro** :  
- L’outil `kafka-streams-application-reset` est très dangereux (efface tout). Sauvegardez d’abord.  
- Les topics internes sont nommés `<application-id>-<suffixe>-repartition` et `<application-id>-<store-name>-changelog`.

---

### 7. Intégration avec Kafka Connect – flux entrant / sortant

Bien que Kafka Streams ne dépende pas de Connect, on les utilise souvent ensemble :
- **Source** : un connecteur ingère des données (CDC) dans un topic.
- **Streams** : transforme/agrège ces données.
- **Sink** : un autre connecteur exporte le résultat.

📌 **Exemple** :  
CDC Debezium → topic `mysql.customers` → Streams (enrichissement, nettoyage) → topic `customers_enriched` → Elasticsearch sink.

💡 **Astuce pro** :  
- Utilisez des SMT dans Connect pour faire des transformations simples avant Streams.  
- Évitez de faire des traitements lourds dans Connect ; préférez Streams.

---

### 8. Exemple complet de topologie

```java
// Topologie d’analyse de ventes : total par catégorie, fenêtre glissante 1 heure
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Sale> sales = builder.stream("sales", Consumed.with(Serdes.String(), saleSerde));

KTable<String, Double> totalByCategory = sales
    .groupBy((key, sale) -> sale.getCategory(), Grouped.with(Serdes.String(), Serdes.Double()))
    .windowedBy(TimeWindows.of(Duration.ofHours(1)).advanceBy(Duration.ofMinutes(1)))
    .aggregate(() -> 0.0,
        (aggKey, newValue, aggValue) -> aggValue + newValue.getAmount(),
        Materialized.<String, Double, WindowStore<Bytes, byte[]>>as("sales-window-store")
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Double()))
    .toStream()
    .selectKey((windowedKey, value) -> windowedKey.key());

totalByCategory.to("sales-total-by-category", Produced.with(Serdes.String(), Serdes.Double()));

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
```

💡 **Astuce pro** :  
- Utilisez `Materialized.as` pour rendre le store accessible en **interactive queries**.  
- Les objets `Windowed<String>` peuvent être sérialisés via `WindowedSerdes.timeWindowedSerdeFrom(...)`.

---

### 9. Résumé – commandes et métriques essentielles pour Kafka Streams

| Besoin | Commande / Action |
|--------|-------------------|
| Voir les topics internes | `kafka-topics --bootstrap-server ... --list | grep "application-id"` |
| Réinitialiser l’application | `kafka-streams-application-reset --application-id my-app --bootstrap-servers ...` |
| Surveiller le lag | `kafka-consumer-groups --describe --group my-app` |
| Vérifier l’état des stores | JMX : `kafka.streams:type=stream-state-metrics` |
| Visualiser la topologie | Ajouter `Topology.describe()` dans le code, ou utiliser `TopologyTestDriver` |

---

## Conclusion du chapitre 3 – points clés

- **KStream** = flux d’événements. **KTable** = dernière valeur par clé.
- **State stores** : persistants (RocksDB), fault‑tolerant via changelog topics.
- **Windowing** : tumbling, hopping, session avec grace period.
- **Exactly‑once** : `processing.guarantee=exactly_once_v2`.
- **Monitoring** : lag, latence, état des stores.

**Prochain chapitre** : Kafka Connect (sources, sinks, SMT, CDC) – avec le même niveau de détail.
