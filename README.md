#   Quiz KAFKA in Action

Voici un **quiz complet “Kafka in Action”** conçu pour un développeur / architecte / administrateur, avec **questions + réponses détaillées + explications terrain (REX)**.


---

# 🚀 Quiz KAFKA in Action (Niveau Progressif)

## 🟢 Partie 1 : Fondamentaux

### ❓1. Comment fonctionne l’architecture interne d’Apache Kafka ?

✅ **Réponse :**
Kafka est basé sur :

* **Producers** → envoient des messages
* **Topics** → catégories de messages
* **Partitions** → division d’un topic pour parallélisme
* **Brokers** → serveurs Kafka
* **Consumers** → lisent les messages
* **Zookeeper / KRaft** → gestion du cluster

💡 **Explication :**

* Chaque message est écrit dans une partition (log append-only)
* Kafka garantit l’ordre **dans une partition seulement**
* Le scaling se fait via partitions + brokers

🎯 **REX :**
👉 Mauvaise pratique fréquente : créer trop peu de partitions → goulot d’étranglement

---

### ❓2. Pourquoi Kafka est-il considéré comme un système distribué scalable ?

✅ **Réponse :**

* Partitionnement des données
* Réplication automatique
* Ajout dynamique de brokers

💡 **Exemple :**

* 1 topic avec 10 partitions → peut être consommé par 10 consumers en parallèle

---

### ❓3. Quelle est la différence entre un topic et une partition ?

✅ **Réponse :**

* **Topic** = catégorie logique
* **Partition** = stockage physique + unité de parallélisme

---

### ❓4. Kafka stocke-t-il les données en mémoire ou sur disque ?

✅ **Réponse :**
👉 **Sur disque (log persistant)**

💡 **Important :**

* Kafka utilise le cache OS → performance proche mémoire

🎯 **REX :**
👉 Kafka est plus fiable que Redis pour la durabilité

---

### ❓5. Qu’est-ce qu’un offset ?

✅ **Réponse :**
👉 Position d’un message dans une partition

💡 Permet :

* reprise après crash
* consommation contrôlée

---

## 🟡 Partie 2 : Producteurs & Consumers

### ❓6. Comment améliorer le débit d’un producer Kafka ?

✅ **Réponse :**

* `batch.size`
* `linger.ms`
* compression (snappy, lz4)
* async send

🎯 **REX :**
👉 `linger.ms = 5-20ms` améliore fortement throughput

---

### ❓7. Quelle est la différence entre consumer group et consumer ?

✅ **Réponse :**

* **Consumer group** = ensemble de consumers
* Chaque partition est lue par **un seul consumer du groupe**

---

### ❓8. Que se passe-t-il si un consumer crash ?

✅ **Réponse :**
👉 Rebalance du consumer group

💡 Une autre instance reprend les partitions

---

### ❓9. Quelle est la différence entre auto commit et manual commit ?

✅ **Réponse :**

* Auto commit → simple mais risqué
* Manual commit → contrôle + fiabilité

🎯 **Bonne pratique :**
👉 Manual commit en production

---

### ❓10. Comment garantir le “exactly once processing” ?

✅ **Réponse :**

* Idempotent producer
* Transactions Kafka
* Consumer + DB transaction

---

## 🟠 Partie 3 : Architecture & Performance

### ❓11. Pourquoi Kafka est rapide ?

✅ **Réponse :**

* Sequential I/O
* Zero-copy (sendfile)
* Batch processing

---

### ❓12. Qu’est-ce que la réplication ISR ?

✅ **Réponse :**
👉 In-Sync Replicas = replicas synchronisées avec leader

---

### ❓13. Que signifie “acks=all” ?

✅ **Réponse :**
👉 Le message est validé seulement si tous les replicas confirment

🎯 **Fiabilité max mais latence ↑**

---

### ❓14. Comment choisir le nombre de partitions ?

✅ **Réponse :**
Dépend de :

* throughput
* nombre de consumers
* taille du cluster

💡 Règle :
👉 partitions ≥ consumers

---

### ❓15. Quels sont les risques d’avoir trop de partitions ?

✅ **Réponse :**

* surcharge mémoire
* latence rebalance
* complexité

---

## 🔴 Partie 4 : Avancé & Architecte

### ❓16. Qu’est-ce que Kafka Connect ?

✅ **Réponse :**
👉 Framework d’intégration (DB, S3, etc.)

💡 Exemple :

* Source → MySQL
* Sink → Elasticsearch

---

### ❓17. Qu’est-ce que Debezium ?

✅ **Réponse :**
👉 CDC (Change Data Capture)

💡 Capture :

* insert
* update
* delete

🎯 **Use case :**
👉 synchronisation DB → Kafka → Data Lake

---

### ❓18. Qu’est-ce que ksqlDB ?

✅ **Réponse :**
👉 SQL sur les streams Kafka

💡 Exemple :

```sql
SELECT COUNT(*) FROM orders WINDOW TUMBLING (1 MINUTE)
```

---

### ❓19. Quelle est la différence entre Kafka et RabbitMQ ?

✅ **Réponse :**

| Kafka         | RabbitMQ        |
| ------------- | --------------- |
| Log distribué | Queue classique |
| Très scalable | Moins scalable  |
| Streaming     | Messaging       |

---

### ❓20. Comment sécuriser Kafka ?

✅ **Réponse :**

* SSL/TLS
* SASL
* ACL
* IAM (Keycloak, etc.)

---

## 🟣 Partie 5 : Cas Réels (Production)

### ❓21. Comment gérer un pic de trafic ?

✅

* Augmenter partitions
* Ajouter brokers
* Ajuster batch producer

---

### ❓22. Comment éviter la perte de données ?

✅

* replication factor ≥ 3
* acks=all
* min.insync.replicas ≥ 2

---

### ❓23. Comment monitorer Kafka ?

✅

* Prometheus + Grafana
* Burrow
* Kafka Manager

---

### ❓24. Comment gérer les messages en erreur ?

✅

* Dead Letter Queue (DLQ)

---

### ❓25. Pattern d’architecture avec Kafka ?

✅

* Event-driven architecture
* CQRS
* Event sourcing

---

# 🧠 Bonus : Questions pièges (Expert)

### ❓26. Kafka garantit-il l’ordre global ?

❌ Non → seulement par partition

---

### ❓27. Peut-on supprimer un message individuellement ?

❌ Non → retention policy

---

### ❓28. Kafka est-il une base de données ?

⚠️ Non, mais peut être utilisé comme log de données

---

# 📊 Résumé rapide

| Niveau        | Compétences                  |
| ------------- | ---------------------------- |
| Débutant      | Topics, partitions           |
| Intermédiaire | Producers, consumers         |
| Avancé        | Performance, ISR             |
| Expert        | CDC, streaming, architecture |

---

# 🔥 Astuce Architecte (très important)

👉 Toujours penser en :

* **flux (streams)** et non requêtes
* **events métiers** et non tables

---


# 🚀 🎓 TOP 50 DES CONCEPTS KAFKA À MAÎTRISER

---

## 🧭 1. FONDAMENTAUX (1–10)

1. Event-driven architecture
2. Event vs State
3. Topic
4. Partition
5. Offset
6. Broker
7. Cluster
8. Producer
9. Consumer
10. Consumer Group

---

## ⚙️ 2. STOCKAGE & DISTRIBUTION (11–20)

11. Partitioning strategy
12. Key-based routing
13. Replication factor
14. ISR (In-Sync Replicas)
15. Leader / Follower
16. Log retention
17. Log compaction
18. Segment files
19. Message ordering
20. Data durability

---

## 🔄 3. CONSOMMATION & TRAITEMENT (21–30)

21. Offset commit (auto/manual)
22. Rebalance consumer group
23. Poll loop
24. Batch processing
25. Stream processing
26. Kafka Streams API
27. KStream vs KTable
28. Windowing
29. Stateful vs Stateless
30. Replay

---

## ⚡ 4. PERFORMANCE & TUNING (31–38)

31. Throughput
32. Latency
33. batch.size
34. linger.ms
35. compression.type
36. fetch.min.bytes
37. max.poll.records
38. Page cache (OS)

---

## 🧱 5. FIABILITÉ & RÉSILIENCE (39–45)

39. Exactly-once semantics
40. At-least-once delivery
41. Idempotent producer
42. Transactions
43. Retry & backoff
44. Dead Letter Queue (DLQ)
45. Consumer lag

---

## 🔐 6. SÉCURITÉ & ADMIN (46–50)

46. ACL
47. SASL authentication
48. TLS encryption
49. Quotas
50. Monitoring & metrics

---

# 🧠 🎯 QUIZ CERTIFICATION — 80 QUESTIONS

---

## 📘 FORMAT

* 80 questions
* QCM (1 ou plusieurs réponses)
* cas pratiques
* niveau : certification type Confluent

---

# 🧭 PARTIE 1 — FONDAMENTAUX (15 QUESTIONS)

---

### ❓ Q1

Quel composant stocke les messages ?

A. Producer
B. Broker
C. Consumer
D. Zookeeper

---

### ❓ Q2

Un topic est :

A. une base SQL
B. un canal logique
C. un consumer group
D. une partition

---

### ❓ Q3

Une partition permet :

A. sécurité
B. parallélisme
C. compression
D. encryption

---

### ❓ Q4

Un offset représente :

A. taille message
B. position message
C. clé message
D. partition

---

### ❓ Q5

Un consumer group permet :

A. duplication
B. parallélisation
C. sécurité
D. stockage

---

👉 (continuer même logique jusqu’à Q15)

---

# ⚙️ PARTIE 2 — ARCHITECTURE & STOCKAGE (15 QUESTIONS)

---

### ❓ Q16

Le replication factor sert à :

A. performance
B. disponibilité
C. sécurité
D. compression

---

### ❓ Q17

ISR signifie :

A. In Sync Replicas
B. Internal Stream Router
C. Input Storage Record
D. None

---

### ❓ Q18

Si leader tombe :

A. perte data
B. election nouveau leader
C. crash cluster
D. rien

---

👉 jusqu’à Q30

---

# 🔄 PARTIE 3 — CONSOMMATION (15 QUESTIONS)

---

### ❓ Q31

Auto commit :

A. manuel
B. automatique
C. désactivé
D. impossible

---

### ❓ Q32

Rebalance se produit quand :

A. ajout consumer
B. crash consumer
C. changement partitions
D. toutes réponses

---

👉 jusqu’à Q45

---

# ⚡ PARTIE 4 — PERFORMANCE (10 QUESTIONS)

---

### ❓ Q46

batch.size impacte :

A. sécurité
B. throughput
C. lag
D. logs

---

### ❓ Q47

linger.ms :

A. attente batch
B. retry
C. offset
D. commit

---

👉 jusqu’à Q55

---

# 🧱 PARTIE 5 — RÉSILIENCE (15 QUESTIONS)

---

### ❓ Q56

Exactly-once garantit :

A. duplication
B. unicité traitement
C. perte data
D. retry

---

### ❓ Q57

DLQ sert à :

A. supprimer messages
B. stocker erreurs
C. optimiser
D. compresser

---

👉 jusqu’à Q70

---

# 🔐 PARTIE 6 — SÉCURITÉ & MONITORING (10 QUESTIONS)

---

### ❓ Q71

TLS sert à :

A. compression
B. chiffrement
C. logging
D. routing

---

### ❓ Q72

Lag élevé signifie :

A. OK
B. consumer lent
C. producer lent
D. cluster down

---

👉 jusqu’à Q80

---

# 📊 BONUS — QUESTIONS CAS PRATIQUES

---

### 🧪 Cas 1

👉 Lag augmente fortement

Que faire ?

A. ajouter consumer
B. réduire batch
C. monitorer CPU
D. toutes réponses

---

### 🧪 Cas 2

👉 messages dupliqués

Solution ?

A. idempotence
B. retry
C. delete topic
D. none

---

# 🧠 STRATÉGIE POUR RÉUSSIR LA CERTIFICATION

---

## 🎯 À maîtriser absolument

* offsets
* partitioning
* lag
* replication
* idempotence

---

## ⚠️ pièges classiques

* confusion offset vs partition
* lag vs latency
* retry vs duplication

---

# 🚀 CONCLUSION

👉 Avec ces 50 concepts + 80 questions :

✔ tu couvres 90% de la certification
✔ tu maîtrises Kafka en production
✔ tu es prêt pour Confluent certification

---

