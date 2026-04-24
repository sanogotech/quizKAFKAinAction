# 🔌 Chapitre 4 : Kafka Connect – Sources, Sinks, SMT, CDC  
*(Version ultra-détaillée – 5 fois plus de contenu que la précédente)*

---

### 1. Qu’est-ce que Kafka Connect ?

Kafka Connect est un framework **intégré à Kafka** pour importer ou exporter des données de manière **évolutive**, **tolérante aux pannes**, et **sans code**. Il utilise des **connecteurs** paramétrés par configuration (pas de programmation dans le cas général).

- **Mode standalone** : processus unique, pour le développement ou les petits volumes.
- **Mode distribué** : cluster de workers, répartition des tâches et haute disponibilité.

📌 **Exemple d’utilisation** :  
Ingérer une base MySQL dans Kafka avec Debezium, puis exporter vers Elasticsearch avec un sink connector.

💡 **Astuce pro** :  
- Préférez le **mode distribué** en production (au moins 2 workers).  
- Tous les workers doivent avoir la même `group.id` pour former le cluster.

---

### 2. Architecture d’un connecteur

Un connecteur se compose de deux parties :

- **Connector** : définit la configuration globale (par ex. quels topics lire ou écrire). Il est instancié une fois.
- **Task** : exécute le vrai transfert de données. Plusieurs tâches peuvent être créées pour paralléliser (paramètre `tasks.max`).

📌 **Cycle de vie** :  
1. On soumet une configuration JSON via l’API REST.  
2. Le worker crée le **connector** et le distribue.  
3. Le connector génère les **tâches** (autant que demandées, selon les partitions).  
4. Les tâches tournent en boucle : `poll()` pour les sources, `put()` pour les sinks.

💡 **Astuce pro** :  
- Le nombre de tâches pour une source est souvent lié au nombre de partitions des topics cibles ou au découpage possible du système source.  
- Pour les sinks, `tasks.max` peut être augmenté jusqu’au nombre de partitions du topic consommé.

---

### 3. Source connectors – Injecter des données dans Kafka

Un **source connector** lit les données d’un système externe (base, fichier, API, IoT) et les publie dans un ou plusieurs topics Kafka.

#### 3.1. Fonctionnement interne d’une tâche source

- `poll()` est appelée périodiquement (toutes les `poll.interval.ms`) par le worker.
- La tâche retourne une liste de `SourceRecord` (contenant topic, partition (optionnelle), clé, valeur, timestamp, headers).
- Les `SourceRecord` sont écrits dans Kafka par le worker (via un producteur intégré).

📌 **Exemple simplifié** (`FileStreamSourceConnector`) :  
lit un fichier ligne par ligne et produit chaque ligne comme un message Kafka.

💡 **Astuce pro** :  
- Implémentez `start()` et `stop()` pour ouvrir/fermer les connexions.  
- Utilisez la **récupération d’offset** (`offset` dans `SourceRecord`) pour la reprise après crash.

#### 3.2. Connecteurs sources populaires

| Connecteur | Utilisation | Particularité |
|------------|-------------|----------------|
| `JDBCSourceConnector` | Interroge des tables SQL avec polling | Supporte le mode incremental (colonne timestamp) |
| `DebeziumSourceConnector` | CDC (Change Data Capture) depuis MySQL, Postgres, etc. | Mode binaire via logs, faible latence |
| `MongoDBSourceConnector` | Capture les changements dans l’oplog MongoDB | Nécessite une réplica set |
| `S3SourceConnector` | Lit des fichiers depuis S3 | Avro, JSON, Parquet |
| `FileStreamSource` | Fichiers texte (démonstration) | À éviter en production |

📌 **Exemple de configuration JDBC source** :  
```json
{
  "name": "jdbc-source",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:mysql://localhost:3306/orders",
    "mode": "incrementing",
    "incrementing.column.name": "id",
    "topic.prefix": "mysql-",
    "table.whitelist": "customers,orders"
  }
}
```

💡 **Astuce pro** :  
- En mode `incrementing` (basé sur une colonne qui croît), les nouvelles lignes sont détectées, mais les mises à jour ne le sont pas → utilisez `timestamp` ou mieux, **Debezium** pour le CDC.  
- Toujours définir `tasks.max` proportionnel au nombre de tables (ou de partitions logiques).

#### 3.3. Gestion des offsets côté source

Les connecteurs sources doivent **committer les offsets** pour pouvoir reprendre au bon endroit après un redémarrage. Le framework appelle `commit()` sur la tâche régulièrement (ou après un certain nombre de records).

📌 **Exemple de stockage d’offset** :  
Pour un JDBC source, l’offset est la dernière valeur de la colonne `incrementing.column`. Pour Debezium, l’offset est la position dans le binlog.

💡 **Astuce pro** :  
- Le topic `connect-offsets` (interne) contient ces offsets. Il doit avoir une réplication factor ≥ 3 et une politique `cleanup.policy=compact`.  
- Ne désactivez pas les commits d’offsets, sinon vous reconsumerez toutes les données à chaque redémarrage.

---

### 4. Sink connectors – Exporter des données depuis Kafka

Un **sink connector** lit les messages d’un ou plusieurs topics Kafka et les écrit dans un système externe.

#### 4.1. Fonctionnement interne d’une tâche sink

- Le framework fournit à la tâche une liste de `SinkRecord` (via `put()`).
- La tâche écrit ces enregistrements dans le système cible.
- Après succès, le framework commit les offsets dans Kafka (via le consommateur intégré).

📌 **Exemple simplifié** (`FileStreamSinkConnector`) :  
écrit chaque message dans un fichier texte.

💡 **Astuce pro** :  
- Évitez les traitements lourds dans `put()` qui pourraient bloquer le thread.  
- Utilisez des batchs pour améliorer le débit (regroupez plusieurs `SinkRecord` avant d’écrire).

#### 4.2. Connecteurs sinks populaires

| Connecteur | Utilisation | Options clés |
|------------|-------------|----------------|
| `ElasticsearchSinkConnector` | Indexe dans Elasticsearch | `write.method=upsert`, `type.name` |
| `JdbcSinkConnector` | Écrit dans une table SQL | `insert.mode=upsert`, `pk.mode=record_key` |
| `S3SinkConnector` | Écrit dans S3 (parquet, avro, json) | `rotate.schedule.interval.ms`, `flush.size` |
| `HDFSSinkConnector` | Écrit dans HDFS (Hadoop) | `hdfs.url`, `rotate.interval.ms` |
| `CassandraSinkConnector` | Écrit dans Cassandra | `cassandra.host`, `keyspace` |

📌 **Exemple de configuration Elasticsearch** :  
```json
{
  "name": "es-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "connection.url": "http://elasticsearch:9200",
    "topics": "my-topic",
    "key.ignore": "true",
    "schema.ignore": "true",
    "write.method": "upsert",
    "type.name": "_doc"
  }
}
```

💡 **Astuce pro** :  
- Pour les bases de données relationnelles, utilisez `insert.mode=upsert` pour mettre à jour ou insérer.  
- Le paramètre `pk.mode=record_key` permet d’utiliser la clé Kafka comme clé primaire.

#### 4.3. Dead Letter Queue (DLQ)

Quand une erreur survient lors du traitement d’un enregistrement (souvent dans un sink), on peut le rediriger vers un topic **dead letter queue** au lieu d’échouer la tâche.

```json
"errors.deadletterqueue.topic.name": "my-dlq",
"errors.deadletterqueue.context.headers.enable": "true"
```

💡 **Astuce pro** :  
- Activez la DLQ pour éviter qu’un seul record corrompu n’arrête tout le pipeline.  
- Analysez périodiquement la DLQ pour détecter des problèmes de format ou de schéma.

---

### 5. Single Message Transforms (SMT) – Transformer les messages à la volée

Les SMT permettent de modifier les messages **individuellement** pendant le passage dans le connecteur, sans code personnalisé. Ils s’appliquent aussi bien aux sources qu’aux sinks.

#### 5.1. SMT couramment utilisés

| SMT | Effet | Paramètres |
|-----|-------|-------------|
| `InsertField` | Ajoute un champ (ex: timestamp) | `static.field`, `static.value` |
| `ReplaceField` | Renomme ou supprime des champs | `renames`, `blacklist` |
| `MaskField` | Masque une partie d’un champ | `fields`, `replacement` (ex: `****`) |
| `ValueToKey` | Déplace un champ de la valeur vers la clé | `fields`, `schema.name` |
| `ExtractTopic` | Extrait le nom du topic depuis un champ | `field` |
| `Filter` | Conserve ou rejette les records selon une expression (EL) | `predicate` (ex: `value.age > 18`) |
| `TimestampConverter` | Convertit le format d’un timestamp | `target.type`, `format` |

📌 **Exemple** : masquer le champ `email` dans les records d’un source connector.  
```json
"transforms": "maskemail",
"transforms.maskemail.type": "org.apache.kafka.connect.transforms.MaskField$Value",
"transforms.maskemail.fields": "email",
"transforms.maskemail.replacement": "***"
```

💡 **Astuce pro** :  
- Les SMT sont chainés dans l’ordre de la liste `transforms` (virgule).  
- Pour des transformations complexes (jointures, fenêtres), préférez **Kafka Streams** plutôt que des SMT.

#### 5.2. Prédicats (Conditional SMT)

Depuis Kafka 2.6, vous pouvez appliquer un SMT uniquement si une condition est remplie, grâce aux **prédicats**.

📌 **Exemple** : n’ajouter un champ que pour la table `orders`.  
```json
"predicates": "isOrders",
"predicates.isOrders.type": "org.apache.kafka.connect.transforms.predicates.TopicNameMatches",
"predicates.isOrders.pattern": "db.orders",
"transforms.addField.condition": "isOrders"
```

💡 **Astuce pro** :  
- Les prédicats évitent de multiplier les connecteurs pour des traitements distincts.  
- Ils fonctionnent avec n’importe quel SMT.

---

### 6. CDC (Change Data Capture) avec Debezium

Debezium est la solution de facto pour capturer les changements des bases de données relationnelles (MySQL, PostgreSQL, Oracle, SQL Server, etc.).

#### 6.1. Principe de Debezium

- Se connecte à la base comme un **slave logique** (binlog pour MySQL, slot de réplication pour Postgres).  
- Pour chaque INSERT, UPDATE, DELETE, il génère un événement Kafka structuré contenant `before`, `after`, et des métadonnées (`op`, `ts_ms`).  
- Les événements sont publiés dans un topic nommé `server.database.table`.

📌 **Exemple de message CDC** (MySQL) :  
```json
{
  "op": "u",
  "before": {"id": 1, "name": "Alice"},
  "after": {"id": 1, "name": "Alice Updated"},
  "source": {"db": "mydb", "table": "users", "ts_ms": 1620000000000},
  "ts_ms": 1620000001000
}
```

💡 **Astuce pro** :  
- Le topic doit avoir `cleanup.policy=compact` pour garder la dernière version de chaque clé (car chaque UPDATE remplace l’ancienne).  
- Ne pas oublier de configurer `table.whitelist` pour limiter les tables capturées.

#### 6.2. Configuration typique d’un connecteur Debezium MySQL

```json
{
  "name": "mysql-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "debezium",
    "database.server.id": "184054",
    "database.server.name": "dbserver1",
    "database.include.list": "inventory",
    "table.include.list": "inventory.customers,inventory.orders",
    "topic.prefix": "dbserver1",
    "schema.history.internal.kafka.topic": "schema-changes.inventory",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092"
  }
}
```

💡 **Astuce pro** :  
- `database.server.id` doit être unique parmi tous les slaves (y compris autres connecteurs).  
- Le topic `schema-changes.inventory` stocke l’historique des modifications de schéma (nécessaire pour rester à jour).  
- Pour les grosses bases, augmentez `tasks.max` et assurez-vous que la base supporte plusieurs connexions.

#### 6.3. Gestion des événements et des clés

- Par défaut, la **clé** du message Kafka est la clé primaire de la ligne (au format structuré).  
- La **valeur** contient le `before/after` et les métadonnées.  
- Vous pouvez utiliser des SMT pour ne conserver que l’`after` ou pour extraire le champ d’intérêt.

📌 **Exemple de SMT** pour ne garder que la nouvelle valeur :  
```json
"transforms": "unwrap",
"transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
"transforms.unwrap.drop.tombstones": "true"
```

💡 **Astuce pro** :  
- Le SMT `ExtractNewRecordState` simplifie le contenu pour le sink (plus besoin de gérer `op` manuellement).  
- Pensez à gérer les suppressions : elles produisent un message `op=d` et par défaut un *tombstone*.

---

### 7. Monitoring, debugging et gestion des erreurs

#### 7.1. API REST de Kafka Connect

Kafka Connect expose une API REST (port 8083 par défaut) pour administrer les connecteurs.

| Endpoint | Action |
|----------|--------|
| `GET /connectors` | Liste des connecteurs |
| `POST /connectors` | Créer un connecteur |
| `GET /connectors/<name>/status` | État (RUNNING, FAILED, UNASSIGNED) |
| `PUT /connectors/<name>/config` | Modifier la configuration |
| `POST /connectors/<name>/restart` | Redémarrer le connecteur ou une tâche |
| `GET /connectors/<name>/tasks` | Liste des tâches et leurs statuts |

📌 **Exemple de status** (après une erreur) :  
```json
{
  "name": "my-sink",
  "connector": { "state": "RUNNING" },
  "tasks": [
    { "id": 0, "state": "FAILED", "trace": "..." }
  ]
}
```

💡 **Astuce pro** :  
- Utilisez l’API pour construire des dashboards de supervision.  
- Sur une erreur de tâche, vous pouvez la redémarrer sans arrêter le connecteur entier :  
  `POST /connectors/<name>/tasks/0/restart`

#### 7.2. Métriques JMX importantes pour Connect

| MBean | Description |
|-------|-------------|
| `kafka.connect:type=connect-worker-metrics` | Nombre de workers, rebalances, production d’offsets |
| `kafka.connect:type=connector-task-metrics,connector=<name>,task=<id>` | Traitements par seconde, erreurs |
| `kafka.connect:type=sink-task-metrics` | Lag du sink (important) |

💡 **Astuce pro** :  
- Le lag du sink est le nombre de messages en attente d’être écrits dans la cible. S’il augmente indéfiniment, le système cible est peut‑être surchargé ou en panne.  
- Activez les métriques JMX avec `KAFKA_JMX_OPTS` dans `connect-distributed.properties`.

#### 7.3. Logs et debugging

- Les logs de Connect sont dans les fichiers de log des workers (ou stdout).  
- Niveau de log par paquet (`log4j.logger.io.confluent.connect=DEBUG`).  
- Pour inspecter les topics internes (offsets, status, config) :

```bash
kafka-console-consumer --bootstrap-server localhost:9092 --topic connect-offsets --from-beginning
kafka-console-consumer --bootstrap-server localhost:9092 --topic connect-status --from-beginning
```

💡 **Astuce pro** :  
- Les topics internes doivent avoir une politique de rétention adaptée (`cleanup.policy=compact` pour `connect-offsets`).  
- Si vous désinstallez un connecteur, supprimez manuellement sa configuration dans le topic `connect-configs`.

---

### 8. Bonnes pratiques pour les connecteurs

| Pratique | Explication |
|----------|-------------|
| Utiliser des variables d’environnement | Éviter de stocker les mots de passe en clair dans la config JSON |
| Limiter les tables (`table.whitelist`) | Réduire la charge source et les topics inutiles |
| Activer la DLQ | Éviter les blocages pour des records corrompus |
| Définir `tasks.max` proportionnel | Ne pas dépasser le parallelism réel du système source/sink |
| Surveiller le lag des sinks | Détecter les bottlenecks cibles |
| Utiliser `transforms` avec parcimonie | Trop de SMT pénalise le débit (re-sérialisation à chaque étape) |
| Tester les changements en mode standalone | Puis basculer en distribué |

📌 **Exemple de configuration sécurisée (avec variables)** :  
```json
{
  "name": "jdbc-source-secure",
  "config": {
    "connection.url": "${file:/path/secrets.properties:db.url}",
    "connection.user": "${file:/path/secrets.properties:db.user}",
    "connection.password": "${file:/path/secrets.properties:db.password}"
  }
}
```

💡 **Astuce pro** :  
- Pour les environnements containerisés (Kubernetes), utilisez des secrets montés comme fichiers.  
- Le fichier de propriétés doit avoir un format `key=value`.

---

### 9. Résumé des commandes et métriques essentielles (Connect)

| Besoin | Commande / Action |
|--------|-------------------|
| Lister les connecteurs | `curl -X GET http://localhost:8083/connectors` |
| Créer un connecteur | `curl -X POST -H "Content-Type: application/json" -d @config.json http://localhost:8083/connectors` |
| Voir le statut | `curl -X GET http://localhost:8083/connectors/<name>/status` |
| Redémarrer une tâche | `curl -X POST http://localhost:8083/connectors/<name>/tasks/<taskId>/restart` |
| Pause / reprise | `curl -X PUT http://localhost:8083/connectors/<name>/config -d '{"state":"PAUSED"}'` |
| Surveiller le lag sink | Métrique JMX `records-lag` |
| Analyser les DLQ | Consommer le topic `dlq-topic` avec `kafka-console-consumer` |

---

## Conclusion du chapitre 4 – points clés

- **Source** : importe de l’extérieur → Kafka. **Sink** : exporte de Kafka → extérieur.
- **SMT** : transformations légères sans code.  
- **CDC** : Debezium capture le binlog ou la WAL ; les événements contiennent `before` et `after`.  
- **Monitoring** : API REST, JMX, et logs sont indispensables.  
- **Tolérance aux pannes** : workers distribués, DLQ, commits d’offsets.  

**Dernier chapitre (5)** : Monitoring, CLI et JMX – outils avancés du quotidien.
