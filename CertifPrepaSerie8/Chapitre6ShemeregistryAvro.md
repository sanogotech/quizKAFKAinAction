# 🧩 Chapitre 6 : Schema Registry – Sérialisation Avro/Protobuf, compatibilité et évolution  
*(Version ultra-détaillée – 5 fois plus de contenu que la précédente)*

---

### 1. Pourquoi Schema Registry ?

Kafka ne stocke que des octets (byte arrays). Sans schéma, les producteurs et consommateurs doivent se mettre d’accord **manuellement** sur la structure des messages. Cela pose des problèmes :

- **Fragilité** : tout changement dans un producteur peut casser les consommateurs.
- **Redondance** : chaque message transporte l’intégralité de son schéma (en JSON, Avro, etc.) → surcharge réseau.
- **Impossibilité d’évoluer** : sans versioning, on ne peut pas ajouter un champ optionnel sans risque.

**Schema Registry** (SR) résout ces problèmes :
- Stocke **centralement** les schémas associés à un sujet (topic + clé ou valeur).
- Attribue un **identifiant unique** à chaque version de schéma.
- Les messages ne contiennent que cet ID (4 octets) → économie de bande passante.
- Vérifie la **compatibilité** des nouvelles versions avec les anciennes.

📌 **Exemple** :  
Message normal (JSON) : `{"id": 1, "name": "Alice"}`.  
Avec Avro + SR : le producteur envoie `[ID, sérialisation binaire]`. Le consommateur récupère le schéma via l’ID et désérialise.

💡 **Astuce pro** :  
Le SR n’est pas obligatoire, mais il est vivement recommandé pour tout système en production qui évolue.

---

### 2. Formats supportés et sérialisation

#### 2.1. Apache Avro

- Format **binaire** compact, avec schéma défini en JSON.
- Supporte l’évolution (champs optionnels, défauts, alias).
- Schéma obligatoire pour sérialiser/désérialiser.

📌 **Schéma Avro (utilisateur.avsc)** :  
```json
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "int"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": ["null", "string"], "default": null}
  ]
}
```

💡 **Astuce pro** :  
Utilisez `avro-maven-plugin` ou `avro-gradle-plugin` pour générer les classes Java automatiquement.

#### 2.2. Protobuf (Protocol Buffers)

- Format binaire, plus compact qu’Avro (parfois).
- Définition dans un fichier `.proto`.
- Supporte l’évolution (champs optionnels, réservés).

📌 **Fichier user.proto** :  
```proto
message User {
  int32 id = 1;
  string name = 2;
  optional string email = 3;
}
```

💡 **Astuce pro** :  
Protobuf est plus rapide en désérialisation et plus utilisé dans les environnements gRPC.

#### 2.3. JSON Schema

- Format texte (plus lourd mais lisible).
- Utile pour les pipelines où les consommateurs attendent du JSON (ex: API REST, Elasticsearch).
- Support de l’évolution (propriétés additionnelles, `required`).

📌 **Schéma JSON** :  
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "id": {"type": "integer"},
    "name": {"type": "string"},
    "email": {"type": "string"}
  },
  "required": ["id", "name"]
}
```

💡 **Astuce pro** :  
Pour des raisons de performance, **Avro** et **Protobuf** sont préférés aux JSON Schema dans le cluster.

---

### 3. Modes de compatibilité

Le SR vérifie qu’un nouveau schéma est compatible avec les versions existantes selon un **mode configurable** par sujet.

| Mode | Signification |
|------|----------------|
| `BACKWARD` | Les consommateurs utilisant l’ancien schéma peuvent lire les messages produits avec le nouveau schéma. (Le nouveau schéma ne doit supprimer que des champs ayant une valeur par défaut). |
| `FORWARD` | Les consommateurs utilisant le nouveau schéma peuvent lire les messages produits avec l’ancien schéma. (Le nouveau schéma ne doit ajouter que des champs optionnels). |
| `FULL` | Les deux conditions précédentes réunies. Ideal pour une évolution sans temps d’arrêt. |
| `NONE` | Compatibilité désactivée (tout changement est autorisé, mais peut casser les clients). |
| `BACKWARD_TRANSITIVE` | Comme BACKWARD, mais vérifie aussi toutes les versions antérieures, pas seulement la précédente. |
| `FORWARD_TRANSITIVE` | Idem, mais pour FORWARD. |
| `FULL_TRANSITIVE` | Idem, mais pour FULL. |

📌 **Exemple** :  
- **Backward** : ajouter un champ avec `default` → un ancien consommateur peut toujours lire (le nouveau champ est ignoré).  
- **Forward** : supprimer un champ (qui avait une valeur par défaut) → un nouveau consommateur peut toujours lire les anciens messages (le champ manque, on utilise la défaut).

💡 **Astuce pro** :  
- **FULL_TRANSITIVE** est le plus sûr, mais plus strict.  
- Pour les sujets en production, commencez avec `BACKWARD` ou `FULL` selon votre stratégie d’évolution.

---

### 4. Configuration du Schema Registry

#### 4.1. Démarrage du serveur SR (Confluent)

```bash
# docker-compose typique
schema-registry:
  image: confluentinc/cp-schema-registry:7.5.0
  environment:
    SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: PLAINTEXT://kafka:9092
    SCHEMA_REGISTRY_HOST_NAME: schema-registry
    SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
```

💡 **Astuce pro** :  
Le SR doit avoir le même `group.id` que les brokers (si plusieurs instances).

#### 4.2. Client Producer (Avro)

```java
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(AbstractKafkaAvroSerDeConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://localhost:8081");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class);

KafkaProducer<String, User> producer = new KafkaProducer<>(props);
User user = User.newBuilder().setId(1).setName("Alice").build();
producer.send(new ProducerRecord<>("users", "key", user));
```

💡 **Astuce pro** :  
Si vous utilisez Confluent Platform, les sérialiseurs `KafkaAvroSerializer` sont dans `io.confluent.kafka.serializers`.

#### 4.3. Client Consumer

```java
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class);
props.put(AbstractKafkaAvroSerDeConfig.SPECIFIC_AVRO_READER_CONFIG, true); // pour les classes générées
KafkaConsumer<String, User> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("users"));
```

💡 **Astuce pro** :  
- `SPECIFIC_AVRO_READER_CONFIG=true` utilise la classe générée (plus rapide).  
- Si `false`, renvoie `GenericRecord`.

#### 4.4. Autres configurations importantes

| Paramètre | Effet |
|-----------|-------|
| `auto.register.schemas` (défaut true) | Inscrit automatiquement le schéma dans le SR s’il est nouveau. |
| `use.latest.version` | Force l’utilisation de la dernière version du schéma (utile pour les consommateurs forwards). |
| `subject.name.strategy` | Stratégie de nommage des sujets. Par défaut : `TopicNameStrategy` → topic + `-key` / `-value`. |

---

### 5. Stratégies de nommage des sujets (`SubjectNameStrategy`)

Le **sujet** dans SR est l’entité qui regroupe les versions d’un schéma. Par défaut : `<topic>-key` et `<topic>-value`. Mais on peut changer :

- `TopicNameStrategy` : par topic (défaut).
- `RecordNameStrategy` : par nom d’enregistrement (ex: `com.example.User`).
- `TopicRecordNameStrategy` : par topic + nom d’enregistrement (permet plusieurs types par topic).

📌 **Exemple** :  
Un topic `events` peut contenir des `UserCreated` et `UserDeleted`. Avec `TopicRecordNameStrategy`, les sujets SR seront `events-com.example.UserCreated` et `events-com.example.UserDeleted`.

💡 **Astuce pro** :  
Utilisez `TopicRecordNameStrategy` quand un topic véhicule des types différents.

---

### 6. API REST du Schema Registry

Le SR expose une API REST (port 8081) pour interagir manuellement ou par script.

```bash
# Lister les sujets
curl -X GET http://localhost:8081/subjects

# Récupérer la dernière version d’un schéma
curl -X GET http://localhost:8081/subjects/users-value/versions/latest

# Ajouter un nouveau schéma (auto‑register)
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",...}"}' \
  http://localhost:8081/subjects/users-value/versions

# Vérifier la compatibilité d’un nouveau schéma
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "..."}' \
  http://localhost:8081/compatibility/subjects/users-value/versions/latest

# Modifier le mode de compatibilité d’un sujet
curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "FULL"}' \
  http://localhost:8081/config/users-value
```

💡 **Astuce pro** :  
Utilisez `jq` pour formater la réponse JSON. Par ex : `curl ... | jq '.'`.

---

### 7. Évolution des schémas – règles et pièges

#### 7.1. Ajouter un champ optionnel (sous Avro)

Avro : champ avec `"default": null` (type union `["null", "string"]` par exemple).  
✅ Compatibilité **BACKWARD** (ancien consommateur peut lire, ignore le champ).  
✅ Compatibilité **FORWARD** (nouveau consommateur peut lire les anciens messages, car le champ est optionnel).

#### 7.2. Supprimer un champ (avec valeur par défaut)

Seulement FORWARD – les nouveaux consommateurs peuvent utiliser la valeur par défaut.

#### 7.3. Renommer un champ

**Non compatible** directement. Solution : ajouter un alias (`aliases`) dans le nouveau schéma, et le consommateur peut utiliser l’ancien nom.

```json
{"name": "fullName", "aliases": ["name"], "type": "string"}
```

#### 7.4. Changer le type d’un champ

Généralement incompatible, sauf transformation avancée (ex: `int` → `long` est compatible pour Avro ? non, c’est cassant). Utilisez plutôt un nouveau champ et dépréciez l’ancien.

💡 **Astuce pro** :  
- **Ne jamais supprimer un champ sans valeur par défaut** – cassant.  
- **Ne pas réutiliser un ancien nom** pour un autre usage.  
- **Tout changement** doit être testé avec l’outil de compatibilité avant déploiement.

---

### 8. Gestion des clés et valeurs séparées

Les clés et les valeurs d’un message peuvent avoir des schémas **différents** (et souvent indépendants). Le SR utilise :

- Sujet pour la clé : `<topic>-key`
- Sujet pour la valeur : `<topic>-value`

Cela permet de faire évoluer la clé (ex: passer de `int` à `string`) sans impacter la valeur, et vice‑versa.

📌 **Exemple** :  
Topic `orders` → clé : `int` (order_id), valeur : `Order` (Avro).  
On peut changer la valeur d’un champ sans modifier la clé.

💡 **Astuce pro** :  
Dans vos applications, utilisez un schéma explicite pour la clé (même si c’est un simple `int` ou `string`) pour faciliter l’évolution future.

---

### 9. Intégration avec Kafka Connect

Les connecteurs peuvent utiliser le SR pour sérialiser/désérialiser les données. Dans la configuration du connecteur :

```json
"key.converter": "io.confluent.connect.avro.AvroConverter",
"key.converter.schema.registry.url": "http://localhost:8081",
"value.converter": "io.confluent.connect.avro.AvroConverter",
"value.converter.schema.registry.url": "http://localhost:8081"
```

Pour les sources, un nouveau schéma est automatiquement enregistré (si `auto.register` true).  
Pour les sinks, il récupère le schéma du topic pour désérialiser.

💡 **Astuce pro** :  
Pour les connecteurs sink JDBC, assurez-vous que les champs correspondent exactement au schéma cible (via SMT `ReplaceField`).

---

### 10. Monitoring et métriques du Schema Registry

| Métrique (JMX) | Description | Seuil |
|----------------|-------------|-------|
| `io.confluent.kafka.schemaregistry:type=SchemaRegistry` `schema-registry-request-rate` | Taux de requêtes | Alerte si très élevé (attaque, cache mal configuré) |
| `io.confluent.kafka.schemaregistry:type=SchemaRegistry` `schema-registry-error-rate` | Taux d’erreurs (compatibilité, I/O) | > 0 |
| `kafka.consumer:type=schema-registry-consumer` `lag` | Retard de lecture de l’interne | Suivre comme un consommateur normal |

💡 **Astuce pro** :  
- Augmentez le cache client (`schema.registry.cache.size`) pour réduire les appels SR.  
- Mettez en cache les schémas dans votre application avec une `ConcurrentHashMap`.

---

### 11. Bonnes pratiques résumées

- **Toujours versionner vos schémas** dans un dépôt git (avec le code).  
- **Utilisez la compatibilité `FULL_TRANSITIVE`** pour les sujets critiques.  
- **Ne changez pas le type d’un champ** sans créer un nouveau champ.  
- **Définissez des valeurs par défaut** pour tous les champs additionnels dès le départ.  
- **Évitez les champs obligatoires** qui apparaissent tard (ils cassent la backward compatibility).  
- **Testez les évolutions avec l’API de compatibilité** dans votre CI.  
- **Activez le mode `read_committed`** si vous utilisez les transactions et le SR.  

📌 **Exemple de test de compatibilité dans un pipeline CI** :  
```bash
#!/bin/bash
# Télécharge l’ancien schéma, compare avec le nouveau
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data @new-schema.json \
  http://schema-registry:8081/compatibility/subjects/my-topic-value/versions/latest
# Si la réponse contient "is_compatible": true, ok.
```

---

## Conclusion du chapitre 6 – points clés

- **Schema Registry** centralise les schémas, réduit la taille des messages et permet l’évolution.
- **Avro, Protobuf, JSON Schema** : choisissez selon vos outils (Avro est le plus courant avec Kafka).
- **Compatibilité** : BACKWARD, FORWARD, FULL – définit ce qui est permis.
- **Sujets** : par défaut `<topic>-key` et `<topic>-value`.
- **API REST** : pour administrer et tester.
- **Métriques** : surveillez les erreurs de compatibilité et la charge.

Ce chapitre comble le vide entre le monitoring (chapitre 5) et les tests (chapitre 7). Vous avez désormais une couverture complète pour la certification. 🚀
