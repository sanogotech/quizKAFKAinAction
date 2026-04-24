
## 🚀 Chapitre 10 : Tuning Production, résolution d’incidents et préparation à la production  
*(Version ultra-détaillée – 5 fois plus de contenu que la précédente)*

---

### 1. Introduction – D’un POC à un cluster de production robuste

Un cluster Kafka en production doit être **dimensionné**, **configuré**, **surveillé** et **prêt à faire face aux incidents** sans intervention humaine immédiate. Ce chapitre couvre :

- Le dimensionnement des brokers (CPU, RAM, disque, réseau)
- L’optimisation des producteurs / consommateurs / streams
- La gestion des partitions et de la réplication
- La configuration JVM et OS pour la performance
- Les meilleures pratiques pour éviter les incidents courants
- Les runbooks et la résolution des problèmes typiques
- Les outils de diagnostic avancés

💡 **Objectif** : Transformer un cluster Kafka « qui marche » en un cluster « qui ne tombe jamais » et qui se répare tout seul.

---

### 2. Dimensionnement des brokers – calculs et règles empiriques

#### 2.1. CPU (cores)

Kafka traite principalement deux types d’opérations :

- **Réseau** (encodage/décodage des requêtes) → threads réseau (`num.network.threads`, défaut 3 per broker)
- **Disque** (écriture/lecture des logs) → threads I/O (`num.io.threads`, défaut 8)

> La plupart des workloads sont **I/O bound**, sauf la compression/décompression (gzip, lz4, zstd) qui consomme du CPU.

**Formule pratique** :  
- 2 cores pour le syscall et la réplication, plus 1 core par 200‑400 MB/s de débit attendu.  
- Pour 1 GB/s de trafic total, 6‑8 cores sont recommandés.

📌 **Exemple** :  
Débit écriture max : 600 MB/s, débit lecture : 400 MB/s → total 1 GB/s → 6 à 8 vCPU.

💡 **Astuce pro** :  
Dans les environnements virtualisés, les cores « bruyants » (noisy neighbors) peuvent dégrader les performances. Isolez les brokers sur des cœurs dédiés si possible (CPU pinning).

#### 2.2. RAM – bien plus que la mémoire Java heap

La mémoire totale du broker se répartit :

- **Heap JVM** (recommandé : **5‑8 GB**, pas plus) – pour les structures de données, les caches clients, les requêtes temporaires. **Un heap trop grand augmente les pauses GC**.
- **Page cache OS** – le reste de la RAM (souvent 70‑80 %) est utilisé par le système d’exploitation pour cacher les segments de log. **C’est le secret des performances de lecture de Kafka**.

**Recommandation** :  
- Pour 1 Gb/s de débit, au moins 16 GB de RAM (dont 6 GB heap, 10 GB page cache).  
- Pour des clusters très sollicités (3 Gb/s), passer à 32 GB ou 64 GB.

💡 **Astuce pro** :  
Ne laissez jamais la swap active sur les machines brokers. Désactivez‑la (`swapoff -a`) ou réglez `vm.swappiness=1`.

#### 2.3. Disque – IOPS et latence

Les disques doivent être **rapides** (SSD NVMe recommandés). Les disques magnétiques (HDD) ne supportent que des débits modestes (∼100 MB/s) et des latences élevées.

- **IOPS** : plus de 10 000 IOPS pour des clusters haute densité.
- **Throughput** : minimum 300 MB/s par broker pour de l’écriture soutenue.
- **Mount noatime** : `mount -o noatime` pour éviter les mises à jour d’accès inutiles.

💡 **Astuce pro** :  
Utilisez plusieurs disques (raid 0 si risque contrôlé, sinon JBOD avec réplication Kafka). Les performances augmentent avec le nombre de disques.

#### 2.4. Réseau – bande passante et latence

- **Trafic entrant** (productions) + **trafic de réplication** (interne) + **trafic des consommateurs**.
- La bande passante recommandée est **le double** du débit maximal attendu, pour absorber les pics.

Exemple : débit produit 500 MB/s, débit consommé 300 MB/s → trafic inter‑broker (réplication) ∼500 MB/s (si facteur 3). Total ∼1,3 GB/s. Prévoir une carte réseau 2.5 Gb/s ou 10 Gb/s.

💡 **Astuce pro** :  
Activez le `jumbo frames` (MTU 9000) pour réduire le nombre de paquets.

---

### 3. Optimisations producteurs – débit ou latence ?

#### 3.1. Réglages pour le débit maximal

```properties
acks=0                # ou 1 selon la tolérance
batch.size=131072     # 128 KB
linger.ms=100         # attendre 100ms pour remplir le lot
compression.type=zstd # meilleure compression
max.in.flight.requests.per.connection=5
buffer.memory=67108864 # 64 MB
```

- **batch.size** : plus il est grand, plus l’envoi est efficace. Attention à la mémoire.
- **linger.ms** : sacrificiel pour la latence, vital pour le débit (permet d’agréger des petits messages).

#### 3.2. Réglages pour la plus faible latence

```properties
acks=1
linger.ms=0
batch.size=16384
compression.type=none
max.in.flight.requests.per.connection=1
```

- `linger.ms=0` → envoi immédiat, chaque message part dans sa propre requête (overhead).
- Pas de compression – le CPU est sollicité, mais la latence est plus faible.

#### 3.3. Idempotence et exactly‑once – impact sur les performances

L’activation de `enable.idempotence=true` impose :

- `acks=all`
- `max.in.flight.requests.per.connection ≤ 5`
- Une légère dégradation du débit (5‑15 % selon les cas).

Les transactions (exactly‑once) ajoutent une overhead supplémentaire : chaque transaction ajoute deux messages de contrôle. **Pour des débits massifs (> 1 million msg/s), évitez les transactions ou utilisez `exactly_once_v2`** dans Kafka Streams.

---

### 4. Optimisations consommateurs – éviter les lags et les rebalances

#### 4.1. Augmenter le parallélisme

- **Nombre de consommateurs** ≤ nombre de partitions.
- **Partition Count** : doit être au minimum le nombre total de threads consommateurs.

#### 4.2. Réglages du `poll`

```properties
max.poll.records=500            # ne pas saturer la mémoire
fetch.min.bytes=1               # réveil rapide (si latence critique)
fetch.max.wait.ms=500           # attendre au max 500ms
max.partition.fetch.bytes=1048576 # 1 Mo
```

- `max.poll.records` trop grand → traitement trop long → risque de rebalance.
- `max.poll.interval.ms` (défaut 5 min) – à augmenter si traitement long.

#### 4.3. Gestion des rebalances

- Utilisez la stratégie **CooperativeStickyAssignor** (disponible depuis 2.4).
- Réduisez `session.timeout.ms` à 10‑30 secondes pour détecter plus vite les consommateurs morts, mais ajustez `heartbeat.interval.ms` à un tiers de cette valeur.
- Évitez les `onPartitionsRevoked` trop lourds.

#### 4.4. Monitoring des lag

La commande de base :

```bash
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group monGroupe
```

**Seuils d’alerte** :  
- Lag persistant > 50 000 → déclassé.
- Lag croissant linéairement → consommateur trop lent (besoin d’ajouter plus de partitions ou de consommateurs).

💡 **Astuce pro** :  
Utilisez **Burrow** (LinkedIn) ou `kafka-consumer-groups` automatisé (cron) pour envoyer les métriques à Prometheus.

---

### 5. Configuration JVM et OS – les leviers cachés

#### 5.1. GC et heap

- **G1GC** recommandé pour les heaps > 8 GB ; **ParallelGC** pour heaps < 8 GB.
- Limiter le heap à 6‑8 GB maximum pour éviter les pauses GC trop longues.

Exemple de `KAFKA_HEAP_OPTS` :

```bash
export KAFKA_HEAP_OPTS="-Xms6G -Xmx6G -XX:+UseG1GC -XX:MaxGCPauseMillis=20"
```

#### 5.2. OS – paramètres réseau et file descriptors

```bash
# Augmenter le nombre de fichiers ouverts
ulimit -n 1000000

# Réglages TCP
net.core.somaxconn = 32768
net.ipv4.tcp_max_syn_backlog = 8192
net.core.netdev_max_backlog = 5000
```

- `somaxconn` : taille de la file d’attente de connexions.
- `tcp_rmem` / `tcp_wmem` : au moins 4 MB.

#### 5.3. Système de fichiers – XFS recommandé

- XFS offre de meilleures performances que ext4 sous charge Kafka.
- Montage avec `noatime, nodiratime, nobarrier` (attention au risque de perte en cas de panne).

---

### 6. Gestion des partitions et réplication – prévenir les déséquilibres

#### 6.1. Répartition initiale des partitions

Utilisez `kafka-reassign-partitions --generate` pour obtenir une distribution **équilibrée**.

#### 6.2. Surveillance des `UnderReplicatedPartitions`

```bash
kafka-topics --bootstrap-server localhost:9092 --describe --under-replicated-partitions
```

Si > 0, une partition a moins de réplicas que prévu → cause : broker mort, disque plein, réseau lent.

#### 6.3. Rééquilibrage des leaders (Préférée leadeur election)

```bash
kafka-leader-election --bootstrap-server localhost:9092 --election-type preferred --all-topic-partitions
```

Planifiez cette commande chaque nuit pour éviter qu’un broker ne soit surchargé de leadership.

---

### 7. Runbook de production – les scénarios critiques

Un runbook est un document actionnable que les équipes d’exploitation suivent en cas d’incident. Voici les quatre incidents les plus fréquents et leur résolution.

#### 🚨 Incident 1 : Sous‑réplication persistante

**Symptômes** :  
- `UnderReplicatedPartitions` > 0 depuis plus de 10 min.  
- Les logs broker montrent `ReplicaFetcherThread` en erreur.

**Causes** :

- Un broker en panne ou déconnecté.
- Un disque de follower saturé (IO waiting).
- Réseau saturé entre les brokers.

**Actions** :

1. **Identifier les brokers défaillants** :  
   `kafka-broker-api-versions --bootstrap-server ...`  
2. **Redémarrer le follower** si nécessaire.  
3. **Augmenter les requêtes de réplication** (`replica.fetch.max.bytes`, `replica.fetch.wait.max.ms`).  
4. **En dernier recours** : réassigner la partition sur un autre broker via `kafka-reassign-partitions`.

#### 🚨 Incident 2 : Consumer lag explosif

**Symptômes** :  
- Lag d’un groupe qui double toutes les heures.  
- CPU du consommateur à 100 %.  
- Pas d’erreur dans les logs.

**Actions rapides** :

1. **Ajouter plus de consommateurs** :  
   ```bash
   # augmenter le nombre d’instances du consommateur
   docker-compose scale consumer=3
   ```
2. **Si la cause est une partition unique** (hot partition), augmenter le nombre de partitions du topic :  
   ```bash
   kafka-topics --alter --topic mytopic --partitions 20
   ```
   (cela ne répartit pas les anciennes données, mais aide pour le futur.)
3. **Accélérer le traitement** : baisser `max.poll.records` pour réduire la taille des lots, ou passer à un traitement parallèle multi‑thread.

#### 🚨 Incident 3 : Rebalance intempestive (groupe instable)

**Symptômes** :  
- `RebalanceInProgressException` dans les logs.  
- Le consommateur perd sa partition plusieurs fois par heure.

**Causes** :

- `max.poll.interval.ms` trop court.
- Un consommateur prend plus de 5 minutes à traiter un lot.
- Heartbeat manqué.

**Réglages** :

```properties
max.poll.interval.ms=600000   # 10 minutes
session.timeout.ms=45000
heartbeat.interval.ms=10000
```

**Action immédiate** : redémarrer les consommateurs pour forcer une reprise.

#### 🚨 Incident 4 : Boîte aux lettres pleine (disque broker saturé)

**Symptômes** :  
- Erreurs `Disk full` dans les logs broker.  
- Le broker refuse les nouvelles écritures.

**Actions** :

1. **Supprimer manuellement des segments** (danger !) ou **augmenter la rétention**.  
   ```bash
   kafka-configs --alter --add-config retention.ms=86400000 --topic mytopic
   ```
2. **Ajouter un nouveau disque** et déplacer certaines partitions.  
3. **Activer la compression** sur les topics qui ne l’utilisent pas (réduit l’espace).

---

### 8. Outils de diagnostic avancés – la boîte à outils du vétéran

#### 8.1. Logs de mutation (`kafka-dump-log`)

Inspectez un segment de log :

```bash
kafka-dump-log --files /var/lib/kafka/data/mytopic-0/00000000000000000000.log --deep-iteration
```

Utile pour trouver des messages corrompus, des offsets qui sautent, ou des incohérences index/data.

#### 8.2. Vérification des réplicas (`kafka-replica-verification`)

```bash
kafka-replica-verification --bootstrap-server localhost:9092 --report-interval-ms 5000
```

Ceci affiche en continu le décalage max entre tous les réplicas. Si un réplica est en retard de manière permanente, il y a un problème de réseau ou de disque.

#### 8.3. Analyse des `stack traces` des threads brokers

```bash
kill -3 <pid_broker>   # génère un thread dump dans logs/server.log
```

Cherchez les threads bloqués (deadlock) ou les longues opérations (`ReplicaFetcherThread` bloqué).

#### 8.4. Simuler des pannes avec `chaos`

Pour entraîner votre runbook :

- Tuer aléatoirement des brokers (`kill -9`)
- Désactiver le réseau d’un follower (`iptables -D`)
- Ralentir le disque via `cgroup` (throttle)

---

### 9. Checklist de passage en production

Avant de dire « le cluster est prêt pour la production », vérifiez ces points :

| Domaine | Élément vérifié |
|---------|----------------|
| **Réseau** | Les adresses `advertised.listeners` sont bien résolubles par tous les clients. |
| **Sécurité** | TLS activé, authentification configurée, ACL définies pour chaque application. |
| **Réplication** | `replication.factor` ≥ 3, `min.insync.replicas` = 2 pour les topics critiques. |
| **Retention** | Aucun topic n’a de rétention infinie sans surveillance. |
| **Monitoring** | Prometheus + Grafana, alerts sur `UnderReplicatedPartitions`, `ConsumerLag` et `DiskUsage`. |
| **Backup des offsets** | Les offsets sont stockés dans un topic (automatique) ; prévoyez un outil de sauvegarde externe. |
| **Healthchecks** | Les brokers répondent à un `curl` sur `/v3/brokers` (Admin API). |
| **Documentation** | Runbook accessible, schéma du cluster, informations de contact des équipes. |

---

### 10. Exemple complet : résolution d’un incident réel

**Situation** :  
Un topic `transactions` montre un lag de 500 000 messages sur une partition spécifique. La consommation est vitale pour le pipeline fraudes.  

**Étapes** :

1. **Identifier la partition à problème** :  
   ```bash
   kafka-consumer-groups --describe --group fraud-group
   ```
   Résultat : partition 2 lag = 500k, les autres 0.

2. **Examiner la configuration du consommateur** :  
   `max.poll.records` = 500, traitement moyen par message = 200ms → 500 * 200 ms = 100s de traitement → > `max.poll.interval.ms`(60s). ⇒ Rebalance permanente.

3. **Solution temporaire** :  
   Augmenter `max.poll.interval.ms` à 180 secondes.

4. **Solution pérenne** :  
   - Augmenter le nombre de partitions du topic (passer de 3 à 12).  
   - Réassigner les données existantes avec un script de rejeu.  
   - Passer le traitement en parallèle (augmenter le nombre de threads du consommateur).

5. **Vérification** :  
   Lag descend à 0 après une heure. Plus de rebalance.

---

### 11. Outils utiles – résumé pratique

| Outil | Usage |
|-------|-------|
| `kafka-consumer-groups` | Lag, réinitialisation d’offsets |
| `kafka-reassign-partitions` | Rééquilibrage |
| `kafka-dump-log` | Inspection de segment, recherche de corruption |
| `kafka-replica-verification` | Détection de réplication en retard |
| `jstack`, `jmap` | Diagnostique JVM (thread dump, heap) |
| `iostat`, `iftop`, `netstat` | Performance disque/réseau |

---

## Conclusion du chapitre 10 – ce que vous devez absolument retenir

- Un cluster Kafka fiable se **dimensionne** (CPU, RAM, disque, réseau) et se **configure** (JVM, OS) avant tout.
- Les **producteurs** doivent être réglés selon votre compromis débit/latence.
- Les **consommateurs** doivent éviter les rebalances avec des réglages adaptés et un partitionnement suffisant.
- La surveillance (`UnderReplicatedPartitions`, lag, disque) est obligatoire.
- Un **runbook** d’incident (sous‑réplication, lag, rebalance, disque plein) vous sauve la vie.
- Les **outils de diagnostic** (`dump‑log`, `replica-verification`) sont vos meilleurs alliés face à des anomalies silencieuses.

Avec ce dixième chapitre, vous possédez un ensemble complet pour passer de la théorie de la certification à l’opération de clusters Kafka robustes en production. Félicitations ! 🚀
