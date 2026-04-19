

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

