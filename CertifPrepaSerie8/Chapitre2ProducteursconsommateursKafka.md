# ⚙️ Chapitre 2 : Producteurs et consommateurs Kafka – Idempotence, Exactly‑once, Rebalance, Configurations avancées  
*(Version ultra-détaillée – 5 fois plus de contenu que la précédente)*

---

### 1. Producteurs – principes fondamentaux et tuning

Un **producteur** envoie des messages à un topic. Il décide de la partition (via clé ou partitionneur personnalisé), compresse les lots, gère les accusés de réception (acks) et les retries.

#### 1.1. Cycle de vie d’un message côté producteur

1. **Sérialisation** : la clé et la valeur sont transformées en tableau d’octets (`Serializer`).
2. **Partitionnement** : détermination de la partition cible (clé + partitionner par défaut ou personnalisé).
3. **Accumulation dans un batch** : les messages pour une même partition sont regroupés (buffer mémoire, paramètre `buffer.memory`).
4. **Envoi** : le batch est expédié au leader de la partition.
5. **Attente de l’ACK** : en fonction du paramètre `acks`, le producteur attend la confirmation du broker.
6. **Callback ou Future** : l’appelant est notifié de la réussite ou de l’échec.

📌 **Exemple avec code Java (fragment)** :  
```java
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("acks", "all");
props.put("retries", 10);
props.put("enable.idempotence", "true");

Producer<String, String> producer = new KafkaProducer<>(props);
producer.send(new ProducerRecord<>("orders", "key123", "value456"), (metadata, exception) -> {
    if (exception == null) {
        System.out.println("offset: " + metadata.offset());
    } else {
        exception.printStackTrace();
    }
});
producer.flush();
producer.close();
```

💡 **Astuce pro** :  
- Toujours fermer le producteur avec `close()` pour libérer les ressources.  
- Le callback est asynchrone, ne faites pas de traitements bloquants à l’intérieur.  
- Pour les applications haute performance, utilisez le même producteur (thread‑safe) pour tous les threads.

#### 1.2. Paramètres critiques du producteur – analyse approfondie

| Paramètre | Défaut | Description | Impact |
|-----------|--------|-------------|--------|
| `acks` | 1 | `0` = pas d’accusé, `1` = leader accusé, `all` = tous les ISR | `0` : perte possible, latence minimale. `all` : durabilité max, latence plus élevée |
| `buffer.memory` | 32 MB | Taille totale du tampon pour accumuler les messages | Trop petit → `TimeoutException`. Trop grand → gaspillage mémoire |
| `batch.size` | 16 KB | Taille maximale d’un lot avant envoi | Augmenter (> 64KB) pour les gros débits |
| `linger.ms` | 0 | Temps d’attente pour compléter un lot | Mettre 5-100 ms pour améliorer le batching |
| `max.request.size` | 1 MB | Taille maximale d’une requête de production | Doit être **≤** `message.max.bytes` du broker |
| `retries` | 2147483647 | Nombre de tentatives en cas d’erreur transitoire | Avec idempotence, peut être infini |
| `retry.backoff.ms` | 100 ms | Délai entre deux tentatives | Augmenter (500 ms) pour éviter de surcharger |
| `delivery.timeout.ms` | 120 s | Temps maximal entre envoi et accusé (inclut retries) | Réduire pour détecter plus vite les échecs |
| `max.in.flight.requests.per.connection` | 5 | Nombre de requêtes en parallèle non acquittées | Avec idempotence = peut rester 5 ; sans idempotence = 1 pour éviter réordonnancement |

💡 **Astuce pro** :  
- Pour la **meilleure durabilité** : `acks=all`, `enable.idempotence=true`, `retries=MAX`, `max.in.flight.requests=5`.  
- Pour la **meilleure latence** : `acks=0`, `linger.ms=0`, `batch.size=16384`.  
- Pour la **meilleure compacité** : activez `compression.type=lz4` ou `zstd` ; les batches plus grands (= `linger.ms` > 0) améliorent le taux de compression.

#### 1.3. Gestion des erreurs producteur – cas typiques

| Exception | Cause | Action recommandée |
|-----------|-------|--------------------|
| `TimeoutException` | Délai d’attente dépassé (réseau, broker surchargé) | Augmenter `delivery.timeout.ms`, vérifier la charge broker |
| `RecordTooLargeException` | Message > `max.request.size` | Augmenter la config ou découper le message |
| `TopicAuthorizationException` | Pas de droit d’écriture sur le topic | Vérifier les ACLs |
| `NotEnoughReplicasException` | ISR < `min.insync.replicas` | Attendre que des réplicas reviennent, ou réduire `minISR` |
| `UnknownProducerIdException` | Transaction périmée | Renouveler `transactional.id` |

💡 **Astuce pro** :  
- Implémentez un **retry supervisé** avec backoff exponentiel si vous n’utilisez pas les retries intégrés.  
- Utilisez les **callbacks** pour journaliser les erreurs ; ne pas simplement les ignorer.

---

### 2. Idempotence – la clé pour éviter les doublons

#### 2.1. Fonctionnement technique

L’idempotence repose sur trois mécanismes :
- **Numéro de séquence** (`sequence number`) par partition, incrémenté à chaque message.
- **Producer ID (PID)** : identifiant unique attribué par le broker au premier envoi.
- **Époque** : incrémentée à chaque initialisation, évite les conflits entre anciennes et nouvelles instances.

📌 **Déroulement** :  
1. Le producteur s’enregistre auprès du broker et obtient un PID (et une époque).
2. Pour chaque partition, le producteur envoie des messages avec des séquences 0, 1, 2, ...
3. Le broker garde en mémoire le dernier numéro de séquence reçu par PID+partition.
4. Si le broker reçoit une séquence déjà traitée, il ignore le message en doublon et répond avec succès.

💡 **Astuce pro** :  
- L’idempotence est **indispensable** pour toute application qui retente des envois.  
- Elle fonctionne **par partition** : l’ordre global entre partitions n’est pas garanti.  
- Activez‑la avec `enable.idempotence=true`. Elle impose `acks=all` et `max.in.flight.requests.per.connection ≤ 5`.

#### 2.2. Limitations de l’idempotence seule

- Ne couvre pas les opérations atomiques sur plusieurs topics ou partitions.  
- Un redémarrage du producteur (nouveau PID) peut écraser l’état et provoquer des erreurs de séquence si le broker garde l’ancien PID.

📌 **Exemple d’échec** :  
Producteur A (PID 1) envoie message 0,1,2. Redémarrage : nouveau PID 2, époque=2. Le broker reçoit séquence 0 pour PID 2 → c’est accepté (car nouveau PID). Mais le message 0 a déjà été écrit par PID 1 → potentiel doublon. Pour éviter cela, il faut **les transactions**.

💡 **Astuce pro** :  
- Ne vous fiez pas à l’idempotence seule pour garantir exactly‑once ; utilisez plutôt les transactions.

---

### 3. Exactly‑once semantics (EOS) – Transactions et atomicité

#### 3.1. Concepts : transaction, producteur transactionnel, consommateur `read_committed`

Une transaction regroupe plusieurs écritures (sur un ou plusieurs topics) et éventuellement la mise à jour des offsets des consommateurs.  
- `beginTransaction()` : démarre une transaction.  
- `send()` : envois normaux, mais pas encore visibles.  
- `sendOffsetsToTransaction()` : envoie les offsets des consommateurs (nécessite un groupe de consommateurs).  
- `commitTransaction()` : tous les messages deviennent visibles **atomiquement**.  
- `abortTransaction()` : aucun message n’est visible.

Le consommateur doit être configuré avec `isolation.level=read_committed` pour ne voir que les messages des transactions commitées.

📌 **Exemple (en Java)** :  
```java
producer.initTransactions();
producer.beginTransaction();
producer.send(new ProducerRecord<>("topic1", "key", "value1"));
producer.send(new ProducerRecord<>("topic2", "key", "value2"));
producer.sendOffsetsToTransaction(getCurrentOffsets(), "group1");
producer.commitTransaction();
```

💡 **Astuce pro** :  
- Le `transactional.id` doit être **unique** et persistant (pas de changement au redémarrage).  
- Une transaction expirée (dépassement de `transaction.timeout.ms`, défaut 1 min) est automatiquement avortée.  
- En cas de crash du producteur, le broker attend la fin du timeout puis avorte.

#### 3.2. Exactly‑once dans Kafka Streams – `processing.guarantee`

Kafka Streams intègre EOS en interne sans avoir à gérer les transactions manuellement.  
- `processing.guarantee=at_least_once` (défaut) : peut générer des doublons après un recalcul.  
- `processing.guarantee=exactly_once` (v1) : utilise les transactions, mais avec une latence plus élevée et des limitations de débit.  
- `processing.guarantee=exactly_once_v2` (recommandé) : version optimisée, moins de verrous, meilleur débit.

📌 **Configuration Streams EOS** :  
```java
Properties props = new Properties();
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, "exactly_once_v2");
props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 100); // par défaut 30s, meilleure granularité
```

💡 **Astuce pro** :  
- `exactly_once_v2` nécessite Kafka 2.5+.  
- Dans un environnement Kubernetes, assurez-vous que le `transactional.id` est unique par pod ; utilisez des noms statiques (ex: `my-app-$HOSTNAME`).

#### 3.3. Coût des transactions – dégradation de performance

Les transactions ajoutent une latence et réduisent le débit maximal :
- Chaque transaction ajoute au moins deux messages de contrôle (marqueur de début/fin) dans le log et dans le topic `__transaction_state`.
- Avec de petites transactions (1 message), le débit peut chuter de 50 %.  
- Avec des transactions groupant des milliers de messages, la perte est de ~10-15 %.

📌 **Recommandation** :  
Pour des flux très haute fréquence (ex: logs), acceptez `at_least_once` et dédoublonnez côté application. Pour des flux financiers, utilisez des transactions avec des lots de taille raisonnable (quelques centaines de messages).

💡 **Astuce pro** :  
- Surveillez la métrique `kafka.producer:type=producer-metrics,client-id=*` `transaction‑duration` pour détecter des transactions trop lentes.  
- Le topic `__transaction_state` doit avoir un facteur de réplication élevé (≥3) et une politique de clean-up `compact`.

---

### 4. Rebalance des consommateurs – mécanique et bonnes pratiques

#### 4.1. Cycle d’un rebalance

1. Un consommateur rejoint ou quitte le groupe, ou le pattern de topics change, ou des partitions sont ajoutées.
2. Le **coordinateur de groupe** (un broker désigné) détecte le changement (via heartbeat ou via `JoinGroup`).
3. Une génération est incrémentée. Tous les consommateurs reçoivent une `RebalanceInProgressException` et doivent arrêter de consommer.
4. Les consommateurs envoient leur `JoinGroup` avec leurs métadonnées.
5. Le **leader** du groupe calcule la nouvelle affectation des partitions et la distribue via `SyncGroup`.
6. Chaque consommateur reçoit sa nouvelle liste de partitions et appelle `onPartitionsAssigned`.
7. La consommation reprend.

📌 **Chronologie typique** :  
- `session.timeout.ms` (45s) : délai avant de considérer un consommateur mort si plus de heartbeat.  
- `heartbeat.interval.ms` (3s) : fréquence des heartbeats.  
- `max.poll.interval.ms` (5min) : délai maximum entre deux `poll()`. Si dépassé, le consommateur est considéré mort → rebalance.

💡 **Astuce pro** :  
- Pour éviter les rebalances intempestives, **ne bloquez jamais** dans la boucle de `poll()` plus longtemps que `max.poll.interval.ms`.  
- Réduisez `max.poll.records` pour traiter moins de records à la fois.

#### 4.2. Stratégies d’affectation (`partition.assignment.strategy`)

| Stratégie | Comportement | Avantage | Inconvénient |
|-----------|--------------|----------|--------------|
| `RangeAssignor` (défaut) | Affecte les partitions par topic : les consommateurs prennent des blocs contigus | Simple | Peut être déséquilibrée |
| `RoundRobinAssignor` | Distribution en tourniquet sur toutes les partitions de tous les topics | Équilibrage optimal | Peut provoquer des réaffectations complètes |
| `StickyAssignor` | Minimise les mouvements de partition lors des rebalances | Stable | Complexe |
| `CooperativeStickyAssignor` | Réaffecte uniquement les partitions nécessaires, en plusieurs « rounds » | Réduit le temps d’indisponibilité | Nécessite Kafka 2.4+ |

📌 **Exemple : Range vs RoundRobin**  
3 consommateurs, 2 topics chacun avec 3 partitions.  
- Range : C1 = t1p0, t2p0 ; C2 = t1p1, t2p1 ; C3 = t1p2, t2p2 → équilibré. Mais si un topic a 4 partitions et l’autre 2, le déséquilibre apparaît.  
- RoundRobin : les partitions sont triées par topic puis par index, puis distribuées cycliquement.

💡 **Astuce pro** :  
- Utilisez **CooperativeStickyAssignor** en priorité (Kafka ≥ 2.4).  
- Pour des groupes avec des souscriptions variables, `RoundRobinAssignor` est plus équitable.

#### 4.3. Callbacks de rebalance – `ConsumerRebalanceListener`

Pour préserver l’état ou les derniers offsets, il faut implémenter :
- `onPartitionsRevoked(Collection<TopicPartition>)` : appelé avant que les partitions ne soient perdues. Idéal pour committer les offsets finaux ou sauvegarder un état.
- `onPartitionsAssigned(Collection<TopicPartition>)` : appelé après réaffectation. Pour réinitialiser un état local.

📌 **Exemple d’implémentation sécurisée** :  
```java
consumer.subscribe(Collections.singletonList("topic"), new ConsumerRebalanceListener() {
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        consumer.commitSync(); // commit avant de perdre la partition
    }
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // restaurer un état local à partir du state store
    }
});
```

💡 **Astuce pro** :  
- **Ne faites pas** de longs traitements dans `onPartitionsRevoked`, car ils retardent le rebalance.  
- Utilisez `commitSync` (bloquant) plutôt qu’asynchrone pour garantir la persistance.

#### 4.4. Détection des rebalances problématiques (capture)

- **Rebalance trop fréquentes** : souvent dû à des `poll()` qui prennent trop de temps. Augmentez `max.poll.interval.ms` ou réduisez `max.poll.records`.
- **Membres fantômes** : un consommateur qui ne répond pas heartbeat mais reste en vie → réglez `session.timeout.ms` à une valeur plus basse (10 s) pour détecter plus vite.
- **Coordination lente** : si le groupe est très grand, le leader peut mettre du temps à calculer l’affectation. Utilisez `CooperativeStickyAssignor` pour réduire la charge.

💡 **Astuce pro** :  
- Utilisez l’outil `kafka-consumer-groups --describe --group <group>` avec l’option `--members` pour voir les membres actifs.  
- Les métriques JMX `kafka.consumer:type=consumer-coordinator-metrics,name=rebalance-latency-avg` donnent le temps moyen des rebalances.

---

### 5. Gestion des offsets – commit, reset, et rejeu

#### 5.1. Commit automatique vs manuel

- **Auto‑commit** (`enable.auto.commit=true`, `auto.commit.interval.ms=5000`): les offsets sont commités périodiquement (toutes les 5 secondes). Risque : perte d’offsets entre deux commits.  
- **Commit manuel synchrone** : `consumer.commitSync()` – bloque jusqu’à confirmation, garantit la persistance.  
- **Commit manuel asynchrone** : `consumer.commitAsync()` – non bloquant, peut échouer sans retry. Combine souvent avec un commit final synchrone sur fermeture.

📌 **Bonnes pratiques** :  
- Pour `at-least-once`, traitez les records, puis `commitSync()`.  
- Pour `exactly-once`, utilisez `commitTransaction()` ou les transactions du producteur.  
- Dans les applications Kafka Streams, le commit est géré automatiquement par le paramètre `commit.interval.ms`.

💡 **Astuce pro** :  
- Ne jamais appeler `commitSync` dans un callback (sauf rebalance) sans gestion des erreurs.  
- Utilisez `commitAsync` avec un callback pour journaliser les échecs.

#### 5.2. Reset des offsets – `auto.offset.reset`

Définit le comportement quand aucun offset n’a encore été commité (premier démarrage) ou quand l’offset demandé n’existe plus (supprimé par rétention).

- `earliest` : repartir du premier message disponible.  
- `latest` : commencer depuis la fin (ne lire que les nouveaux messages).  
- `none` : lever une exception si aucun offset n’existe.

📌 **Exemple d’utilisation** :  
- Rejeu complet : `auto.offset.reset=earliest`, `seekToBeginning(topicPartition)`.  
- Pipeline temps réel : `auto.offset.reset=latest`.  
- Application qui ne doit jamais manquer un message : gérer l’exception en réinitialisant manuellement avec `seek`.

💡 **Astuce pro** :  
- Pour rejouer depuis un timestamp précis, utilisez `offsetsForTimes(Map<TopicPartition, Long>)`, puis `seek()`.  
- Après un reset, assurez‑vous que vos producers idempotents ou transactions ne recréent pas de doublons.

---

### 6. Monitoring et tuning avancé des producteurs/consommateurs

#### 6.1. Métriques JMX essentielles

| MBean | Description | Seuil d’alerte |
|-------|-------------|----------------|
| `kafka.producer:type=producer-metrics` `request-latency-avg` | Latence moyenne des requêtes | > 100 ms |
| `kafka.producer:type=producer-metrics` `record-error-rate` | Taux d’erreurs (par seconde) | > 0 |
| `kafka.producer:type=producer-metrics` `buffer-available-bytes` | Mémoire tampon disponible | < 10% de la taille totale |
| `kafka.consumer:type=consumer-fetch-manager-metrics` `records-lag-max` | Lag max du consommateur | > 10 000 |
| `kafka.consumer:type=consumer-coordinator-metrics` `assigned-partitions` | Nombre de partitions assignées | Variation inattendue (rebalance) |

💡 **Astuce pro** :  
- Exposez via Prometheus JMX Exporter.  
- Utilisez Grafana pour visualiser `records-lag` par partition.

#### 6.2. Commandes CLI pour consommateurs

```bash
# Voir le lag actuel
kafka-consumer-groups --bootstrap-server localhost:9092 --group monGroupe --describe

# Réinitialiser les offsets au début (nécessite l’option --execute)
kafka-consumer-groups --bootstrap-server localhost:9092 --group monGroupe --reset-offsets --to-earliest --topic monTopic --execute

# Lister les groupes
kafka-consumer-groups --bootstrap-server localhost:9092 --list

# Supprimer un groupe (tous les offsets sont effacés)
kafka-consumer-groups --bootstrap-server localhost:9092 --delete --group monGroupe
```

💡 **Astuce pro** :  
- En KRaft, remplacez toujours `--zookeeper` par `--bootstrap-server`.  
- Pour supprimer les offsets d’un topic spécifique, utilisez `--reset-offsets` avec `--to-earliest` puis `--execute`.

#### 6.3. Tests de charge et performances

```bash
# Producteur de test (1 M messages, 100 bytes chacun, acks=1)
kafka-producer-perf-test --topic perf --num-records 1000000 --record-size 100 --throughput -1 --producer-props bootstrap.servers=localhost:9092 acks=1

# Consommateur de test
kafka-consumer-perf-test --topic perf --messages 1000000 --broker-list localhost:9092 --timeout 60000
```

💡 **Astuce pro** :  
- Surveillez GC et threads avec `jstat` et `top -H` en parallèle.  
- Comparez `acks=1` vs `acks=all` pour évaluer le coût de durabilité.

---

## Conclusion du chapitre 2 – points critiques à maîtriser

- **Idempotence** : élimine les doublons, indispensable pour exactly‑once.  
- **Transactions** : atomicité multi‑topics et exactement une fois.  
- **Rebalance** : utilisez `CooperativeStickyAssignor` et gérez les callbacks proprement.  
- **Monitoring** : lag, latence, taux d’erreur sont vos radios.  
- **Reset offsets** : contrôlez précisément la reprise avec `kafka-consumer-groups` et `seek`.

**Prochain chapitre** : Kafka Streams (KStream, KTable, state stores, windowing).
