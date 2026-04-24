
# 🗄️ Chapitre 9 : ksqlDB – Le moteur SQL temps réel pour Apache Kafka  
*(Version ultra-détaillée – 5 fois plus de contenu que la précédente)*

---

### 1. Qu’est‑ce que ksqlDB ?

ksqlDB est une **base de données orientée streaming** construite sur Kafka Streams. Elle permet de manipuler des flux temps réel grâce à une syntaxe **SQL simple**, sans avoir à écrire de code Java/Scala.

**Idée fondatrice** : si l’on peut interroger des données statiques avec SQL, pourquoi ne pas interroger le temps réel de la même façon, avec `SELECT, JOIN, GROUP BY` sur des flux en mouvement ?

📌 **Exemple introductif** :  
On veut compter le nombre de clics par utilisateur à la volée :
```sql
CREATE STREAM clicks (user_id VARCHAR, url VARCHAR) WITH (...);
CREATE TABLE click_counts AS
  SELECT user_id, COUNT(*) AS nb
  FROM clicks
  GROUP BY user_id;
```
Chaque nouvelle entrée met automatiquement à jour `click_counts`. On peut ensuite interroger cette table en tout temps pour obtenir le compteur de n’importe quel utilisateur.

💡 **ksqlDB n’est pas juste un terminal SQL sur Kafka** :  
C’est un **moteur de streaming à part entière**, scalable, tolérant aux pannes, capable de matérialiser des vues et de servir des millions de requêtes par seconde.

---

### 2. ksqlDB vs Kafka Streams – Quand choisir quoi ?

Les deux technologies sont construites sur Kafka Streams, mais s’adressent à des usages très différents.

| Critère | ksqlDB | Kafka Streams |
|---------|--------|----------------|
| **Public cible** | Analystes, ingénieurs SQL, data engineers | Développeurs JVM |
| **Courbe d’apprentissage** | Faible (SQL) | Élevée (API Streams Processor) |
| **Flexibilité** | Limitée à ce que SQL permet | Très grande (API processeur) |
| **Performance pure** | Très bonne, mais un peu de surcoût | Optimale, surcoût quasi nul |
| **Intégration** | Native avec Kafka, Connect, SR | Native, mais plus bas niveau |
| **Déploiement** | Serveur(s) ksqlDB + cluster Kafka | Aucun serveur supplémentaire |
| **Cas typique** | Dashboards en ligne, ETL rapide, prototypage | Streaming avancé, routage conditionnel, logique métier complexe |

> 💡 **Règle d’or** : ksqlDB pour la rapidité de mise en œuvre ; Kafka Streams pour la finesse de contrôle.

ksqlDB est idéal pour :

- Les pipelines ETL temps réel sans code ;
- La démocratisation du streaming auprès des équipes data (SQL‑first)；
- Le prototypage rapide de topologies complexes；
- Les vues matérialisées accessibles par requêtes `pull`.

Kafka Streams est préférable quand :

- On a besoin d’**intégrer une librairie** dans une application existante (micro‑service) ;
- On veut un **contrôle fin** sur l’état, les fenêtres, le temps ;
- On ne peut pas se permettre la latence ou la surcharge d’un serveur additionnel.

> ⚠️ ksqlDB **reste cantonné à l’écosystème Kafka** ; pour des sources en dehors de Kafka, mieux vaut un outil comme Flink ou Spark.

---

### 3. Architecture et concepts clés

#### 3.1. Streams, tables, matérialisations

- **Stream** : flux ininterrompu d’événements, append‑only, immuable.  
    📌 Composez `CREATE STREAM ...` sur un topic Kafka.

- **Table** : représentation mutable du dernier état de chaque clé (concept semblable à une table SQL classique).  
    📌 Exemple : `employee_table` contenant le dernier salaire de chaque employé.

- **Vue matérialisée** : table dérivée d’un stream ou d’une autre table, maintenue en temps réel.  
    📌 L’exemple `click_counts` ci‑dessus est une **table matérialisée**.

#### 3.2. Trois types de requêtes

- **Persistent queries** : exécutées en continu sur le serveur, elles transforment des flux et produisent des flux ou tables. Syntaxe = `CREATE STREAM/TABLE ... AS SELECT ...`.  
    → Remplacent des topologies Kafka Streams.

- **Push queries** : requêtes **émettent des résultats incrémentaux** à mesure que de nouveaux événements arrivent. Ouvertes via `EMIT CHANGES`.  
    📌 `SELECT * FROM orders EMIT CHANGES;` → affiche chaque nouvelle commande.

- **Pull queries** : requêtes classiques **ponctuelles** sur l’état d’une table matérialisée, réponse immédiate.  
    📌 `SELECT * FROM click_counts WHERE user_id = 'bob';` → retourne le compteur instantané.

#### 3.3. Exécution distribuée

Le serveur ksqlDB s’exécute en cluster. Chaque requête persistante est parallélisée en fonction du nombre de partitions des topics sources. Les tâches (tasks), héritées de Kafka Streams, tournent dans les serveurs. La haute disponibilité pour les requêtes `pull` s’obtient en activant la réplication des state stores.

---

### 4. Jointures et fenêtres de temps

ksqlDB implémente plusieurs **types de fenêtres** pour les agrégations.

#### 4.1. Fenêtres Tumbling (fenêtres fixes, sans chevauchement)

Découpage du temps en intervalles contigus (ex: 0h‑5h, 5h‑10h, 10h‑15h...).

```sql
CREATE TABLE sales_per_5min AS
  SELECT region, SUM(amount)
  FROM sales
  WINDOW TUMBLING (SIZE 5 MINUTES)
  GROUP BY region;
```

- **Chaque événement appartient à une et une seule fenêtre**.
- L’agrégat se termine quand la fenêtre est close.

💡 **Accès rapide** : la table résultante est partitionnée par clé + fenêtre.

#### 4.2. Fenêtres Hopping (fenêtres chevauchantes)

Taille fixe mais un pas de décalage plus petit. Utile pour les moyennes glissantes.

```sql
WINDOW HOPPING (SIZE 10 MINUTES, ADVANCE BY 1 MINUTE)
```

- Une fenêtre se déplace toutes les minutes, chaque minute une nouvelle valeur est calculée.
- Même événement peut contribuer à plusieurs fenêtres.

📌 **Exemple concret** : température moyenne des 10 dernières minutes, rafraîchie chaque minute.

#### 4.3. Fenêtres Session (périodes d’inactivité)

Fenêtre délimitée par une **absence d’événement** pendant une durée donnée.

```sql
WINDOW SESSION (5 MINUTES)
```

- Dès qu’un événement survient, une session s’ouvre. Si aucun événement pendant 5 minutes, la session se ferme.
- Idéal pour analyser le comportement utilisateur (parse de session web, temps passé sur une application).

#### 4.4. Gestion du temps et `grace period`

Par défaut, les fenêtres utilisent le timestamp du message (`ROWTIME`). Pour tolérer des messages en retard, on peut ajouter un **grace period** :

```sql
WINDOW HOPPING (SIZE 10 MINUTES, ADVANCE BY 1 MINUTE, GRACE PERIOD 2 MINUTES)
```

Les messages jusqu’à 2 minutes après la fermeture de la fenêtre **seront encore pris en compte**. Au delà, ils sont ignorés.

> Ce paramètre évite la croissance indéfinie de l’état tout en acceptant du désordre modéré.

#### 4.5. Types de jointures supportées

- `JOIN` entre deux streams (fenêtré) ;
- `JOIN` entre un stream et une table (enrichissement) ;
- `JOIN` entre deux tables (fusion d’états) ;
- `JOIN` avec clé étrangère (depuis ksqlDB 0.24).

📌 **Exemple d’enrichissements dynamiques** :

```sql
-- table des clients (dernière version)
CREATE TABLE customers (id INT PRIMARY KEY, name VARCHAR) WITH (...);

-- flux des commandes
CREATE STREAM orders (cust_id INT, amount DECIMAL) WITH (...);

-- commandes enrichies avec le nom client
CREATE STREAM enriched_orders AS
  SELECT o.cust_id, c.name, o.amount
  FROM orders o
  JOIN customers c ON o.cust_id = c.id;
```

Enrichissement **en temps réel**, sans code.

---

### 5. Déploiement en production – haute disponibilité et configuration

#### 5.1. Modes d’exécution

- **Single‑server** (mode local) : un seul serveur, pour les tests, le staging ou les petits pipelines.
- **Cluster multi‑servers** (mode distributed) : plusieurs serveurs ksqlDB partageant la charge, avec tolérance aux pannes.

#### 5.2. Activer la haute disponibilité

La HA (High Availability) est **désactivée par défaut** ; on l’active dans `ksql-server.properties` :

```properties
ksql.query.pull.table.high.availability.enabled=true
```

**Conditions à respecter** :

- Au moins **2 serveurs ksqlDB** dans le cluster.
- Utiliser **le groupe de consommation `_confluent-ksql-query-<random>`** plutôt que le défaut.
- Dans Confluent Cloud, les clusters de ≥ 8 CSU activent automatiquement la HA.

#### 5.3. Configuration essentielle côté serveur

```properties
# Nom court pour l’identité du cluster ksqlDB
ksql.service.id=ksql_cluster_production

# Tampon de mémoire pour les état stores (RocksDB)
ksql.streams.state.dir=/mnt/ssd/ksql-state-store

# Rétention des topics internes (en ms)
ksql.streams.commit.interval.ms=10000

# Garantie exactly‑once (via transactions)
ksql.streams.processing.guarantee=exactly_once_v2

# Nombre total de threads
ksql.streams.num.stream.threads=8
```

#### 5.4. Démarrage en cluster

1. Créer un topic `_confluent-ksql-<service_id>` (géré automatiquement).
2. Lancer chaque serveur avec les mêmes paramètres `bootstrap.servers`, `ksql.service.id` et `ksql.streams.application.id`.
3. Les serveurs s’auto‑découvrent en lisant le profil du topic.

> Une fois le cluster démarré, toute requête persistante est **distribuée** entre les serveurs. En cas de panne d’un serveur, la charge est rééquilibrée, et les requêtes `pull` restent disponibles via les répliques des state stores.

---

### 6. Intégration avec Kafka Connect – ETL complet sans code

ksqlDB peut **embarquer nativement n’importe quel connecteur** Kafka Connect.

📌 **Pipeline ETL** :  
Source (base de données) → ksqlDB → transformations → sink (Elasticsearch, S3, etc.)

#### 6.1. Créer un connecteur source depuis ksqlDB

```sql
CREATE SOURCE CONNECTOR jdbc_mysql WITH (
  'connector.class'          = 'io.confluent.connect.jdbc.JdbcSourceConnector',
  'connection.url'           = 'jdbc:mysql://mysql:3306/test',
  'topic.prefix'             = 'jdbc-',
  'mode'                     = 'incrementing',
  'incrementing.column.name' = 'id',
  'tasks.max'                = '2'
);
```

**Une fois créé**, les données arrivent automatiquement dans Kafka sous les topics `jdbc-...`. ksqlDB peut immédiatement les lire.

#### 6.2. Créer un connecteur sink

```sql
CREATE SINK CONNECTOR es_sink WITH (
  'connector.class'          = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
  'connection.url'           = 'http://elasticsearch:9200',
  'topics'                   = 'click_counts',
  'key.ignore'               = 'true',
  'schema.ignore'            = 'true'
);
```

> Le sink exporte la table `click_counts` en temps réel vers Elasticsearch. Dès qu’une nouvelle valeur de compteur est calculée, elle est immédiatement indexée.

#### 6.3. Gérer tout l’ETL SQL‑first

L’avantage est de **centraliser** dans un seul langage :

- L’ingestion (source connector)
- Les transformations (SQL)
- Les jointures, agrégations, fenêtres
- L’export (sink connector)

🔁 **Résultat** : un pipeline ETL complet, versionné dans les scripts SQL.

---

### 7. Monitoring et performance – que surveiller ?

#### 7.1. Métriques clés (JMX + API REST)

| Métrique / Outil | Description | Seuil d’alerte |
|------------------|-------------|----------------|
| `ksql.queries.persistent.count` | Nombre de requêtes persistantes en cours | Augmentation inattendue |
| `ksql.queries.persistent.error_rate` | Taux d’erreur des requêtes | > 0 |
| `ksql.streams:type=stream‑thread` `task‑latency‑avg` | Latence moyenne de traitement | ≥ 500 ms |
| `kafka.consumer:type=consumer‑fetch‑manager‑metrics` `records‑lag‑max` | Lag des topics d’entrée | > 10 000 |
| Utilisation CPU / mémoire des serveurs ksqlDB | — | > 80 % |

#### 7.2. Observabilité via Confluent Control Center

- Dashboard **ksqlDB** dédié : liste des requêtes, débit, lag.
- Graphique des **emissions** (messages en sortie par requête).
- Historique des changements de schémas (Schema Registry).

#### 7.3. Commandes CLI de diagnostic

```bash
# Lancer l’interface interactive
ksql http://localhost:8088

# Lister toutes les requêtes persistantes
SHOW QUERIES;

# Expliquer le plan d’exécution d’une requête
EXPLAIN SELECT * FROM clicks WHERE url LIKE '%shop%' EMIT CHANGES;

# Visualiser les topics internes créés par ksqlDB
SHOW TOPICS;

# Arrêter une requête persistante (peut prendre un certain temps)
TERMINATE QUERY <query_id>;
```

---

### 8. ksqlDB vs Flink SQL – comparaison au regard du marché

ksqlDB et Flink SQL sont deux moteurs SQL‑over‑streaming, mais leurs domaines d’excellence diffèrent.

| Critère | ksqlDB | Flink SQL |
|---------|--------|-----------|
| **Écosystème natif** | Kafka uniquement | Plus de 30 connecteurs (Kafka, Pulsar, Kinesis, JDBC…) |
| **Déploiement** | Serveur unique ou cluster | Cluster très scalable, avec isolation de job |
| **Planification des ressources** | Statique (par tâches) | Dynamique, adaptable à la charge |
| **Support Exactly‑once** | Oui (via transactions Kafka) | Oui, plus robuste sur plusieurs types de sources |
| **Complexité opérationnelle** | Faible (SQL only) | Plus élevée (gestion de flink cluster) |
| **Communauté** | Limitée (Confluent) | Très vaste (Apache, Alibaba, AWS, etc.) |

📌 **Quand choisir ksqlDB ?**

- Toute l’infrastructure **tourne déjà sur Kafka**.
- On souhaite la **plus grande simplicité** (pas besoin d’un cluster Flink).
- Les équipes sont **compétentes en SQL** mais pas en Java Streams.

📌 **Quand préférer Flink SQL ?**

- On a des sources/sinks **hors Kafka** (Kinesis, Pulsar, RDBMS en streaming, etc.).
- La charge est très variable, on veut **une scalabilité dynamique**.
- On prépare un avenir multi‑moteurs (Kafka + autres brokers).

> ⚠️ Confluent investit davantage dans Flink (conjointement) que dans ksqlDB aujourd’hui ; l’avenir de ksqlDB est principalement **niche Kafka** / usage interne Confluent Cloud.

---

### 9. Bonnes pratiques et pièges à éviter

#### ✅ À faire

- **Toujours définir les `PRIMARY KEY`** pour les tables (partitionnement correct des requêtes persistantes).
- **Utiliser `CREATE OR REPLACE`** pour les évolutions de requêtes (met à jour le plan sans arrêt complet).
- **Tester avec `EXPLAIN`** avant de déployer une requête lourde (vérifier la cardinalité des jointures).
- **Configurer `ksql.streams.commit.interval.ms`** entre 2000 et 10000 ms (équilibre latence / reprise).
- **Activer la haute disponibilité** sur les clusters d’au moins 3 serveurs ksqlDB.

#### ❌ À éviter

- **Ne jamais exécuter de requête `EMIT CHANGES`** sur une table qui reçoit des millions d’événements par seconde sans filtre — vous submergeriez le client.
- **Ne pas laisser de requêtes persistantes inutiles tourner** (consomment des ressources et des topics internes).
- **Ne pas ignorer le `grace period`** : trop grand, l’état grossit indéfiniment ; trop petit, les données tardives sont perdues.
- **Ne pas exécuter ksqlDB sur les mêmes brokers Kafka** – isolez les ressources CPU/mémoire des serveurs ksqlDB.

---

### 10. Scripts d’exemple complets (prêts à utiliser)

#### 10.1. Pipeline entier : source JDBC → transformation → sink Elasticsearch

```sql
-- Source connector (MySQL)
CREATE SOURCE CONNECTOR mysql_source WITH (
  'connector.class'          = 'io.confluent.connect.jdbc.JdbcSourceConnector',
  'connection.url'           = 'jdbc:mysql://mysql:3306/customers',
  'topic.prefix'             = 'mysql-',
  'table.whitelist'          = 'orders',
  'mode'                     = 'incrementing',
  'incrementing.column.name' = 'id'
);

-- Stream à partir du topic généré
CREATE STREAM raw_orders (order_id INT, customer_id INT, amount DECIMAL) 
  WITH (KAFKA_TOPIC='mysql-orders', VALUE_FORMAT='AVRO');

-- Agrégation par client
CREATE TABLE total_per_customer AS
  SELECT customer_id, SUM(amount) AS total
  FROM raw_orders
  GROUP BY customer_id;

-- Sink connector vers Elasticsearch
CREATE SINK CONNECTOR es_sink WITH (
  'connector.class' = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
  'connection.url' = 'http://elasticsearch:9200',
  'topics'         = 'TOTAL_PER_CUSTOMER'
);
```

Après l’exécution, tout pipeline tourne en production.

#### 10.2. Monitoring des départs en session

```sql
CREATE STREAM user_activity (user_id VARCHAR, action VARCHAR) WITH (...);

-- Sessions de 15 minutes d'inactivité
CREATE TABLE user_sessions AS
  SELECT user_id,
         COUNT(*) AS actions_per_session,
         WINDOWSTART() AS session_start,
         WINDOWEND()   AS session_end
  FROM user_activity
  WINDOW SESSION (15 MINUTES)
  GROUP BY user_id;
```

---

### 11. Feuille de route : de zéro à l’expertise ksqlDB

| Niveau | Compétences | Durée estimée |
|--------|-----------|----------------|
| **Découverte** | Comprendre la notion de stream/table, exécuter une première requête push | 2 heures |
| **Intermédiaire** | Maîtriser les trois types de fenêtres, les jointures, utiliser Kafka Connect | 2 jours |
| **Avancé** | Optimiser les performances, déployer un cluster HA, mettre en place la HA pour les pull queries | 1 semaine |
| **Expert** | Écrire des requêtes complexes (sous‑requêtes, fonctions lambda), exploiter les VW (materialized views) sur d’énormes volumes | 3 mois |

---

## Conclusion du chapitre 9 – points clés

- **ksqlDB est un moteur SQL temps réel** bâti sur Kafka Streams, accessible sans code.
- Ses trois types de requêtes (**persistent, push, pull**) couvrent analytique continue, streaming interactif et accès ponctuel.
- **Jointures et fenêtres** (tumbling, hopping, session) sont supportées nativement.
- Intégré à **Kafka Connect** → ETL complet sans écriture de code.
- **HA pour les requêtes `pull`** active manuellement, indispensable en production.
- **Comparé à Flink SQL**, ksqlDB reste plus simple sur Kafka pur, mais moins extensible au‑delà.

Avec ce chapitre, vous maîtrisez l’outil SQL‑first pour le streaming, idéal pour les pipelines en self‑service et les tableaux de bord en temps réel. 🚀
