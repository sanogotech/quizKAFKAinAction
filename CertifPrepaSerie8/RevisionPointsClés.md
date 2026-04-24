# 🔥 Architecture Kafka, Producteurs/Consommateurs, Streams, Connect, Monitoring – Version ultra-détaillée (3x plus)

## 📘 Chapitre 1 : Architecture Kafka (partitions, réplication, ISR, KRaft)

### 1.1 Partitions – le cœur de la scalabilité

Une partition est un **fichier journal (log) append-only** sur le disque d’un broker. Chaque message écrit dans une partition reçoit un **offset** (numéro séquentiel unique dans la partition).

- **Pourquoi découper un topic en partitions ?**  
  → Parallélisme : plusieurs producteurs écrivent sur des partitions différentes, plusieurs consommateurs lisent des partitions différentes.  
  → Ordre garanti **à l’intérieur d’une partition** seulement.

📌 **Exemple concret** :  
Topic `commandes` avec 6 partitions.  
Clé de message = `id_client`.  
Tous les messages d’un même client iront toujours dans la même partition (via hachage de la clé). Ainsi, l’ordre des commandes pour ce client est préservé.

💡 **Astuce pro** :  
- Le nombre de partitions **limite la concurrence de consommation** : 1 partition ne peut être lue que par 1 consommateur d’un groupe.  
- Augmenter les partitions augmente la parallélisation, mais trop de partitions (milliers par broker) dégrade les performances (ouverture de fichiers, métadonnées).  
- Règle empirique : 1 partition = débit max ~10-20 MB/s. Calculez : débit souhaité / débit par partition = nombre de partitions.

### 1.2 Réplication – la haute disponibilité

Chaque partition a **plusieurs copies (réplicas)** sur différents brokers.  
- **Leader** : reçoit toutes les écritures et lectures (par défaut).  
- **Followers** : copient passivement les données du leader.

📌 **Exemple** :  
Facteur de réplication = 3.  
Partition 0 : leader sur broker 101, followers sur brokers 102 et 103.  
Si broker 101 tombe, le broker 102 (ou 103) devient automatiquement le nouveau leader.

💡 **Astuce pro** :  
- `replication.factor` doit être **≥ 3** en production.  
- `min.insync.replicas` = nombre minimum de réplicas qui doivent accuser réception pour qu’une écriture soit validée. Avec RF=3, `min.insync.replicas=2` → tolère la perte d’un broker sans perte de données.  
- **Attention** : si le nombre d’ISR tombe sous `min.insync.replicas`, le producteur reçoit `NotEnoughReplicasException`.

### 1.3 ISR (In-Sync Replicas) – les réplicas à jour

L’ISR est l’ensemble des réplicas qui n’ont **pas de retard** par rapport au leader (leur offset est identique ou dans une limite configurable).  
Un follower est retiré de l’ISR s’il ne parvient pas à rattraper le leader dans le temps `replica.lag.time.max.ms` (défaut : 30 secondes).

📌 **Exemple concret** :  
Broker 102 a un disque lent. Pendant une minute, il ne peut pas copier les nouveaux messages. Le leader le retire de l’ISR. Une fois le disque rétabli, il rattrape son retard et réintègre l’ISR.

💡 **Astuce pro** :  
- **N’autorisez jamais l’élection d’un leader hors ISR** sauf si vous acceptez une perte de données (`unclean.leader.election.enable=false`).  
- Surveillez la métrique `kafka.controller:type=ControllerStats,name=ShrinkingISRRate`. Une valeur élevée indique des followers qui décrochent régulièrement.

### 1.4 KRaft – Kafka sans ZooKeeper

Jusqu’à Kafka 2.x, ZooKeeper était nécessaire pour gérer les métadonnées (élection du contrôleur, liste des topics, etc.). Avec **KRaft** (Kafka Raft), ZooKeeper est remplacé par un **quorum de contrôleurs** intégrés.

📌 **Exemple** :  
Vous voulez créer un topic. Au lieu d’écrire dans ZooKeeper, la requête va au **contrôleur actif** (élu par Raft). Tous les contrôleurs synchronisent les métadonnées via le protocole Raft.

💡 **Astuce pro** :  
- Un cluster KRaft doit avoir un nombre **impair de contrôleurs** (3 ou 5).  
- Les contrôleurs peuvent être des brokers dédiés (`process.roles=controller`) ou des brokers mixtes (déconseillé pour la production).  
- La migration de ZooKeeper vers KRaft est possible sans downtime via `kafka-features` et des mises à niveau roulantes.  
- Outil essentiel : `kafka-dump-log --cluster-metadata` pour inspecter les métadonnées KRaft.

---

## ⚙️ Chapitre 2 : Producteurs et consommateurs (idempotence, exactly‑once, rebalance)

### 2.1 Idempotence – pas de doublons

Un producteur idempotent attribue un **numéro de séquence** à chaque message par partition. Le broker vérifie que le numéro reçu est exactement celui attendu (séquence + 1). Si identique, il ignore le doublon.

📌 **Exemple** :  
Le producteur envoie un message (séquence 5), perd l’ACK réseau, et réessaie. Le broker reçoit à nouveau séquence 5. Il le rejette silencieusement car il a déjà enregistré le message 5. Le producteur sait que son message a été écrit (via le callback de succès).

💡 **Astuce pro** :  
- Activez l’idempotence avec `enable.idempotence=true`.  
- Elle impose automatiquement `acks=all` et `max.in.flight.requests.per.connection ≤ 5`.  
- Nécessaire mais pas suffisant pour exactly‑once (il manque les transactions)

### 2.2 Exactly‑once semantics (EOS)

Garantie qu’un message sera écrit **exactement une fois** (ni perte, ni doublon) et que plusieurs écritures atomiques (productions + offsets) peuvent être regroupées en une transaction.

📌 **Exemple** :  
Une application lit un topic `paiements`, les traite, puis écrit dans `validations` et `rejets` (deux topics). En mode EOS : si l’écriture dans `rejets` échoue, les écritures dans `validations` et la mise à jour des offsets sont **annulées** – le message sera rejoué.

💡 **Astuce pro** :  
- Pour un producteur simple : `enable.idempotence=true` + `transactional.id` unique.  
- Pour Kafka Streams : `processing.guarantee=exactly_once_v2` (meilleures performances que v1).  
- Le consommateur doit être en mode `isolation.level=read_committed` pour ne pas voir les messages non commités.  
- Attention : les transactions dégradent légèrement le débit (environ 10-15%).

### 2.3 Rebalance – gestion dynamique des consommateurs

Quand un consommateur rejoint ou quitte un groupe, ou que des partitions sont ajoutées, les partitions sont **redistribuées** entre les membres.

📌 **Exemple** :  
Groupe `monGroupe`, topic avec 4 partitions.  
Consommateur A rejoint → reçoit les partitions 0,1.  
Consommateur B rejoint → reçoit les partitions 2,3.  
A quitte le groupe → les partitions 0,1 sont réaffectées à B (B reçoit alors 0,1,2,3).

💡 **Astuce pro** :  
- **Évitez les rebalances longues** : réduisez `session.timeout.ms` (défaut 45s) et `heartbeat.interval.ms` (défaut 3s).  
- Utilisez **CooperativeStickyAssignor** (`partition.assignment.strategy`) : les rebalances ne révoquent que les partitions nécessaires, toutes ne sont pas arrêtées.  
- Pour les applications stateful, gérez `onPartitionsRevoked` (commit) et `onPartitionsAssigned` (restauration).  
- Surveillez la métrique `kafka.consumer:type=consumer-coordinator-metrics,name=rebalance-latency-avg`.

---

## 🔁 Chapitre 3 : Kafka Streams (KStream, KTable, state stores, windowing)

### 3.1 KStream vs KTable – la dualité flux/table

- **KStream** : flux d’événements immutables. Chaque message est un événement indépendant.  
- **KTable** : vue mutable sur un flux de **mises à jour**. Pour une clé donnée, seule la dernière valeur est conservée.

📌 **Exemple concret** :  
Topic `positions-vehicules`.  
- KStream : toutes les positions GPS (même clé plusieurs fois).  
- KTable : dernière position connue de chaque véhicule (équivalent à une table de base de données).

💡 **Astuce pro** :  
- Une KTable peut être matérialisée en **state store queryable** (interactive queries).  
- `GlobalKTable` : chaque instance de l’application charge **toute** la table – idéal pour les petites tables de référence (pays, codes postaux).  
- Attention : une KTable **repartionnée** engendre un topic interne.

### 3.2 State stores – le stockage local stateful

Les opérations stateful (aggregations, joins, windowing) nécessitent un stockage local. Par défaut, il s’agit de **RocksDB** (embarqué).

📌 **Exemple** :  
Vous comptez le nombre de clics par utilisateur. L’application maintient dans RocksDB un map `utilisateur -> count`. À chaque nouveau clic, on lit, on incrémente, on réécrit. Un topic changelog (`__click-counts-changelog`) enregistre chaque modification pour permettre la reprise après panne.

💡 **Astuce pro** :  
- Taille du state store : surveillez `rocksdb.bytes-written` et ajustez la mémoire via `rocksdb.config.setter`.  
- Pour des états énormes (> mémoire), RocksDB utilise le disque – utilisez du SSD.  
- Les **standby replicas** (`num.standby.replicas=1`) accélèrent la restauration.

### 3.3 Windowing – découpage temporel

- **Tumbling window** : intervalles fixes, sans chevauchement (ex: de 10h00 à 10h05, 10h05 à 10h10).  
- **Hopping window** : intervalles qui se chevauchent (ex: fenêtre de 10 min, avance de 5 min).  
- **Session window** : fenêtres délimitées par une période d’inactivité (ex: 30 min sans événement clôture la session).

📌 **Exemple** :  
Analyse de ventes.  
- Tumbling : ventes par heure de la journée (0h-1h, 1h-2h...).  
- Hopping : moyenne glissante des 15 dernières minutes.  
- Session : panier d’achat par utilisateur – une session se termine quand l’utilisateur n’interagit plus pendant 30 minutes.

💡 **Astuce pro** :  
- Le paramètre **grace period** (`window.grace`) tolère les événements en retard.  
- Utilisez `Suppressed` pour réduire le nombre d’émissions de résultats (ex: n’émettre qu’à la fin de la fenêtre).  
- Pour des fenêtres très larges (jours, mois), utilisez une logique d’heures de l’événement plutôt que de l’horloge système.

---

## 🔌 Chapitre 4 : Kafka Connect (sources, sinks, SMT, CDC)

### 4.1 Source connectors – injecter des données dans Kafka

Un source connector lit des données depuis un système externe (BDD, fichier, API, IoT) et les écrit dans un topic Kafka.

📌 **Exemple** :  
Connecteur JDBC source interroge toutes les 5 secondes une table `commandes` et publie chaque ligne comme un message Kafka dans le topic `mysql.commandes`.

💡 **Astuce pro** :  
- Utilisez `tasks.max` pour paralléliser l’ingestion (chaque tâche lit un sous‑ensemble des partitions de la source).  
- Le mode **incremental** (requête avec colonne de timestamp) évite de recharger toutes les données à chaque cycle.  
- Pour les bases de données, préférez **Debezium** (CDC) plutôt que le polling JDBC – plus faible latence.

### 4.2 Sink connectors – envoyer les données hors de Kafka

Un sink connector lit depuis un topic et écrit dans un système externe (Elasticsearch, S3, Snowflake, base de données).

📌 **Exemple** :  
Connecteur Elasticsearch sink lit le topic `logs` et indexe chaque document dans Elasticsearch avec un ID dérivé de la clé Kafka.

💡 **Astuce pro** :  
- Activez le **dead letter queue** (`errors.deadletterqueue.topic.name`) pour ne pas bloquer le pipeline sur des erreurs de format.  
- Définissez `insert.mode=upsert` pour les bases de données relationnelles (mettre à jour ou insérer).  
- Surveillez le lag du sink : s’il augmente, le système cible est trop lent → augmentez `tasks.max` ou augmentez la taille des batches.

### 4.3 SMT (Single Message Transform) – transformations légères

Les SMT modifient chaque message **individuellement** à la volée, avant (source) ou après (sink) la conversion.

📌 **Exemple** :  
- `MaskField` : masquer un champ `email` ou `motdepasse`.  
- `InsertField` : ajouter un champ `timestamp_ms`.  
- `Filter` : ne garder que les messages avec `prio > 5`.  
- `ReplaceField` : renommer `customer_id` → `clientId`.

💡 **Astuce pro** :  
- Les SMT s’enchaînent dans l’ordre de la liste `transforms`.  
- Évitez les SMT lourdes (appels externes, calculs complexes) – ce ne sont pas des stream processors.  
- Pour des transformations avancées, utilisez **Kafka Streams** ou **ksqlDB**.

### 4.4 CDC (Change Data Capture) – capturez chaque modification de BDD

CDC transforme chaque INSERT/UPDATE/DELETE d’une base de données en un événement Kafka. Debezium est le standard.

📌 **Exemple** :  
Avec Debezium MySQL connector, un UPDATE sur la table `clients` produit un message Kafka dans le topic `dbserver1.inventory.clients` avec :  
- `op` = `"u"`  
- `before` = anciennes valeurs  
- `after` = nouvelles valeurs  
- `ts_ms` = timestamp

💡 **Astuce pro** :  
- Le connecteur CDC nécessite des droits de lecture sur le binlog (pour MySQL) ou la replication slot (PostgreSQL).  
- Les messages CDC peuvent être très volumineux (avant/après) – compressez (`compression.type`).  
- Pour des raisons de sécurité, masquez les champs sensibles avec SMT `MaskField` avant de publier.

---

## 📊 Chapitre 5 : Monitoring, CLI, JMX – outils du quotidien

### 5.1 Consumer lag – le pouls de votre pipeline

Le **lag** est la différence entre le dernier offset du topic et l’offset du dernier message consommé par le consommateur.  
- Un lag qui augmente = consommateur trop lent.  
- Un lag stable = débit consommateur = débit producteur.

📌 **Exemple** :  
Commande : `kafka-consumer-groups --bootstrap-server localhost:9092 --group monGroupe --describe`  
Résultat :  
```
TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
commandes       0          42              100             58
commandes       1          78              100             22
```

💡 **Astuce pro** :  
- Lag moyen idéal : proche de zéro pour les applications temps réel.  
- Si le lag persiste, augmentez le nombre de consommateurs ou ajoutez des partitions.  
- Utilisez `kafka-consumer-groups --reset-offsets` pour rejouer depuis le début (`--to-earliest`).

### 5.2 CLI (command line tools) – la boîte à outils indispensable

| Commande | Utilité |
|----------|---------|
| `kafka-topics --bootstrap-server ... --list` | Lister les topics |
| `kafka-topics --describe --topic nom` | Voir les partitions, réplicas, ISR |
| `kafka-console-producer --topic nom --property parse.key=true` | Produire depuis stdin |
| `kafka-console-consumer --topic nom --from-beginning` | Consommer et afficher |
| `kafka-consumer-groups --describe --group g` | Lag par partition |
| `kafka-configs --alter --add-config retention.ms=86400000 --topic nom` | Changer rétention sans redémarrage |
| `kafka-reassign-partitions --generate` | Rééquilibrer les partitions |
| `kafka-dump-log --files /var/lib/kafka/data/topic-0/00000000000000000000.log` | Inspecter un segment de log |

💡 **Astuce pro** :  
- En KRaft, tous les outils utilisent **`--bootstrap-server`** au lieu de `--zookeeper`.  
- Pour les gros clusters, utilisez `--command-config` pour passer des fichiers de configuration (SSL, SASL).  
- La commande `kafka-run-class kafka.tools.DumpLogSegments --deep-iteration` montre chaque message en clair.

### 5.3 JMX – les métriques cachées

Kafka expose des milliers de métriques via JMX (Java Management Extensions). On les récupère avec un agent JMX (ex: Prometheus JMX exporter) ou via `jconsole`.

📌 **Quelques métriques critiques** :  

| MBean | Description | Seuil d’alerte |
|-------|-------------|----------------|
| `kafka.controller:type=KafkaController,name=ActiveControllerCount` | 1 = un seul contrôleur actif | ≠ 1 |
| `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions` | Partitions sous‑répliquées | > 0 |
| `kafka.network:type=RequestMetrics,name=RequestsPerSec` | Taux de requêtes par seconde | Variation anormale |
| `kafka.consumer:type=consumer-fetch-manager-metrics,name=records-lag-max` | Lag max du consommateur | > 10 000 |
| `kafka.producer:type=producer-metrics,client-id=...` | Latence, erreurs, compression | Latence > 500ms |

💡 **Astuce pro** :  
- Exposez JMX sur un port spécifique (`JMX_PORT=9999`) et connectez un exporter Prometheus.  
- Utilisez **Grafana** avec dashboards préconstruits (ex: `Kafka Exporter`).  
- Les métriques `kafka.log:type=LogFlushStats` permettent de détecter des problèmes de disque.

### 5.4 Tests avec TopologyTestDriver (Kafka Streams)

Plutôt que de démarrer un vrai cluster Kafka, `TopologyTestDriver` exécute votre topologie **en mémoire** pour des tests unitaires rapides.

📌 **Exemple** :  
```java
TopologyTestDriver driver = new TopologyTestDriver(topology, config);
TestInputTopic<String, String> input = driver.createInputTopic("input", stringSerde, stringSerde);
TestOutputTopic<String, String> output = driver.createOutputTopic("output", stringSerde, stringSerde);
input.pipeInput("key", "value");
assertThat(output.readValue()).isEqualTo("expected");
```

💡 **Astuce pro** :  
- Les tests unitaires avec `TopologyTestDriver` sont jusqu’à 100 fois plus rapides qu’avec un vrai Kafka.  
- Vous pouvez aussi tester la gestion du temps via `advanceWallClockTime()` ou `advanceStreamTime()`.  
- Pour les state stores, utilisez `driver.getKeyValueStore("storeName")` pour inspecter l’état.

---

## ✅ Récapitulatif des commandes et métriques (aide‑mémoire)

| Contexte | Commande ou métrique à retenir |
|----------|--------------------------------|
| Voir le lag | `kafka-consumer-groups --describe --group <g>` |
| Changer rétention | `kafka-configs --alter --add-config retention.ms=86400000` |
| Lister les topics | `kafka-topics --list` |
| Inspecter un segment | `kafka-dump-log --files <logfile>` |
| Lag max JMX | `records-lag-max` (consumer) |
| Partitions sous‑répliquées | `UnderReplicatedPartitions` (JMX) |
| État des contrôleurs KRaft | `kafka-features --list` |
| Simuler un consommateur | `kafka-console-consumer --topic ... --from-beginning` |
| Rééquilibrer les partitions | `kafka-reassign-partitions --generate --execute` |

Avec cette base ultra-détaillée, vous êtes armé pour comprendre, diagnostiquer et coder en confiance sur l’écosystème Kafka. Bonne maîtrise ! 🚀
