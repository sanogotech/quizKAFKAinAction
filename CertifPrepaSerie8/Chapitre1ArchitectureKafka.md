## 📘 Chapitre 1 : Architecture Kafka – Partitions, Réplication, ISR, KRaft  
*(Version ultra-détaillée – 5 fois plus de contenu que la précédente)*

---

### 1. Partitions – Fondements et comportements avancés

Une partition est une **unité fondamentale de stockage et de parallélisme** dans Kafka. Chaque partition est un **fichier journal append-only** (on ne modifie jamais un message déjà écrit) sur le disque du broker. Ce journal est découpé en **segments** (fichiers `.log`, `.index`, `.timeindex`).

#### 1.1. Structure interne d’une partition
- **Log segment** : fichier contenant les messages (en format binaire). Taille par défaut : 1 Go (`log.segment.bytes`). Quand un segment est plein, Kafka en crée un nouveau.
- **Offset index** (`.index`) : mapping offset → position physique dans le segment. Permet une recherche rapide par offset.
- **Timestamp index** (`.timeindex`) : mapping timestamp → offset. Permet de chercher des messages par temps.
- **Leader epoch cache** : protège contre les incohérences après une élection de leader.

📌 **Exemple concret** :  
Topic `orders` avec 3 partitions.  
Partition 0 : 5 segments (1 Go chacun) contenant les messages 0 à 4 999 999.  
L’offset 1 234 567 se trouve dans le segment 2. Le fichier index de ce segment contient une entrée "1 234 567 → position 8 192", ce qui évite de scanner tout le segment.

💡 **Astuce pro** :  
- Utilisez `kafka-dump-log --files <segment>.index` pour visualiser l’index.  
- Listez tous les segments avec `ls -la /var/lib/kafka/data/topic-0/`.  
- La taille des segments influence la vitesse de nettoyage (segments trop petits → trop de fichiers, segments trop grands → purge plus lourde).

#### 1.2. Choix du nombre de partitions – calculs et compromis

**Formule de dimensionnement** :  
`Nombre de partitions = max( débit d'écriture / débit par partition , débit de lecture / débit par consommateur )`

- **Débit par partition** : en moyenne 10–20 MB/s en écriture (selon compression, type de disque). Sur des SSD NVMe, on peut atteindre 50-100 MB/s par partition.
- **Parallelisme de consommation** : 1 partition lue par 1 thread de consommateur maximum.

📌 **Exemple dimensionnement** :  
Besoin : ingérer 800 MB/s. Un broker avec 10 partitions à 80 MB/s chacune est irréaliste (la plupart des brokers plafonnent à 200-400 MB/s total). Il faut donc **plusieurs brokers**.  
Avec 10 brokers, 20 partitions par broker = 200 partitions totales. Chaque partition gère 4 MB/s → acceptable.

**Limites à ne pas dépasser** :
- Chaque partition augmente la mémoire du leader (replica fetcher) et le temps d’élection du leader.
- Le nombre total de partitions par broker ne devrait pas dépasser **4000** dans les clusters standards, 10000 dans les très gros clusters avec 64 Go de RAM.

💡 **Astuce pro** :  
- Commencez avec 3 partitions par topic, surveillez la charge, puis augmentez.  
- Il est **impossible** de réduire le nombre de partitions (sauf à supprimer le topic). Planifiez dès le départ le maximum souhaité.  
- Utilisez `kafka-reassign-partitions --generate` pour redistribuer les partitions si certains brokers sont surchargés.

#### 1.3. Ordonnancement des messages – clés et partitionnement

- **Avec clé** : `hash(clé) % nombre de partitions` → toutes les messages d’une même clé vont dans la même partition → **ordre garanti** pour cette clé.
- **Sans clé** (et partitioner par défaut "sticky") : les messages sont envoyés en batch vers une partition, puis on bascule vers une autre de façon à remplir les batches, améliorant la compression.

📌 **Exemple** :  
Topic `clicks` avec 4 partitions. Clé `session_id=abc123`. Tous les clics de cette session atterrissent dans la partition 2 (car hash("abc123") % 4 = 2). Ainsi on peut reconstruire l’ordre complet des actions de la session.

**Problème** : Si une clé reçoit beaucoup plus de messages que les autres, une partition devient chaude (skew). Solution : ajouter un salt (`clé|rand`) pour répartir, mais on perd l’ordre.

💡 **Astuce pro** :  
- Pour des clés très chaudes, utilisez une stratégie de **sous‑partitionnement** : ajoutez un suffixe aléatoire pour répartir, et une étape de regroupement ensuite (ex: dans Kafka Streams).  
- Le partitionner `UniformStickyPartitioner` (par défaut depuis 2.4) améliore le batching par rapport à l’ancien round‑robin.

---

### 2. Réplication – Modes, configurations et scénarios de panne

La réplication garantit la **tolérance aux pannes** et la **haute disponibilité**.

#### 2.1. Rôles – leader vs followers

- **Leader** : seul à accepter les écritures (`ProduceRequest`) et les lectures (`FetchRequest` par défaut). Les consommateurs peuvent aussi lire depuis les followers (option `client.rack` + configuration `broker.rack` pour la localité).
- **Followers** : envoient périodiquement des `FetchRequest` au leader pour copier les nouveaux messages.

📌 **Exemple** :  
Broker 101 (leader), Broker 102 et 103 (followers).  
Le producteur écrit sur 101. 102 demande régulièrement les messages. 101 lui renvoie les messages non encore vus. 102 les écrit sur son disque et accuse réception (mais n’envoie pas d’ACK au producteur). Seul le leader envoie l’ACK au producteur après avoir respecté `min.insync.replicas`.

#### 2.2. Paramètres essentiels de réplication

| Paramètre | Défaut | Rôle |
|-----------|--------|------|
| `replication.factor` | 1 (topic) | Nombre total de copies (leader + followers) |
| `min.insync.replicas` | 1 (topic) | Nombre minimal de réplicas ISR qui doivent accuser réception pour valider une écriture |
| `replica.fetch.max.bytes` | 1 MB | Taille max d’une requête de réplication |
| `replica.fetch.wait.max.ms` | 500 ms | Temps d’attente max si aucun nouveau message |
| `replica.lag.time.max.ms` | 30 s | Temps max depuis la dernière requête du follower pour rester dans l’ISR |

📌 **Scénario classique** :  
`replication.factor=3`, `min.insync.replicas=2`.  
- Si 1 broker (le follower) est lent, l’écriture est quand même validée (car 2 réponses reçues : leader + l’autre follower). Le follower lent sort de l’ISR, mais le topic reste disponible.
- Si 2 brokers tombent, `min.insync.replicas` ne peut être satisfait → le producteur reçoit `NotEnoughReplicasException`.

💡 **Astuce pro** :  
- Pour les topics critiques (finances, logs métier), utilisez `min.insync.replicas=2` avec RF=3.  
- Surveillez la métrique `kafka.controller:type=ControllerStats,name=UnderReplicatedPartitions`. Une valeur > 0 indique qu’au moins une partition a moins de réplicas disponibles que prévu.  
- La commande `kafka-topics --describe --topic <topic>` montre les réplicas actuels et l’ISR. Un réplica absent de l’ISR signifie qu’il est en retard.

#### 2.3. Gestion des élections de leader – le rôle du contrôleur

Le **contrôleur** (un broker parmi tous, élu via ZooKeeper ou KRaft) détecte la mort d’un broker (via heartbeat manqués). Il sélectionne un nouveau leader parmi les ISR de chaque partition dont le leader est mort.

📌 **Scénario de panne** :  
Broker 101 (leader de 20 partitions) tombe. Le contrôleur (sur broker 105) reçoit la notification. Pour chaque partition : il choisit un follower dans l’ISR -> par ex broker 102 pour partition 0, broker 103 pour partition 1, etc. Il persiste la nouvelle metadata (`leader_epoch`) et la propage à tous les brokers. Les producteurs/consommateurs reçoivent une erreur `NotLeaderForPartitionException` et rafraîchissent leurs métadonnées.

💡 **Astuce pro** :  
- Le temps d’élection est typiquement de quelques secondes. Accélérez‑le en réduisant `zookeeper.session.timeout.ms` (ZooKeeper mode) ou `broker.session.timeout.ms` (KRaft).  
- Surveillez la métrique `kafka.controller:type=ControllerStats,name=LeaderElectionRate`. Une haute fréquence d’élections indique une instabilité.

#### 2.4. Rééquilibrage manuel des leaders (preferred replica election)

Le **preferred replica** est la première réplique de la liste des réplicas d’une partition. Après des redémarrages, le leader peut être sur un broker non préféré, ce qui déséquilibre la charge. L’outil `kafka-leader-election --election-type preferred` rétablit le leader sur le broker préféré.

📌 **Exemple** :  
Partition 0 : réplicas = [101, 102, 103]. Leader = 102 (à cause d’un redémarrage). Une élection préférée repasse le leader sur 101.

💡 **Astuce pro** :  
- Activez `auto.leader.rebalance.enable=true` sur les brokers pour qu’ils rééquilibrent périodiquement (toutes les `leader.imbalance.check.interval.seconds`).  
- Mais attention : une réélection fréquente peut provoquer des petits blips. Privilégiez une exécution planifiée (nuit).

---

### 3. ISR – In-Sync Replicas : le cœur de la durabilité

#### 3.1. Qu’est-ce que l’ISR exactement ?

Un réplica est dans l’ISR si :
- Il a envoyé une requête `FetchRequest` au leader dans les `replica.lag.time.max.ms` (défaut 30s).
- Son offset est au moins égal à `highWatermark - 1` (pratiquement à jour).
- Il ne présente aucun retard de réplication significatif.

L’ISR est **dynamique** : des followers peuvent en sortir (en cas de ralentissement) et y rentrer (après avoir rattrapé).

📌 **Exemple concret** :  
Leader offset = 1000, highWatermark = 990.  
Follower A : offset 1000 → ISR.  
Follower B : offset 980 (en panne réseau passagère) → sort de l’ISR. Après récupération, il rattrape et repasse à offset 1000 → rentre dans l’ISR.

💡 **Astuce pro** :  
- La sortie de l’ISR n’est pas dangereuse en soi, mais une partition **sans ISR** (ISR vide) est inerte (plus d’écriture possible). Cela arrive si tous les followers sont hors ISR et que le leader tombe.  
- Utilisez `unclean.leader.election.enable=false` pour ne jamais élire un non-ISR comme leader, au prix d’une indisponibilité temporaire.

#### 3.2. High Watermark (HWM) et Last Stable Offset (LSO)

- **High Watermark** : le plus petit offset parmi tous les ISR. Les consommateurs ne peuvent lire que les messages avec offset < HWM. Cela garantit qu’un message lu est déjà dupliqué sur tous les ISR.
- **Last Stable Offset** : pour les transactions, le plus haut offset des messages **commités** (les messages de transactions non commitées ou abortées ne sont pas visibles en `read_committed`).

📌 **Exemple** :  
Leader offset = 1000, ISR offsets = [1000, 995, 992] → HWM = 992 (la deuxième follower est en retard). Un consommateur ne lira que les messages 0 à 991. Le message 995 n’est pas encore lisible, bien qu’il existe sur le leader.

💡 **Astuce pro** :  
- Pour réduire l’écart entre HWM et leader offset, réduisez `replica.lag.time.max.ms` ou augmentez le débit réseau.  
- Surveillez `kafka.log:type=Log,name=Offsets` pour visualiser HWM des partitions.

#### 3.3. Impact de `min.insync.replicas` sur la performance

- `min.insync.replicas=1` (défaut) : le leader accuse réception seul. Si le leader tombe juste après, les messages non répliqués sont perdus.  
- `min.insync.replicas=2` : au moins 2 réplicas (dont le leader) doivent avoir écrit le message. Le leader attend l’ACK du second follower avant d’envoyer l’ACK au producteur → latence augmentée.

📌 **Benchmark** :  
En local, avec `minISR=1` : latence médiane 2 ms. Avec `minISR=2` sur 3 brokers : latence 4-5 ms. Le débit maximal baisse de ~30% en raison de l’attente additionnelle.

💡 **Astuce pro** :  
- Pour les applications où la **moindre perte de données est inacceptable** (paiements, commandes), utilisez `minISR=2` et RF=3.  
- Pour les metrics ou logs non critiques, restez à 1.

---

### 4. KRaft – La révolution sans ZooKeeper

KRaft (Kafka Raft) remplace ZooKeeper pour la **gestion des métadonnées** (topics, partitions, ISR, ACL, quotas). Un ensemble de **contrôleurs** (au lieu de brokers) utilise le protocole Raft pour maintenir un journal de métadonnées.

#### 4.1. Architecture d’un cluster KRaft

- **Nœuds contrôleurs** (`process.roles=controller`) : élisent un leader via Raft, stockent le journal des métadonnées, répondent aux requêtes des brokers.
- **Brokers** (`process.roles=broker`) : exécutent les partitions, envoient des heartbeats aux contrôleurs.
- **Mode hybride** (un seul processus fait contrôleur+broker) : possible pour les petits clusters, mais déconseillé en production.

📌 **Exemple de configuration** :  
`server.properties` pour un contrôleur pur :  
```properties
process.roles=controller
node.id=1
controller.quorum.voters=1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093
log.dirs=/data/kraft-controller-logs
```

Broker :  
```properties
process.roles=broker
node.id=101
controller.quorum.voters=1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093
```

💡 **Astuce pro** :  
- Utilisez un nombre **impair de contrôleurs** (3 ou 5) pour éviter le split-brain.  
- Formatez les logs des contrôleurs avec `kafka-storage.sh format --cluster-id $(kafka-storage.sh random-uuid)` **avant** de démarrer.  
- Tous les outils CLI doivent utiliser `--bootstrap-server` au lieu de `--zookeeper`.

#### 4.2. Fonctionnement interne de KRaft – le quorum Raft

- Un leader est élu parmi les contrôleurs. Il reçoit les requêtes d’écriture (création de topics, changements d’ISR, ...) et les réplique auprès des **voteurs** (les autres contrôleurs).
- Le journal Raft contient tous les enregistrements de métadonnées. Un **snapshot** est périodiquement pris pour éviter que le journal ne grossisse indéfiniment.
- Les brokers lisent les métadonnées depuis les contrôleurs (pas besoin de se connecter à tous).

📌 **Scénario** :  
Le leader contrôleur tombe. Après `controller.quorum.election.timeout.ms` (défaut 1 s), un autre contrôleur devient leader. Les brokers doivent reconnecter leur heartbeat au nouveau leader. Ce basculement est transparent (mais provoque une brève indisponibilité des métadonnées).

💡 **Astuce pro** :  
- Surveillez `kafka.controller:type=KafkaController` pour l’état des contrôleurs (`ActiveControllerCount` doit être 1).  
- Les logs `raft‑` des contrôleurs sont essentiels pour diagnostiquer des soucis de quorum.  
- KRaft permet de modifier dynamiquement la liste des contrôleurs (ajout/suppression) via `kafka-features`.

#### 4.3. Migration de ZooKeeper vers KRaft – étapes clés

La migration est possible **sans downtime** pour les clusters récents (3.3+).  
1. Mettre à jour tous les brokers vers une version supportant KRaft (3.3 ou 3.4).  
2. Générer un cluster ID et formater les logs des contrôleurs.  
3. Démarrer trois contrôleurs dans un mode **métadonnées version 3.0** (compatible avec ZooKeeper).  
4. Exécuter `kafka-features --bootstrap-server ... upgrade --metadata.version 3.0` pour activer le mode de migration.  
5. Arrêter ZooKeeper, passer les brokers en mode KRaft.  
6. Augmenter la version de métadonnées final.

📌 **Attention** : une migration ratée peut rendre le cluster instable. Testez d’abord dans un environnement de staging.

💡 **Astuce pro** :  
- Vérifiez la compatibilité avec le guide officiel : toutes les versions antérieures à 3.0 ne supportent pas la migration.  
- Conservez des sauvegardes des données ZooKeeper.  
- Après migration, supprimez les dépendances à ZooKeeper pour simplifier l’administration.

#### 4.4. Avantages définitifs de KRaft face à ZooKeeper

| Aspect | ZooKeeper | KRaft |
|--------|-----------|-------|
| Simplicité d’opération | Cluster séparé à maintenir | Plus de ZK, un seul type de processus |
| Scalabilité des métadonnées | ZK limité à quelques milliers de partitions | Supporte des millions de partitions |
| Sécurité | ACL séparées | Uniformisation des ACL Kafka |
| Démarrage à froid | Lenteur du chargement des métadonnées | Récupération plus rapide via snapshots |

💡 **Astuce pro** :  
- Pour les nouveaux clusters, utilisez directement KRaft.  
- Pour les clusters existants, planifiez la migration pour bénéficier de la simplicité et des performances.

---

### 5. Métriques et CLI avancés pour l’architecture

#### 5.1. Métriques JMX critiques – tableau approfondi

| MBean | Description | Action si anormal |
|-------|-------------|-------------------|
| `kafka.cluster:type=Partition` `UnderReplicatedPartitions` | Partitions avec moins de réplicas que prévu | Vérifier les brokers morts ou le réseau |
| `kafka.controller:type=ControllerStats` `ActiveControllerCount` | 1 = ok, 0 ou >1 = problème | Vérifier la configuration de quorum KRaft |
| `kafka.log:type=LogFlushStats` `flush-time` | Temps des fsyncs disque | Disque trop lent (remplacer par SSD) |
| `kafka.network:type=RequestMetrics` `RequestQueueSize` | Taille de la file d’attente des requêtes réseau | > 0 indique une saturation → ajouter des threads réseau (`num.network.threads`) |
| `kafka.server:type=ReplicaFetcherManager` `MaxLag` | Retard max parmi tous les followers | > 10s → vérifier la bande passante inter‑broker |

#### 5.2. Commandes indispensables (en KRaft)

```bash
# Voir la liste des contrôleurs et leur statut
kafka-metadata-quorum --bootstrap-server localhost:9093 describe --status

# Comparer les logs Raft entre contrôleurs
kafka-dump-log --cluster-metadata --files /data/kraft-controller-logs/__cluster_metadata-0/00000000000000000000.log

# Modifier dynamiquement le nombre de partitions d’un topic (uniquement augmentation)
kafka-topics --bootstrap-server localhost:9092 --alter --topic orders --partitions 8

# Vérifier la distribution des partitions par broker
kafka-topics --describe --under-replicated-partitions
```

#### 5.3. Simuler une panne pour tester la résilience

```bash
# Tuer un broker et voir l’élection
kill -9 <pid_broker>
# Observer le nouveau leader avec
kafka-topics --describe --topic test | grep Leader

# Désactiver le réseau pour un follower via iptables
sudo iptables -A INPUT -s <ip_leader> -j DROP

# Rétablir après un moment
sudo iptables -D INPUT -s <ip_leader> -j DROP
```

💡 **Astuce pro** :  
- Montez un cluster de test avec Docker Compose et KRaft pour expérimenter toutes ces pannes.  
- Utilisez `kafka-replica-verification` pour détecter des divergences silencieuses entre réplicas.

---

## Conclusion du chapitre 1 – ce qu’il faut retenir absolument

- Une partition est une séquence ordonnée, immuable, qui ne peut que croître.
- La réplication protège contre la perte de broker, mais le paramètre `min.insync.replicas` détermine le niveau de durabilité.
- L’ISR est dynamique ; surveiller sa taille et ses variations.
- KRaft est l’avenir : migrez quand vous êtes prêt.
- Les métriques JMX et les CLI sont vos meilleurs alliés pour diagnostiquer.

**Prochaine étape** : nous détaillerons de la même manière les producteurs/consommateurs, les streams, connect, et monitoring. Bonne révision ! 🚀
