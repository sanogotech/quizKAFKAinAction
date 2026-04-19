

# 🚀 Quiz Kafka Expert DSI (50 Questions)

---

## 🔵 1. Gouvernance & Urbanisation SI

### ❓1. Pourquoi Kafka est un pilier de l’urbanisation SI moderne ?

✅ **Réponse :**
👉 Permet découplage via événements

💡 **Explication :**

* Remplace ESB rigide
* Favorise Event-Driven Architecture

🎯 **REX :**
👉 Dans un SI avec +100 apps, Kafka réduit dépendances fortes

---

### ❓2. Quelle différence entre ESB et Kafka ?

✅

* ESB = orchestration centralisée
* Kafka = chorégraphie distribuée

---

### ❓3. Quand Kafka devient un anti-pattern ?

✅

* Mauvaise gouvernance des topics
* Explosion des événements inutiles

🎯 **REX :**
👉 “Event spaghetti” fréquent sans référentiel métier

---

### ❓4. Qu’est-ce qu’un “event contract” ?

✅
👉 Schéma d’événement versionné

💡 Exemple :

* Avro + Schema Registry

---

### ❓5. Pourquoi versionner les événements ?

✅
👉 Éviter la casse des consumers

---

## 🟡 2. Data & Streaming Strategy

### ❓6. Kafka peut-il remplacer un Data Warehouse ?

❌ Non

👉 Complément (streaming vs analytique)

---

### ❓7. Différence entre streaming et batch ?

✅

* Streaming = temps réel
* Batch = différé

---

### ❓8. Qu’est-ce que le “data in motion” ?

✅
👉 Données en transit dans Kafka

---

### ❓9. Pourquoi Kafka est clé pour le Data Mesh ?

✅
👉 Domaines publient leurs événements

---

### ❓10. Kafka dans un Data Lake ?

✅
👉 ingestion temps réel

---

## 🟠 3. Performance & Scalabilité

### ❓11. Comment dimensionner un cluster Kafka ?

✅

* Throughput attendu
* Taille messages
* Rétention

---

### ❓12. Facteurs limitant performance ?

✅

* réseau
* disque
* partitions

---

### ❓13. Pourquoi SSD recommandé ?

✅
👉 I/O séquentiel + rapide

---

### ❓14. Impact du nombre de partitions ?

✅

* partitions = + parallélisme
  mais surcharge metadata

---

### ❓15. Comment éviter le “hot partition” ?

✅
👉 bonne clé de partitionnement

🎯 **REX :**
👉 Mauvais choix → 1 partition saturée

---

## 🔴 4. Résilience & Continuité

### ❓16. Que faire en cas de perte d’un broker ?

✅
👉 Leader failover automatique

---

### ❓17. Stratégie multi-datacenter ?

✅
👉 MirrorMaker 2

---

### ❓18. Kafka supporte-t-il DR (Disaster Recovery) ?

✅ Oui

---

### ❓19. RPO/RTO avec Kafka ?

✅

* RPO ≈ 0 avec réplication
* RTO faible

---

### ❓20. Comment tester la résilience ?

✅
👉 Chaos engineering

---

## 🟣 5. Sécurité & Conformité

### ❓21. Comment sécuriser les données ?

✅

* TLS
* encryption at rest

---

### ❓22. Gestion des accès ?

✅
👉 ACL Kafka

---

### ❓23. Kafka + IAM ?

✅
👉 intégration avec Keycloak

---

### ❓24. Risque principal sécurité ?

✅
👉 exposition des brokers

---

### ❓25. Kafka et conformité (RGPD) ?

✅
👉 gestion rétention + anonymisation

---

## 🟤 6. Exploitation & Monitoring

### ❓26. KPI clés Kafka ?

✅

* lag consumer
* throughput
* latency

---

### ❓27. Outils de monitoring ?

✅

* Prometheus
* Grafana

---

### ❓28. Qu’est-ce que le consumer lag ?

✅
👉 retard de consommation

---

### ❓29. Danger du lag élevé ?

✅
👉 backlog + perte métier

---

### ❓30. Logging Kafka ?

✅
👉 logs brokers + audit

---

## ⚫ 7. Architecture Avancée

### ❓31. Kafka dans microservices ?

✅
👉 backbone événementiel

---

### ❓32. Pattern CQRS avec Kafka ?

✅
👉 lecture/écriture séparées

---

### ❓33. Event sourcing ?

✅
👉 stockage des événements comme source de vérité

---

### ❓34. Saga pattern avec Kafka ?

✅
👉 orchestration distribuée

---

### ❓35. Kafka vs API REST ?

✅

* Kafka = async
* REST = sync

---

## 🟢 8. Intégration & Écosystème

### ❓36. Kafka Connect vs API ?

✅
👉 automatisation intégration

---

### ❓37. Cas d’usage Debezium ?

✅
👉 capture DB en temps réel

---

### ❓38. Kafka + IoT ?

✅
👉 ingestion massive capteurs

---

### ❓39. Kafka + IA ?

✅
👉 pipeline data temps réel

---

### ❓40. Kafka + ESB coexistence ?

✅
👉 phase transitoire possible

---

## 🔵 9. Coûts & ROI

### ❓41. Coût caché Kafka ?

✅

* infrastructure
* expertise

---

### ❓42. Kafka open source vs Confluent ?

✅

* OSS = gratuit
* Confluent = enterprise features

---

### ❓43. ROI Kafka ?

✅
👉 réduction couplage + temps réel

---

### ❓44. Quand Kafka est inutile ?

✅
👉 faible volumétrie

---

### ❓45. Coût du sur-dimensionnement ?

✅
👉 gaspillage infra

---

## 🔶 10. Pièges & Bonnes pratiques

### ❓46. Erreur classique ?

✅
👉 trop de topics

---

### ❓47. Mauvais design événement ?

✅
👉 trop verbeux

---

### ❓48. Mauvaise stratégie retention ?

✅
👉 perte ou explosion stockage

---

### ❓49. Documentation Kafka ?

✅
👉 indispensable (catalogue événements)

---

### ❓50. Clé de succès Kafka ?

✅
👉 gouvernance + architecture claire

---

# 🧠 Conclusion Expert DSI

👉 Kafka n’est pas juste un outil technique
👉 C’est un **changement de paradigme SI** :

* Passage **API → Event**
* Passage **synchrone → asynchrone**
* Passage **monolithique → distribué**

---

# 🔥 Bonus (vision stratégique)

Un DSI doit maîtriser :

* Urbanisation Event-Driven
* Gouvernance des événements
* Data streaming stratégique
* Sécurité & conformité


---

# 🎓 Examen Blanc Kafka DSI – Version Approfondie (20 Cas)

---

# 🔵 CAS 1 : Saturation du Producer (Débit insuffisant)

## 🔎 Problème

Dans un système de facturation (type énergie / télécom), le producer doit envoyer **50 000 événements/seconde** (consommation clients).
👉 Mais le débit plafonne à **10 000 msg/s**, provoquant :

* accumulation en amont (microservices bloqués)
* latence globale du SI
* SLA non respecté

---

## 🧠 Démarche de diagnostic (DSI)

1. Vérifier métriques producer :

   * throughput
   * batch size réel
2. Analyser :

   * CPU / RAM producer
   * latence réseau vers brokers
3. Vérifier config Kafka :

   * `acks`
   * compression
4. Observer côté broker :

   * saturation disque ?
   * throttling ?

---

## ✅ Solution

Optimisations clés :

* `batch.size` ↑ (32KB → 128KB+)
* `linger.ms = 5–20ms`
* compression `lz4` ou `snappy`
* activer **envoi asynchrone**
* augmenter parallélisme (multi threads)

---

## 🎯 REX

👉 Dans un projet telco, simple tuning (`linger.ms + compression`)
➡️ débit multiplié par **x4 sans changer infra**

---

## 💡 Tips Architecte

✔ Toujours optimiser producer AVANT d’ajouter des brokers
✔ Kafka est rapide… mais mal utilisé → lent

---

# 🟡 CAS 2 : Consumer Lag Critique (Backlog massif)

## 🔎 Problème

Un système de paiement accumule **2 millions de messages en retard**.
👉 Impact :

* transactions traitées en retard
* incohérences métiers
* risque financier

---

## 🧠 Diagnostic

1. Mesurer :

   * consumer lag
   * processing time
2. Identifier :

   * lenteur applicative ?
   * DB downstream ?
3. Vérifier :

   * nombre de partitions vs consumers

---

## ✅ Solution

* Augmenter nombre de consumers
* Augmenter partitions (si nécessaire)
* Optimiser traitement (batch / async)
* Découpler DB (cache, queue interne)

---

## 🎯 REX

👉 80% des cas :
❌ Kafka n’est PAS le problème
✅ La base de données derrière est le goulot

---

## 💡 Tips

✔ Toujours analyser **end-to-end** (Kafka + app + DB)
✔ Kafka révèle les problèmes… ne les crée pas

---

# 🟠 CAS 3 : Mauvais Partitionnement (Hot Partition)

## 🔎 Problème

Un topic a 10 partitions, mais :

* 1 partition reçoit 80% du trafic
* les autres sont quasi vides

👉 Résultat :

* latence élevée
* déséquilibre cluster

---

## 🧠 Diagnostic

* Analyser clé de partition
* vérifier distribution des clés

---

## ✅ Solution

* changer clé de partitionnement (hash équilibré)
* introduire clé composite (userId + timestamp)
* recréer topic si nécessaire

---

## 🎯 REX

👉 Mauvais choix classique :

* pays → déséquilibre
* statut → déséquilibre

👉 Bon choix :

* userId
* transactionId

---

## 💡 Tips

✔ Le partitionnement = **décision critique d’architecture**
✔ Toujours tester distribution AVANT prod

---

# 🔴 CAS 4 : Perte de Données après Crash

## 🔎 Problème

Après crash d’un broker, des données critiques disparaissent.

---

## 🧠 Diagnostic

* vérifier :

  * `acks`
  * replication factor
  * ISR

---

## ✅ Solution

* `acks=all`
* replication factor = 3
* `min.insync.replicas = 2`

---

## 🎯 REX

👉 Beaucoup de SI critiques tournent encore en `acks=1`
➡️ risque énorme invisible

---

## 💡 Tips

✔ Toujours arbitrer : **latence vs fiabilité**
✔ En finance/énergie → priorité fiabilité

---

# 🟣 CAS 5 : Multi Datacenter / Continuité

## 🔎 Problème

Une entreprise (banque / énergie) doit :

* continuer en cas de panne pays
* synchroniser données entre 2 régions

---

## 🧠 Diagnostic

* besoin RPO/RTO ?
* latence acceptable ?

---

## ✅ Solution

* utiliser MirrorMaker 2
* architecture active-passive ou active-active

---

## 🎯 REX

👉 En Afrique : coupures réseau fréquentes
➡️ Multi-DC indispensable

---

## 💡 Tips

✔ Tester PRA régulièrement
✔ Ne pas attendre incident réel

---

# ⚫ CAS 6 : Explosion des Topics (Chaos)

## 🔎 Problème

+2000 topics sans gouvernance :

* noms incohérents
* duplication
* incompréhension métier

---

## 🧠 Diagnostic

* audit catalogue Kafka
* identifier redondances

---

## ✅ Solution

* mettre en place gouvernance :

  * naming convention
  * validation DSI
  * catalogue événements

---

## 🎯 REX

👉 “Kafka devient inutilisable sans gouvernance”

---

## 💡 Tips

✔ Créer un **Event Catalog** dès le début
✔ Kafka = produit DATA, pas juste infra

---

# 🟢 CAS 7 : Rupture de Compatibilité

## 🔎 Problème

Un changement JSON casse plusieurs consumers.

---

## 🧠 Diagnostic

* absence de versioning
* pas de schéma

---

## ✅ Solution

* utiliser schema registry (Avro)
* versionnement backward compatible

---

## 🎯 REX

👉 Cause majeure d’incidents prod

---

## 💡 Tips

✔ Traiter événements comme API publiques

---

# 🔵 CAS 8 : Latence Élevée

## 🔎 Problème

Latence > 2 secondes sur flux temps réel

---

## 🧠 Diagnostic

* mesurer :

  * producer latency
  * broker latency
  * consumer latency

---

## ✅ Solution

* réduire batch côté consumer
* optimiser réseau
* vérifier disque

---

## 🎯 REX

👉 Latence souvent liée au consumer, pas Kafka

---

## 💡 Tips

✔ Mesurer chaque étape

---

# 🟡 CAS 9 : Rebalance Fréquent

## 🔎 Problème

Consumers se reconnectent souvent → instabilité

---

## 🧠 Diagnostic

* autoscaling ?
* crash containers ?

---

## ✅ Solution

* stabiliser infra
* ajuster `session.timeout.ms`

---

## 🎯 REX

👉 Kubernetes mal configuré = rebalance constant

---

## 💡 Tips

✔ Kafka aime la stabilité

---

# 🟠 CAS 10 : Data Lake Temps Réel

## 🔎 Problème

Besoin d’alimenter Data Lake en temps réel

---

## 🧠 Diagnostic

* volumétrie
* fréquence

---

## ✅ Solution

* Kafka Connect (S3/HDFS)
* ingestion streaming

---

## 🎯 REX

👉 Remplace ETL batch nocturne

---

## 💡 Tips

✔ Kafka = colonne vertébrale data

---

# 🔴 CAS 11 : Sécurité Faible

## 🔎 Problème

Cluster Kafka accessible sans authentification

---

## 🧠 Diagnostic

* audit accès
* scan réseau

---

## ✅ Solution

* TLS
* SASL
* ACL
* IAM avec Keycloak

---

## 🎯 REX

👉 Risque critique souvent sous-estimé

---

## 💡 Tips

✔ Kafka = données sensibles → sécuriser comme DB

---

# 🟣 CAS 12 : Saturation Disque

## 🔎 Problème

Stockage plein → arrêt Kafka

---

## 🧠 Diagnostic

* retention mal définie

---

## ✅ Solution

* config retention (temps / taille)
* log compaction

---

## 🎯 REX

👉 Incident fréquent en prod

---

## 💡 Tips

✔ Surveiller stockage en permanence

---

# ⚫ CAS 13 → 20 (Synthèse avancée rapide)

---

### CAS 13 : CDC base de données

✅ Solution : Debezium
🎯 REX : évite polling DB

---

### CAS 14 : KPI temps réel

✅ Solution : ksqlDB
💡 streaming analytics

---

### CAS 15 : Cloud hybride

✅ réplication inter-zone

---

### CAS 16 : Monitoring

✅ Prometheus + Grafana

---

### CAS 17 : SLA critique

✅ scaling + priorisation topics

---

### CAS 18 : Mauvais usage

❌ Kafka ≠ base transactionnelle

---

### CAS 19 : Messages trop lourds

✅ externaliser payload

---

### CAS 20 : Migration ESB

✅ progressive (coexistence)

---

# 🧠 Conclusion DSI

👉 Les vrais enjeux Kafka ne sont PAS :

* uniquement techniques

👉 MAIS :

* gouvernance
* architecture
* organisation
* data strategy

---

# 🔥 Vision Architecte

Un bon DSI Kafka doit :
✔ penser **flux métier**
✔ maîtriser **data streaming**
✔ imposer **gouvernance forte**
✔ sécuriser et monitorer

---

