## 🔒 Chapitre 8 : Sécurité Kafka – Authentification, Autorisation, Chiffrement, Gestion des secrets  
*(Version ultra-détaillée – 5 fois plus de contenu que la précédente)*

---

### 1. Pourquoi la sécurité dans Kafka ?

Par défaut, Kafka ne dispose d’aucune authentification ni chiffrement. Toute personne pouvant se connecter à un broker peut envoyer/lire n’importe quel message. En production, il est impératif de protéger :

- **Confidentialité** : les données ne doivent pas être lues en clair (chiffrement TLS).
- **Intégrité** : les messages ne doivent pas être altérés en transit (signature TLS).
- **Authentification** : seuls les clients légitimes peuvent produire/consommer.
- **Autorisation** : les clients n’ont que les droits nécessaires (principe du moindre privilège).

📌 **Menaces typiques** :  
- Écoute du trafic réseau (sniffing) → **solution** : TLS.  
- Usurpation d’identité d’un producteur → **solution** : authentification SASL ou TLS client.  
- Client malveillant qui supprime un topic → **solution** : ACL restreintes.  
- Non‑répudiation des accès → **solution** : audit logs.

💡 **Astuce pro** :  
Activez la sécurité **dès le premier environnement de préproduction**. Ajouter la sécurité en production ensuite est très complexe (changement de certificats, migrations, redémarrages).

---

### 2. Authentification – Les mécanismes

#### 2.1. SSL/TLS (« mTLS »)

- **Principe** : chaque client possède un **certificat X.509** signé par une autorité de certification (CA).  
- Le broker et le client s’authentifient mutuellement (mutual TLS).  
- Chiffre le canal en même temps.

📌 **Avantages** : sécurité forte, chiffrement intégré.  
📌 **Inconvénients** : gestion complexe des certificats (renouvellement, révocation).

**Configuration broker** (`server.properties`) :
```properties
listeners=SASL_SSL://0.0.0.0:9093
advertised.listeners=SASL_SSL://broker1.example.com:9093
ssl.keystore.location=/var/private/server.keystore.jks
ssl.keystore.password=changeit
ssl.key.password=changeit
ssl.truststore.location=/var/private/server.truststore.jks
ssl.truststore.password=changeit
ssl.client.auth=required   # mTLS : exige certificat client
```

**Client** (producteur Java) :
```java
props.put("security.protocol", "SSL");
props.put("ssl.truststore.location", "/path/client.truststore.jks");
props.put("ssl.keystore.location", "/path/client.keystore.jks");
props.put("ssl.keystore.password", "changeit");
```

💡 **Astuce pro** :  
- Utilisez des certificats wildcard avec des noms DNS alternatifs (SAN) pour les brokers.  
- Le format PKCS12 est plus moderne que JKS (`ssl.keystore.type=PKCS12`).  
- Pour montée en charge, utilisez une CA interne (ex: `openssl`, `cfssl`) plutôt que des certificats auto‑signés.

#### 2.2. SASL/PLAIN (simple, *insécurisé seul*)

- Le mot de passe est envoyé en clair.  
- **Ne jamais** utiliser sans TLS (chiffrement).

**Configuration broker** :
```properties
listeners=SASL_SSL://0.0.0.0:9093
sasl.enabled.mechanisms=PLAIN
sasl.server.callback.handler.class=org.apache.kafka.common.security.plain.PlainLoginModule
```

**Fichier `jaas.conf`** :
```
KafkaServer {
   org.apache.kafka.common.security.plain.PlainLoginModule required
   user_admin="admin-secret"
   user_producer="prod-secret";
};
```

💡 **Astuce pro** :  
- Idéal pour des environnements de développement sécurisés (avec TLS).  
- En production, préférez SCRAM ou Kerberos.

#### 2.3. SASL/SCRAM (remplace PLAIN)

- **Challenge‑response** : le mot de passe n’est jamais transmis en clair.  
- Les identifiants sont stockés dans ZooKeeper ou KRaft (encryptés).  
- Supporte SHA‑256 et SHA‑512.

**Créer un utilisateur SCRAM** :
```bash
kafka-configs --bootstrap-server localhost:9092 --alter --add-config 'SCRAM-SHA-256=[password=admin-secret]' --entity-type users --entity-name admin
```

**Configuration broker** :
```properties
sasl.enabled.mechanisms=SCRAM-SHA-256
```

**Client** :
```java
props.put("sasl.mechanism", "SCRAM-SHA-256");
props.put("sasl.jaas.config", "org.apache.kafka.common.security.scram.ScramLoginModule required username=\"admin\" password=\"admin-secret\";");
```

💡 **Astuce pro** :  
- SCRAM est plus simple à administrer que Kerberos et plus sécurisé que PLAIN.  
- Pensez à faire tourner les mots de passe régulièrement.

#### 2.4. SASL/Kerberos (environnements déjà Kerberisés)

- Recommandé pour les grandes entreprises ayant déjà Active Directory ou MIT Kerberos.  
- Utilise des tickets (TGT) pour l’authentification sans mot de passe en clair.

**Configuration broker** :
```properties
sasl.enabled.mechanisms=GSSAPI
sasl.kerberos.service.name=kafka
```

**`jaas.conf`** (broker) :
```
KafkaServer {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/etc/security/keytabs/kafka.keytab"
   principal="kafka/broker1@EXAMPLE.COM"
   storeKey=true;
};
```

💡 **Astuce pro** :  
- La synchronisation des temps (NTP) est critique pour Kerberos.  
- Utilisez des `keytab` par broker et par client.

#### 2.5. SASL/OAUTHBEARER (token JWT)

- Idéal pour les environnements cloud et microservices.  
- Client présente un token JWT (ex: issu de Keycloak, Okta).  
- Broker valide le token (signature, expiration, audience) sans stockage d’utilisateur.

**Configuration broker** :
```properties
sasl.enabled.mechanisms=OAUTHBEARER
listener.name.sasl_ssl.oauthbearer.sasl.server.callback.handler.class=org.apache.kafka.common.security.oauthbearer.OAuthBearerValidatorCallbackHandler
```

💡 **Astuce pro** :  
- Implémentez `OAuthBearerValidatorCallbackHandler` pour intégrer votre fournisseur OAuth2.  
- Confluent propose `io.confluent.kafka.security.oauthbearer.OAuthBearerLoginModule`.

---

### 3. Autorisation – ACL (Access Control Lists)

Une fois l’utilisateur authentifié, Kafka contrôle ses droits via des **ACL** (Access Control Lists) basées sur le **principal** (nom d’utilisateur).

#### 3.1. Structure d’une ACL

`Principal` `Opération` sur `Ressource` (`ResourceType`).

- **Ressources** : Topic, ConsumerGroup, Cluster, TransactionalId, DelegationToken.  
- **Opérations** : Read, Write, Create, Delete, Alter, Describe, All.  
- **Principaux** : l’utilisateur (ex: `User:producer`) ou `User:*` (tout le monde).

📌 **Exemple d’ajout d’ACL** (donner le droit d’écrire sur un topic) :
```bash
kafka-acls --bootstrap-server localhost:9092 --add --allow-principal User:producer --operation Write --topic orders
```

#### 3.2. ACL courantes

| Besoin | Commande |
|--------|----------|
| Permettre au producteur `p1` d’écrire sur `topicA` | `--add --allow-principal User:p1 --operation Write --topic topicA` |
| Permettre au consommateur `c1` de lire `topicA` | `--add --allow-principal User:c1 --operation Read --topic topicA` |
| Permettre à `c1` d’utiliser le groupe `g1` | `--add --allow-principal User:c1 --operation Read --group g1` |
| Interdire explicitement à `bad` d’écrire | `--add --deny-principal User:bad --operation Write --topic topicA` |

💡 **Astuce pro** :  
- Par défaut, si aucune ACL ne correspond, la requête est **refusée** (`allow.everyone.if.no.acl.found=false`). C’est le comportement souhaité.  
- Pour un cluster existant sans ACL, on peut basculer progressivement en surveillant les logs.

#### 3.3. Super utilisateurs (`super.users`)

Liste des principaux qui ignorent totalement les ACL (à utiliser avec parcimonie).

```properties
super.users=User:admin;User:kafka
```

💡 **Astuce pro** :  
- Utilisez un compte `admin` pour l’automatisation (scripts, connecteurs) plutôt qu’un super user.  
- Limitez le nombre de super users.

#### 3.4. Gestion des ACL avec API REST / AdminClient

```java
AdminClient admin = AdminClient.create(config);
AclBinding binding = new AclBinding(
    new ResourcePattern(ResourceType.TOPIC, "orders", PatternType.LITERAL),
    new AccessControlEntry("User:producer", "*", AclOperation.WRITE, AclPermissionType.ALLOW)
);
admin.createAcls(Collections.singletonList(binding)).all().get();
```

💡 **Astuce pro** :  
- Versionnez vos ACL (dans git) et appliquez‑les par script à chaque déploiement.  
- Utilisez `kafka-acls --list` pour auditer.

---

### 4. Chiffrement des données au repos (at‑rest)

Kafka n’offre pas de chiffrement intégré des logs sur disque. Pour protéger les messages stockés :

- **Chiffrement au niveau du filesystem** (LUKS, dm‑crypt) : transparent, mais partagé si volumes multiples.
- **Chiffrement applicatif** : l’application chiffre la valeur avant de l’envoyer, seuls les consommateurs autorisés déchiffrent (ex: AES‑GCM, clé gérée par Vault).  
- **KIP‑496** (chiffrement côté broker) : pas encore standard.

📌 **Exemple de chiffrement applicatif avec Avro** :  
Champ `data` est un `bytes` chiffré côté producteur, déchiffré côté consommateur (avec la même clé).

💡 **Astuce pro** :  
- Le chiffrement disque protège contre le vol physique, mais pas contre un administrateur de base.  
- Pour les données sensibles (RGPD, PCI), appliquez un chiffrement applicatif.

---

### 5. Gestion des secrets (mots de passe, clés)

Ne jamais stocker les secrets en clair dans les fichiers de configuration. Utilisez :

#### 5.1. Variables d’environnement

```properties
ssl.keystore.password=${KEYSTORE_PASSWORD}
```

#### 5.2. Fichiers de propriétés externes (sécurisés)

```bash
kafka-configs --bootstrap-server localhost:9092 --alter --add-config 'ssl.keystore.password=xxxx' --entity-type brokers --entity-name 0
```

#### 5.3. Intégration avec HashiCorp Vault ou K8s Secrets

- Injecter les secrets via sidecar container ou CSI driver (ex: `secrets-store-csi-driver`).  
- Configurer le broker avec `ssl.keystore.location=/mnt/secrets/keystore.p12`.

#### 5.4. Confluent Vault Secret Provider

```json
"config.providers": "vault",
"config.providers.vault.class": "io.confluent.common.config.providers.VaultConfigProvider"
```

💡 **Astuce pro** :  
- Utilisez le chiffrement des secrets dans ZooKeeper (ou KRaft) avec `password.encoder.secret`.  
- Faites tourner les secrets périodiquement (rotation) sans downtime (support de plusieurs versions).

---

### 6. Chiffrement entre brokers (intra‑cluster)

Les communications entre brokers (réplication, contrôleur) doivent aussi être sécurisées.

**Configuration** (pour tous les brokers) :
```properties
security.inter.broker.protocol=SASL_SSL
ssl.keystore.location=/path/broker.keystore.jks
ssl.truststore.location=/path/broker.truststore.jks
sasl.mechanism.inter.broker.protocol=PLAIN   # ou SCRAM, GSSAPI
```

💡 **Astuce pro** :  
- Utilisez des certificats distincts pour le trafic inter‑broker et externe, avec des plages d’IP autorisées.  
- En KRaft, les contrôleurs doivent aussi être sécurisés.

---

### 7. Monitoring de la sécurité – journaux et métriques

#### 7.1. Audit logs (Confluent Enterprise)

Active le journal détaillé de toutes les opérations (qui, quand, quoi).

```properties
confluent.metrics.enable=true
confluent.metrics.log.directory=/var/log/kafka-audit
```

Les logs incluent : connexion refusée, ACL modifiée, topic créé/supprimé.

💡 **Astuce pro** :  
- Si vous n’avez pas Confluent Enterprise, les logs `server.log` contiennent les échecs d’authentification et les refus ACL (`Principal xxx is not allowed`).  
- Centralisez ces logs dans un SIEM.

#### 7.2. Métriques JMX de sécurité

| MBean | Description |
|-------|-------------|
| `kafka.security:type=JaasAsService` `authentication-failure-rate` | Taux d’échecs d’authentification (bruteforce) |
| `kafka.network:type=SocketServer,name=FailedAuthenticationRequests` | Requêtes échouées par seconde |
| `kafka.controller:type=KafkaController` `authorization-failure-rate` | Échecs des ACL par type de ressource |

💡 **Astuce pro** :  
- Alerte sur `FailedAuthenticationRequests > 5/min` (possible attaque).  
- Surveillez les changements d’ACL via l’API Admin (peut signaler un compromis).

---

### 8. Mise en œuvre pratique – Guide étape par étape

#### Étape 1 : Chiffrement TLS seul (pas d’authentification)

Générer une CA, un certificat par broker, et un truststore partagé.

#### Étape 2 : Ajout de l’authentification SASL/SCRAM

Créer des utilisateurs (admin, producteur, consommateur). Configurer les brokers avec `sasl.enabled.mechanisms=SCRAM-SHA-256`.

#### Étape 3 : Activer les ACL (mode « autorisation uniquement »)

Définir les règles `allow` explicitement. Vérifier dans les logs que les requêtes non autorisées sont refusées.

#### Étape 4 : Chiffrement inter‑broker

Passer `security.inter.broker.protocol=SASL_SSL` et configurer les certificats broker.

#### Étape 5 : Rotation des secrets

Planifier la rotation des mots de passe des utilisateurs SCRAM et des certificats TLS.

📌 **Exemple de script de rotation** :  
```bash
kafka-configs --bootstrap-server localhost:9092 --alter --add-config 'SCRAM-SHA-256=[password=new-secret]' --entity-type users --entity-name producer
kafka-configs --bootstrap-server localhost:9092 --alter --delete-config 'SCRAM-SHA-256' --entity-type users --entity-name producer
```

💡 **Astuce pro** :  
- Testez les migrations de sécurité en staging avec désactivation temporaire des anciens mécanismes.  
- Conservez un utilisateur de secours (ex: `fallback`) pour ne pas vous enfermer.

---

### 9. Bonnes pratiques résumées

- **Toujours TLS** en production (même avec SASL).  
- **Préférer SCRAM à PLAIN** ; **Kerberos** si l’infrastructure existe.  
- **Super users** limités à l’automatisation critique.  
- **Ne jamais partager les fichiers `.jks`** – isolez par service.  
- **Auditez régulièrement les ACL** (`kafka-acls --list`).  
- **Séparez les certificats brokers des clients** (CAs différentes).  
- **Utilisez PKCS12** au lieu de JKS (plus sûr et interopérable).  
- **Prévoyez la rotation** (certificats tous les 1–2 ans, mots de passe tous les 90 jours).

---

### 10. Dépannage rapide des erreurs de sécurité

| Erreur log | Cause fréquente | Solution |
|------------|----------------|----------|
| `SSLHandshakeException` | Certificat non valide, date désynchronisée | Vérifier keystore, truststore, heure NTP |
| `SASL authentication failed` | Mauvais mot de passe ou utilisateur inconnu | Reconfigurer SCRAM ou vérifier JAAS |
| `Authorization failed` | Pas d’ACL pour l’opération | Ajouter ACL avec `kafka-acls --add` |
| `No password configured` | Fichier JAAS mal configuré | Vérifier la section `KafkaServer` |
| `Timeout` après authentification | ACL manquante pour le groupe de consommateurs | Ajouter `--group` avec `--operation Read` |

💡 **Astuce pro** :  
Activez le log `DEBUG` pour `kafka.security.auth` et `kafka.network.RequestChannel` dans `log4j.properties` pour tracer chaque requête rejetée.

---

## Conclusion du chapitre 8 – points clés

- **Authentification** : mTLS, SCRAM, Kerberos, OAuthBearer.  
- **Autorisation** : ACL fines sur topics, groupes, etc.  
- **Chiffrement** : TLS obligatoire ; données au repos via disque chiffré ou applicatif.  
- **Gestion des secrets** : Vault, variables d’environnement, rotation planifiée.  
- **Monitoring** : audits logs, métriques JMX (taux d’échecs).  
- **Bonnes pratiques** : moindre privilège, super users limités, tests de migration.

La sécurité dans Kafka n’est pas optionnelle en production. Ce chapitre vous donne les clés pour construire un cluster digne de confiance. 🚀
