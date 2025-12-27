# Guide Complet Kafka Streams - Questions & Réponses

## Table des matières

- [Concepts Fondamentaux](#concepts-fondamentaux)
- [Structures de Données](#structures-de-données)
- [Gestion d'État](#gestion-détat)
- [Tolérance aux Pannes](#tolérance-aux-pannes)
- [Fenêtrage et Temps](#fenêtrage-et-temps)
- [Joins et Repartitionnement](#joins-et-repartitionnement)
- [Questions Avancées](#questions-avancées)
- [Design et Production](#design-et-production)

---

## Concepts Fondamentaux

### Qu'est-ce que Kafka Streams exactement ?

**Q:** Pourriez-vous expliquer ce qu'est Kafka Streams et comment il s'intègre dans l'écosystème Kafka ?

**A:** Kafka Streams est une librairie Java permettant de créer des applications de stream processing stateful directement au-dessus de Kafka. Contrairement à un framework distribué traditionnel, ce n'est pas un cluster séparé : c'est une application Java classique qui lit depuis Kafka, traite les données en temps réel et écrit les résultats dans Kafka.

**Avantages clés :**
- Exactly-once semantics nativement supporté
- État local avec réplication via changelog topics
- Déploiement simple (aucun cluster à gérer)
- Haute disponibilité et tolérance aux pannes

---

### Kafka Streams est-il une librairie ou un framework ?

**Q:** Quelle est la distinction entre Kafka Streams et un vrai framework de streaming ?

**A:** Kafka Streams est une **librairie**, pas un framework. Il n'y a pas de cluster séparé, pas de master, pas de worker. C'est simplement une dépendance Java que vous embarquez dans votre application. Cela contraste avec des solutions comme Apache Flink qui sont des frameworks distribués complets.

---

### Comment Kafka Streams se compare-t-il aux Consumer/Producer bas niveau ?

**Q:** Quelle est la différence entre utiliser directement les APIs Consumer/Producer et Kafka Streams ?

**A:** 

| Aspect | Consumer/Producer | Kafka Streams |
|--------|------------------|---------------|
| Niveau d'abstraction | Bas niveau | Haut niveau |
| Gestion d'état | Aucune par défaut | Stateful nativement |
| Offsets et state | Gestion manuelle | Automatique |
| DSL disponible | Non | Oui (Topology DSL + Processor API) |

Kafka Streams encapsule les Consumer/Producer tout en ajoutant une couche de gestion d'état, de windowing et d'agrégations automatiques.

---

## Structures de Données

### Quelle est la différence entre KStream et KTable ?

**Q:** Quand devriez-vous utiliser un KStream plutôt qu'une KTable ?

**A:** 

**KStream** représente un flux d'événements infini où chaque message est traité indépendamment :
- Chaque nouveau message = nouvel événement
- Pas de suppression, pas de mise à jour
- Exemple : logs, transactions, clickstreams

**KTable** représente une vue de l'état courant, où la clé mappe à la dernière valeur :
- Mise à jour remplace l'état précédent
- Comportement similaire à une table SQL
- Exemple : profil utilisateur, taux de change actuel

**Analogie :** KTable = changelog d'une table, KStream = journal infini.

---

### Qu'est-ce qu'une GlobalKTable et quand l'utiliser ?

**Q:** Comment fonctionne une GlobalKTable et en quoi diffère-t-elle d'une KTable ordinaire ?

**A:** Une GlobalKTable est une KTable entièrement répliquée sur chaque instance Kafka Streams, contrairement à une KTable partitionnée.

**Cas d'utilisation idéaux :**
- Petites tables de référence (pays, taux, configurations)
- Joins sans repartitionnement coûteux
- Clé de lookup différente de la clé Kafka

**Avantages :**
- Pas de repartitionnement obligatoire
- Lookup local très rapide
- Pas de shuffle Kafka

**À éviter si :**
- Table volumineuse
- Mises à jour très fréquentes
- Mémoire disponible limitée

---

## Gestion d'État

### Qu'est-ce qu'un state store et comment fonctionne-t-il ?

**Q:** Comment Kafka Streams gère-t-il l'état persistant ?

**A:** Un state store est un stockage local utilisé par Kafka Streams pour maintenir l'état entre les traitements. Il permet les opérations comme :
- Agrégations (count, sum, etc.)
- Joins entre streams
- Windowing

**Types de state stores :**
- **RocksDB** (persistant, par défaut) : stockage disk-based très performant
- **In-memory** (volatil, rare en production)

**Caractéristiques essentielles :**
- Chaque store est sauvegardé dans un changelog topic Kafka
- En cas de crash, l'état est reconstruit automatiquement depuis Kafka
- Kafka reste la source de vérité, le disque local est un cache

---

### Comment Kafka Streams nettoie-t-il automatiquement le state store ?

**Q:** Sans utiliser le windowing, le state store est-il jamais supprimé automatiquement ?

**A:** **Non.** Sans windowing, Kafka Streams conserve l'état indéfiniment. Le state :
- Grandit indéfiniment
- Reste en mémoire/disque
- N'est jamais supprimé automatiquement

C'est dangereux quand la cardinalité des clés est non bornée (userId, sessionId infinis).

**Solutions pour implémenter une durée de vie (TTL) :**

1. **Windowing** (meilleure pratique)
   ```
   TimeWindows.ofSizeWithGrace(
       Duration.ofMinutes(10),
       Duration.ofMinutes(2)
   )
   ```
   Kafka supprime automatiquement l'état après window_end + grace.

2. **Retention sur le state store**
   ```
   Materialized.as("store")
     .withRetention(Duration.ofHours(1))
   ```

3. **Suppression manuelle via Processor API**
   - Stocker un timestamp
   - Utiliser punctuate() pour nettoyer régulièrement

4. **Tombstones (KTable uniquement)**
   - Envoyer une valeur null supprime la clé

---

### Que signifie TTL en streaming ?

**Q:** Comment implémentez-vous une durée de vie limite pour les données ?

**A:** **TTL** (Time To Live) est la durée maximale pendant laquelle une donnée reste valide. Passé ce délai, elle doit être supprimée.

**En Kafka Streams, le TTL s'implémente principalement via le windowing :**

```
TimeWindows.ofSizeWithGrace(
    Duration.ofMinutes(10),   ← durée de la fenêtre (TTL principal)
    Duration.ofMinutes(2)     ← délai accepted pour retards
)
```

Les données sont supprimées après : 10 min + 2 min = 12 minutes.

**Pourquoi c'est critique :**
- Sans TTL : state store grandit indéfiniment ❌
- Avec TTL : state store reste maîtrisé ✅
- Rebalances plus rapides
- Coûts réduits

---

### Où RocksDB stocke-t-il le state store ?

**Q:** Comment récupérer l'état Kafka Streams en cas de besoin ?

**A:** RocksDB stocke le state store sur le disque local de l'instance :

```
/tmp/kafka-streams/<application.id>/
├── 0_0/              (task)
│   ├── rocksdb/
│   │   ├── orders-store/
│   │   └── ...
│   └── checkpoint
├── 0_1/
└── global/           (GlobalKTable)
```

**Points critiques :**

- Chaque task a son propre dossier et RocksDB isolé
- **Kafka reste la source de vérité**, pas le disque
- En cas de crash, l'état est reconstruit depuis le changelog topic
- Pour modifier le chemin :
  ```
  state.dir=/var/lib/kafka-streams
  ```

**En production :**
- Utiliser un disque SSD rapide
- Assez d'espace disponible
- Volume persistant en Kubernetes

---

## Tolérance aux Pannes

### Comment Kafka Streams garantit-il la tolérance aux pannes ?

**Q:** Que se passe-t-il quand une instance Kafka Streams tombe ?

**A:** Kafka Streams utilise une architecture résiliente basée sur Kafka :

**Mécanisme de récupération :**
1. Crash d'une instance
2. Rebalance automatique du consumer group
3. Redistribution des tasks aux instances restantes
4. Restauration de l'état depuis les changelog topics
5. Reprise du traitement

**Éléments clés :**
- **Changelog topics** : chaque state store a un topic Kafka associé
- **Exactly-once semantics** : transactions Kafka garantissent pas de doublon
- **Standby replicas** : copies passives du state pour redémarrage rapide

**Résultat :** Transparence totale pour l'utilisateur.

---

### Qu'est-ce que Exactly Once Semantics (EOS) ?

**Q:** Comment Kafka Streams assure-t-il qu'aucun message n'est traité deux fois ?

**A:** Exactly-once Semantics (EOS) garantit que :
- Aucun message n'est perdu
- Aucun message n'est traité deux fois
- Les résultats sont idempotents

**Implémentation technique :**
- Transactions Kafka
- Producers idempotents
- Commit atomique d'offsets + output

**Configuration :**
```
processing.guarantee=exactly_once_v2
```

**Comparaison avec at-least-once :**

| Sémantique | Garantie | Perfs | Complexité |
|-----------|----------|-------|-----------|
| At-least-once | Messages dupliqués possibles | ⚡ Rapide | ✅ Simple |
| Exactly-once | Aucun doublon | 🐢 Plus lent | ❌ Complexe |

At-least-once est plus rapide mais demande une logique idempotente côté métier.

---

### Différence entre standby replicas et state store actif

**Q:** À quoi servent les standby replicas et quand les configurer ?

**A:** 

**State store actif :**
- Utilisé pour le traitement réel
- Produit les résultats
- Assigné à une task active

**Standby replica :**
- Copie passive synchronisée
- Lit le changelog topic
- Ne traite aucun record
- Devient actif en cas de panne

**Avantages des standby replicas :**
- Redémarrage plus rapide (état déjà local)
- Moins de reprocessing du changelog
- Meilleure disponibilité

**Configuration :**
```
num.standby.replicas=1
```

**Important :** Une standby replica ne peut pas traiter les partitions. Elle ne fait que synchroniser l'état.

---

## Fenêtrage et Temps

### Comment fonctionne le windowing dans Kafka Streams ?

**Q:** Pourquoi avez-vous besoin de fenêtrer vos données en streaming ?

**A:** Le windowing découpe un flux infini en intervalles de temps pour permettre l'agrégation. Sans windowing, vous ne pouvez pas agréger "pour toujours".

**Types de fenêtres :**

**1. Tumbling (fixe, non chevauchant)**
```
|----5min----|----5min----|----5min----|
```
- Fenêtres indépendantes
- Pas de chevauchement
- Performance optimale
- Cas : métriques par minute

**2. Hopping (fixe, chevauchant)**
```
|----5min----|
     |----5min----|
```
- Fenêtres qui se chevauchent
- Recalcul plus fréquent
- Coût en état plus élevé

**3. Sliding (basé sur la distance)**
- Comparaison événement-à-événement
- Pas aligné sur l'horloge
- Cas : détection de fraude

**4. Session (basé sur l'inactivité)**
- Fenêtres dynamiques
- Se fusionnent automatiquement
- Cas : sessions utilisateur

---

### Qu'est-ce que le grace period ?

**Q:** Comment gérez-vous les événements qui arrivent en retard ?

**A:** Le grace period définit combien de temps Kafka Streams accepte les événements arrivant après la fermeture théorique d'une fenêtre.

```
TimeWindows.ofSizeAndGrace(
    Duration.ofMinutes(5),    ← taille de fenêtre
    Duration.ofMinutes(2)     ← grace period
)
```

**Comportement :**
- Fenêtre : 0-5 min
- Grace : jusqu'à 7 min (5 + 2)
- Après 7 min : événements rejetés

**Sans grace period :**
- Événements rejetés immédiatement après fermeture
- Perte de data valide

**Avec grace period :**
- Tolérance aux retards réseau
- Recalcul des agrégations
- Impact sur le state store (gardé plus longtemps)

---

### Event time vs Processing time

**Q:** Quelle est la différence entre le timestamp du message et le moment du traitement ?

**A:** 

**Event time :**
- Timestamp du message Kafka (quand l'événement s'est réellement produit)
- Défini par le producteur ou un TimestampExtractor
- Préféré par Kafka Streams

**Processing time :**
- Moment exact où l'application traite le message
- Variable (latence réseau, retards)
- Non recommandé pour les agrégations

**Pourquoi Kafka Streams préfère event time :**
- Gère les retards et le désordre
- Idempotent (même resultat quel que soit quand on reprocess)
- Compatible avec reprocessing

---

## Joins et Repartitionnement

### Pourquoi le repartitionnement est-il coûteux ?

**Q:** Comment minimiser les repartitionnements dans votre topologie ?

**A:** Un repartitionnement est déclenché quand Kafka Streams doit reshuffle les données :
- `groupBy()` change la clé
- Un join sur clés différentes
- Agrégations non-keyed

**Coût du repartitionnement :**
- I/O réseau vers Kafka
- Latence accrue
- Stockage Kafka supplémentaire
- Changelog topic créé automatiquement

**Solutions pour l'éviter :**

**1. Bien choisir la clé dès la production (MEILLEURE SOLUTION)**
- Design le contrat dès le départ
- Clé Kafka = clé de jointure
- Évite selectKey() plus tard

**2. Utiliser groupByKey() plutôt que groupBy()**
```
stream.groupByKey()  // pas de changement, pas de repartition
```

**3. Utiliser GlobalKTable pour les petites tables**
- Pas de repartition
- Lookup local

**4. Pré-partitionner les topics**
- Même nombre de partitions
- Même stratégie de hashing
- Même clé de partition

**5. Accepter le repartition si nécessaire**
- Mais en le contrôlant :
  - Nombre de partitions
  - Rétention
  - Compaction

---

### Différence entre un join KStream-KTable et KStream-KStream

**Q:** Quel type de join choisir selon vos données ?

**A:** 

**KStream-KTable (inner join)**
```
Stream d'événements + Table d'état
= Enrichissement d'événements
```
- L'événement est enrichi avec la valeur courante de la table
- Pas de windowing requis
- Cas : enrichir les clics avec le profil utilisateur

**KStream-KStream (join)**
```
Deux streams d'événements
= Corrélation temporelle
```
- Nécessite une fenêtre (window)
- Événements proche dans le temps sont joinés
- Cas : détecter les paires achat-retour

**KTable-KTable (join)**
```
Deux tables d'état
= Jointure statique
```
- Pas de windowing
- Mises à jour déclenchent le rejoin
- Cas : produit + catégorie

---

### Combien de partitions un task peut-il traiter ?

**Q:** Comment sont assignées les partitions aux tasks ?

**A:** Un task Kafka Streams peut traiter plusieurs partitions. L'assignation dépend de la topologie.

**Règle clé :**
> Le nombre de tasks est déterminé par le nombre maximal de partitions parmi les topics source co-partitionnés.

**Exemple :**
```
Topic A : 6 partitions
Topic B : 6 partitions
Join A-B (co-partitionné)

Résultat : 6 tasks
Task-0 : A-0 + B-0
Task-1 : A-1 + B-1
...
Task-5 : A-5 + B-5
```

**Cardinalité :**
- Une instance = plusieurs stream threads
- Un thread = traite UNE task à la fois
- Une task = peut avoir plusieurs partitions assignées

---

## Questions Avancées

### Qu'est-ce que le changelog topic ?

**Q:** Comment Kafka Streams persiste-t-il l'état localement ?

**A:** Un changelog topic est un topic Kafka interne associé à chaque state store. Il enregistre chaque modification d'état :

**Mécanisme :**
1. Modification du state store local
2. Écriture atomique dans le changelog topic
3. Kafka devient la source de vérité

**Avantages :**
- Tolérance aux pannes (reconstruction depuis Kafka)
- Rebalancing sans perte d'état
- Kafka = backup automatique
- Compaction possible

**Exemple :**
```
State store "orders" → Topic "_orders-stream-orders-changelog"
```

**En cas de crash :**
1. Instance redémarre
2. Rejoue le changelog topic
3. Reconstruit l'état exactement
4. Reprend le traitement

---

### Pourquoi Kafka Streams utilise-t-il RocksDB ?

**Q:** Quels sont les avantages de RocksDB par rapport aux alternatives ?

**A:** 

**RocksDB :**
- Stockage disk-based performant
- Très rapide pour les opérations key-value
- Gère de gros volumes d'état
- Intégré nativement dans Kafka Streams
- **Choix par défaut en production**

**Alternative : In-Memory**
- Plus rapide
- Mais volatil (perte en crash)
- Rarement utilisé en production
- Déploiement sur une seule instance uniquement

**RocksDB par défaut car :**
- Robustesse
- Performance acceptable
- Scalabilité (disque > RAM)
- Durabilité

---

### Comment gérer les erreurs de désérialisation ?

**Q:** Que faire quand un message ne peut pas être parsé ?

**A:** Kafka Streams permet de gérer les erreurs de manière granulaire :

**Handlers disponibles :**

1. **DeserializationExceptionHandler**
   - Gère les erreurs de parsing
   - Options : skip ou fail

2. **ProductionExceptionHandler**
   - Gère les erreurs lors de l'écriture
   - Options : continue ou fail

3. **Dead Letter Topic**
   - Rediriger les messages problématiques
   - Topic séparé pour analyse

**Approche recommandée :**
```
Essayer → En cas d'erreur → Rediriger vers dead-letter
```

**Importance :**
- Évite que 1 message corrompu arrête le pipeline
- Traçabilité des erreurs
- Retraitement manuel possible

---

### Comment sécuriser une application Kafka Streams ?

**Q:** Quels mécanismes de sécurité implémenter en production ?

**A:** 

**Authentification et Chiffrement :**
- SSL/TLS pour les communications Kafka
- SASL pour l'authentification (SASL/PLAIN, SASL/SCRAM)

**Autorisation :**
- ACLs Kafka (Access Control Lists)
- Contrôle granulaire par topic

**Données sensibles :**
- Encryption at rest
- Stockage des secrets via Vault ou Kubernetes

**Best practices :**
- Secrets en environnement, pas en code
- Certificats valides et renouvelés
- Audit logging
- Monitoring des accès

---

## Design et Production

### Quand NE PAS utiliser Kafka Streams ?

**Q:** Quels cas d'usage ne conviennent pas à Kafka Streams ?

**A:** 

**Kafka Streams n'est pas adapté si :**

1. **Très gros batchs historiques**
   - Kafka Streams = streaming, pas batch
   - Préférer : Spark, Flink

2. **Streaming SQL pur**
   - Utiliser : ksqlDB
   - Kafka Streams = code Java

3. **Besoin multi-langage**
   - Kafka Streams = Java/Scala uniquement
   - Préférer : Flink, Spark

4. **DAGs très complexes**
   - Kafka Streams = topologies simples/modérées
   - Préférer : Flink

5. **State distribué requis**
   - Kafka Streams = state local
   - Préférer : Flink avec state distribué

---

### Comment migrer une application Kafka Streams sans downtime ?

**Q:** Comment passer à une nouvelle version de topologie ?

**A:** 

**Stratégie recommandée : Nouveau consumer group (GOLD STANDARD)**

```
App v1 (application.id = orders-stream-v1)
    ↓
Deploy App v2 (application.id = orders-stream-v2)
    ↓
Reprocess depuis le début ou offset spécifique
    ↓
Double écriture temporaire (v1 et v2)
    ↓
Vérification des résultats
    ↓
Switch downstream vers v2
    ↓
Retrait de v1
```

**Avantages :**
- ✅ Zero downtime
- ✅ Rollback facile
- ✅ Test en parallèle

**Inconvénients :**
- ❌ Double coût temporaire
- ❌ Double trafic

**Alternative : Versionnement des topics**
```
output-v1 → output-v2
```
- Switch consumer downstream
- Rollback facile

**À ÉVITER absolument :**
- ❌ Modifier la topologie avec le même application.id
- ❌ Toucher aux state stores manuellement

---

### Comment versionner une topologie Kafka Streams ?

**Q:** Quel est l'impact d'une modification de topologie ?

**A:** Une topologie Kafka Streams est **immuable**. Tout changement significatif nécessite une nouvelle version.

**Ce qui change = nouvelle topologie :**
- Logique de processing
- Joins
- Fenêtres
- State stores

**Comment versionner :**

**1. Application ID versionnée**
```
application.id = "orders-stream-v2"
```

**2. Noms de state stores versionnés**
```
Materialized.as("orders-store-v2")
```

**3. Topics de sortie versionnés**
```
output-v1 → output-v2
```

**4. Changelog topics**
- Créés automatiquement
- Liés au state store

**Stratégie de rollback :**
- Garder v1 et v2 en parallèle
- Switcher le traffic progressivement

---

### Comment gérer un state store énorme ?

**Q:** Que faire quand le state store devient trop volumineux ?

**A:** 

**Problèmes d'un state store énorme :**
- Temps de restore très longs
- Disque local saturé
- Rebalances lentes
- Startup lent après crash

**Solutions avancées :**

**1. Réduire la taille du state**
- TTL (Time To Live) sur les données
- Windows avec grace period court
- Suppression proactive des données obsolètes

**2. Utiliser des standby replicas**
```
num.standby.replicas=1
```
- Restore plus rapide
- Moins de replay du changelog
- Disponibilité améliorée

**3. Augmenter le parallélisme**
- Plus de partitions
- State plus petit par task
- Meilleure scalabilité

**4. Compression et compaction**
- Changelog compacté
- Réduction de la taille Kafka

**5. Externaliser (si nécessaire)**
- Redis, Cassandra
- ⚠️ Perte possible d'exactly-once

**Approche recommandée :**
```
TTL + Standby replicas + Augmentation du parallélisme
```

---

### Comment tester Kafka Streams ?

**Q:** Comment valider votre logique sans déployer en production ?

**A:** 

**Tests unitaires :**
- **TopologyTestDriver** : test sans Kafka
- Mock des state stores
- Assertions simples

**Tests d'intégration :**
- **Testcontainers** : Kafka en Docker
- Topologie réelle
- Scénarios complexes

**Approche recommandée :**
1. Tests unitaires avec TopologyTestDriver (rapides)
2. Tests d'intégration avec Testcontainers (complets)
3. Déploiement canary en production

---

### Différence entre DSL et Processor API

**Q:** Quand utiliser le Processor API plutôt que la DSL ?

**A:** 

| Aspect | DSL | Processor API |
|--------|-----|---------------|
| Niveau | Haut niveau | Bas niveau |
| Facilité | ⭐⭐⭐⭐⭐ Facile | ⭐⭐ Complexe |
| Paradigme | Déclaratif | Impératif |
| Contrôle | Standard | Total |

**DSL (Recommended)** :
```java
stream.groupByKey()
      .windowedBy(TimeWindows.ofSize(...))
      .count()
```
- Lisible
- Performant
- Peu de code

**Processor API** :
```java
topology.addProcessor("custom", CustomProcessor::new, ...)
```
- Flexible
- Custom logic
- State contrôlé

**Utiliser Processor API si :**
- Logique très complexe
- Custom state management
- Custom punctuation
- Optimisations bas-niveau

---

### Qu'est-ce que Punctuation ?

**Q:** Comment exécuter du code à intervalle régulier ?

**A:** Punctuation est un mécanisme pour exécuter du code périodiquement dans un Processor :

**Types :**
- **Event time** : basé sur le timestamp des messages
- **Wall-clock time** : horloge système

**Cas d'usage :**
- Nettoyage de state
- Flush manuel
- Exécution périodique

**Exemple :**
```java
context.schedule(
  Duration.ofMinutes(5),
  PunctuationType.WALL_CLOCK_TIME,
  timestamp -> {
    // Exécuté chaque 5 minutes
    cleanupExpiredEntries();
  }
);
```

---

### Combien de fois une standby replica peut traiter ?

**Q:** Est-ce qu'une standby replica traite des partitions ?

**A:** **Non, jamais.** Une standby replica ne traite aucun record.

**Rôle d'une standby replica :**
- Synchronisation passive du state
- Lecture du changelog topic
- Préparation à devenir active

**Quand elle devient active :**
- Lors d'un crash d'une instance
- Lors d'un rebalance
- Lors d'un scaling

**Pourquoi ne pas traiter ?**
- ❌ Double processing = violation exactly-once
- ❌ Incohérence de state
- ❌ Résultats dupliqués

**Note spéciale : GlobalKTable**
- Pas de standby replicas
- Chaque instance a une copie active
- Chargée par un global thread

---

### GlobalKTable nécessite-t-elle une co-partition ?

**Q:** Pourquoi GlobalKTable n'a pas besoin d'être co-partitionnée ?

**A:** Parce que **chaque instance possède une copie complète** de la GlobalKTable.

**Contrairement à KTable :**
- KTable est partitionnée
- Chaque partition sur une instance différente
- Join nécessite co-partition

**GlobalKTable :**
- Répliquée entièrement sur chaque instance
- Join = lookup local en mémoire/RocksDB
- **Pas de shuffle Kafka**

**Illustration :**
```
KTable (partitionnée) :
Partition 0 → Instance A
Partition 1 → Instance B

GlobalKTable (répliquée) :
Instance A : [Partition 0, 1, 2, ...]
Instance B : [Partition 0, 1, 2, ...]
Instance C : [Partition 0, 1, 2, ...]
```

**Clé différente :**
```java
stream.join(
  globalTable,
  (streamKey, streamValue) -> streamValue.getCountryCode(),
  joiner
)
```
- Lookup par clé différente de la clé Kafka
- Pas de repartitionnement

**Trade-offs :**

| Avantage | Inconvénient |
|----------|-------------|
| Pas de repartition | Consomme mémoire/disque |
| Latence basse | Scale pas bien |
| Simple | Table volumineuse = risqué |

---

### Comment implémenter un compteur temps réel avec expiration ?

**Q:** Design : compteur utilisateur par minute avec TTL ?

**A:** 

**Approche :**

```java
stream
  .groupByKey()
  .windowedBy(
    TimeWindows.ofSizeWithGrace(
      Duration.ofMinutes(1),   // fenêtre = 1 min
      Duration.ofSeconds(10)   // accepter 10s de retard
    )
  )
  .count()
  .toStream()
  .to("counts-output");
```

**Éléments :**
1. **KStream** : flux des événements utilisateur
2. **groupByKey()** : regrouper par clé (userId)
3. **windowedBy()** : créer des fenêtres de 1 min
4. **count()** : compter
5. **Grace period** : 10s pour accepter les retards

**Où c'est stocké :**
- State store local (RocksDB)
- Changelog topic pour réplication
- TTL = window size + grace = 1 min 10s

**Après 1 min 10s :**
- Fenêtre fermée
- État supprimé automatiquement
- Partition écrite dans output topic

---

### Comment gérer le schema evolution ?

**Q:** Comment gérer les changements de schéma en production ?

**A:** 

**Outil recommandé :**
- **Avro** ou **Protobuf** + Schema Registry

**Modes de compatibilité :**
- **Backward** : nouveaux consumers lisent vieilles données
- **Forward** : vieux consumers lisent nouvelles données
- **Full** : les deux

**Attention aux state stores :**
- Changement de schéma = reprocessing possible
- State incompatible = corruption
- Version du state = nouveau application.id

**Approche :**
1. Nouvelle version du schéma
2. Vérifier compatibilité
3. Déployer avec nouveau application.id si break
4. Reprocess du début

---

### Comment monitorer Kafka Streams ?

**Q:** Quelles métriques surveiller ?

**A:** 

**Métriques disponibles :**
- **JMX Metrics** : détaillées
- **Lag par consumer group** : retard de traitement
- **State store size** : croissance de l'état
- **Throughput / Latency** : performance

**Outils de monitoring :**
- Prometheus + Grafana
- Confluent Control Center
- Datadog, New Relic

**Metriques clés à alerter :**
- Lag trop élevé (retard)
- State store dépasse limite
- Rebalances fréquentes
- Erreurs de processing

---

## Questions Supplémentaires Avancées

### Comment fonctionne exactement le repartition topic en interne ?

**Q:** Quelles sont les étapes exactes lors d'un repartitionnement ?

**A:** Le repartitionnement intervient quand une opération change la clé. Voici le processus :

**Étapes :**
1. Kafka Streams détecte un changement de clé (`groupBy`, `selectKey`, etc.)
2. Un repartition topic est créé automatiquement
3. Les messages sont reclés (répartitionnés par nouvelle clé)
4. Les partitions sont redistribuées
5. Un changelog topic est associé au repartition topic

**Exemple concret :**
```java
stream
  .selectKey((k, v) -> v.getCountryCode())  // Change la clé !
  .groupByKey()
  .count()
```

**Coûts associés :**
- Écriture réseau vers Kafka
- Lecture depuis le repartition topic
- Latence accrue
- I/O supplémentaire

**Comment le reconnaître :**
- Topics Kafka avec pattern : `<app-id>-TOPIC-repartition`
- Augmentation du lag
- Surcharge I/O

---

### Quelle est la différence entre StreamsBuilder et Topology ?

**Q:** À quel niveau de la hiérarchie Kafka Streams travaille-t-on ?

**A:** 

**StreamsBuilder :**
- API DSL haut niveau et facile
- Méthode fluente pour construire la topologie
- Recommandée pour 95% des cas

```java
StreamsBuilder builder = new StreamsBuilder();
builder.stream("input")
       .filter(...)
       .to("output");
```

**Topology :**
- Représentation interne bas niveau
- Générée automatiquement par StreamsBuilder
- Peut être manipulée directement si besoin
- Moins intuitive

**Relation :**
```
StreamsBuilder.build() → Topology
                       ↓
                 KafkaStreams(topology)
```

**En production :**
- Kafka Streams exécute une Topology
- StreamsBuilder = API de développement
- Vous interagissez avec StreamsBuilder, Kafka Streams gère la Topology

---

### Comment fonctionne le task model de Kafka Streams ?

**Q:** Comment le parallélisme est-il implémenté ?

**A:** Le task model est la base du parallélisme dans Kafka Streams.

**Définition :**
- Une **task** = un sous-ensemble de partitions d'un topic
- Une task est traitée par **un et un seul thread**
- Une task = unité d'isolation et d'état

**Composition d'une task :**
- Partitions assignées (ex: partition 0, 1, 2)
- State stores propres
- Changelog topics associés
- Offsets commités

**Parallélisme :**
```
Partitions disponibles = limite maximale de tasks
Tasks = limite du parallélisme possible

Exemple :
Topic : 12 partitions
→ Maximum 12 tasks en parallèle
→ Scaling max = 12 instances
```

**Cardinalité :**
```
1 instance = plusieurs threads
1 thread = traite 1 task à la fois
1 task = peut avoir plusieurs partitions
```

**Cas : 6 partitions, 4 threads :**
```
Thread 1 : Task-0 (partitions 0, 3)
Thread 2 : Task-1 (partitions 1, 4)
Thread 3 : Task-2 (partitions 2, 5)
Thread 4 : (idle)
```

---

### Qu'est-ce que Topology.describe() et pourquoi l'utiliser ?

**Q:** Comment visualiser la structure de votre topologie ?

**A:** `Topology.describe()` fournit une représentation textuelle de votre topologie, très utile pour déboguer.

**Utilité :**
- Vérifier la structure créée
- Valider les connexions
- Trouver les erreurs de design
- Documenter

**Exemple de sortie :**
```
Sub-topologies:
Sub-topology: 0
  Source: KSTREAM-SOURCE-0000000000 (topics: [orders])
    → Processor: KSTREAM-MAPVALUES-0000000001
      → Processor: KSTREAM-FILTER-0000000002
        → Sink: KSTREAM-SINK-0000000003 (topic: filtered-orders)
```

**À toujours utiliser :**
```java
Topology topology = builder.build();
System.out.println(topology.describe());
```

**Vous verrez :**
- Sources (topics consommés)
- Processors (opérations)
- Sinks (topics écrits)
- Repartition topics
- State stores

---

### Comment gérer les keyed vs non-keyed streams ?

**Q:** Quelle est l'importance de la clé en Kafka Streams ?

**A:** 

**Non-keyed stream :**
- Pas de clé (null ou ignorée)
- Chaque message = individuel
- Pas de joins possibles
- Pas d'état par clé

```java
stream.filter(...).map(...)  // clé ignorée
```

**Keyed stream :**
- Message = (clé, valeur)
- Clé détermine la partition
- Joins par clé
- État par clé

```java
stream.groupByKey()  // regroupe par clé
      .count()       // state par clé
```

**Impact critique :**
```
KStream non-keyed → count() → erreur !
KStream keyed → count() → OK
```

**Comment fixer une clé :**
```java
stream.selectKey((k, v) -> v.getUserId())
      .groupByKey()
      .count()
```

**Considérations :**
- Clé = partitionnement
- Clé = co-location des données
- Clé bien choisie = performance
- Clé mal choisie = repartition coûteux

---

### Comment Kafka Streams gère-t-il le rebalance en détail ?

**Q:** Que se passe-t-il exactement lors d'un rebalance ?

**A:** Le rebalance est un processus complexe et critique pour la disponibilité.

**Déclencheurs du rebalance :**
- Nouvelle instance lancée
- Instance crashée (timeout détecté)
- Consumer quitte le groupe
- Rééquilibrage du nombre d'instances

**Étapes du rebalance :**

1. **Pause du traitement**
   - Tous les threads s'arrêtent
   - État sauvegardé

2. **Révocation des tasks**
   ```
   onPartitionsRevoked()
   ```
   - Flush de l'état
   - Commit des offsets

3. **Réassignation des partitions**
   - Calcul de la nouvelle distribution
   - Assigner les tasks aux instances

4. **Rétablissement des tasks**
   ```
   onPartitionsAssigned()
   ```
   - Restauration de l'état depuis Kafka
   - Rattrapage du lag

5. **Reprise du traitement**
   - Normale immédiatement

**Durée typique :**
- Quelques secondes à minutes (selon taille state store)

**Optimisations :**

**Cooperative rebalancing :**
```
processing.guarantee=exactly_once_v2
```
- Rebalance partiel
- Réduit le stop-the-world
- Plus rapide

**Standby replicas :**
```
num.standby.replicas=1
```
- Prépare la restauration
- Rebalance plus rapide

**Impact :**
- Pendant rebalance = zéro traitement
- State à restaurer = latence
- Coordination complexe

---

### Comment Kafka Streams gère le scaling ?

**Q:** Comment ajouter ou retirer des instances ?

**A:** 

**Scaling horizontal (ajouter des instances) :**

Avant :
```
3 instances
12 partitions
→ 4 partitions par instance
```

Après ajout instance 4 :
```
4 instances
12 partitions
→ 3 partitions par instance
→ Rebalance automatique
```

**Kafka Streams fait automatiquement :**
1. Détecte la nouvelle instance
2. Rebalance les tasks
3. Redistribue les partitions
4. Restaure l'état

**Scaling limité par :**
- Nombre de partitions (max N instances pour N partitions)
- Nombre de clés (pour non-windowed)
- Ressources CPU/RAM par instance

**Cas limite :**
```
10 partitions
100 instances
→ 90 instances idle (sans work)
→ Coûteux !
```

**Solution : Augmenter les partitions**
```
10 partitions → 100 partitions
+ Reprocessing du changelog
+ Temps initial
```

**Best practice :**
- Partir avec 2x le nombre d'instances prévu
- Augmenter les partitions avant les instances
- Monitorer le lag pendant rebalance

---

### Qu'est-ce que le cache dans Kafka Streams ?

**Q:** À quoi sert `cache.max.bytes.buffering` ?

**A:** Le cache buffèrise les modifications avant de les écrire au state store et Kafka.

**Comportement :**
```
Update → Cache (en mémoire)
         ↓ (quand plein)
      State store (RocksDB)
         ↓ (async)
      Kafka (changelog)
```

**Configuration :**
```
cache.max.bytes.buffering=10485760  (10 MB par défaut)
```

**Impact :**

**Plus de cache :**
- ✅ Meilleures performances (moins d'I/O)
- ❌ Plus de latence (avant flush)
- ❌ Plus de mémoire

**Moins de cache :**
- ✅ Basse latence
- ❌ Plus d'I/O
- ❌ Overhead I/O

**Relation avec commit.interval.ms :**
```
commit.interval.ms=30000  (30 secondes)
```
- Frequency de flush du cache
- Indépendant de la taille du cache

**Cas d'usage :**
```
High-throughput streaming
→ Gros cache + long commit interval
→ Performance

Low-latency streaming
→ Petit cache + court commit interval
→ Réactivité
```

---

### Comment utiliser TopologyTestDriver ?

**Q:** Comment tester Kafka Streams sans Kafka réel ?

**A:** TopologyTestDriver permet de tester des topologies de manière unitaire et rapide.

**Setup :**
```java
TopologyTestDriver testDriver = new TopologyTestDriver(
    builder.build(),
    new Properties()  // StreamsConfig
);

TestInputTopic<String, String> inputTopic =
    testDriver.createInputTopic("input", 
                               new StringSerializer(), 
                               new StringSerializer());

TestOutputTopic<String, Long> outputTopic =
    testDriver.createOutputTopic("output",
                                new StringDeserializer(),
                                new LongDeserializer());
```

**Test :**
```java
inputTopic.pipeInput("user1", "event");
inputTopic.pipeInput("user2", "event");

KeyValue<String, Long> output = outputTopic.readKeyValue();
assertEquals("user1", output.key);
assertEquals(1L, output.value);
```

**Avantages :**
- ✅ Pas de Kafka requis
- ✅ Très rapide
- ✅ Idéal pour CI/CD
- ✅ Facile à déboguer

**Limitations :**
- ❌ Pas de rebalance testé
- ❌ Pas de multi-instance
- ❌ État en-mémoire uniquement

**Bonne pratique :**
```
Tests unitaires : TopologyTestDriver
Tests d'intégration : Testcontainers (Kafka réel)
Tests canary : Production avec faible traffic
```

---

### Comment implémenter un dead letter topic ?

**Q:** Où envoyer les messages non traitables ?

**A:** Un dead letter topic (DLT) capture les messages qui ne peuvent pas être traités.

**Cas d'usage :**
- Erreurs de désérialisation
- Validation échouée
- Exception non gérée

**Implémentation simple :**
```java
KStream<String, String> stream = builder.stream("input");

stream
  .filter((k, v) -> {
    try {
      validateMessage(v);
      return true;
    } catch (Exception e) {
      // Envoyer au DLT
      return false;
    }
  })
  .to("output");
```

**Meilleure approche (avec détail d'erreur) :**
```java
stream
  .flatMap((k, v) -> {
    try {
      List<KeyValue<String, String>> result = new ArrayList<>();
      result.add(new KeyValue<>(k, processMessage(v)));
      return result;
    } catch (Exception e) {
      // Wrapper avec erreur
      String errorPayload = createErrorPayload(v, e);
      List<KeyValue<String, String>> errors = new ArrayList<>();
      errors.add(new KeyValue<>(k, errorPayload));
      return errors;
    }
  })
  .to("output");  // Ou rediriger vers DLT
```

**Pattern recommandé :**
```
Input Stream
    ├── Valid → Output Topic
    ├── Invalid (parse error) → DLT
    └── Validation failed → DLT
```

**Traitement du DLT :**
- Monitoring / Alerting
- Retraitement manuel
- Analyse des erreurs
- Feedback sur la qualité data

---

### Comment Kafka Streams crée-t-il les topics internes ?

**Q:** Quels topics Kafka Streams crée automatiquement ?

**A:** Kafka Streams crée plusieurs topics automatiquement pour son fonctionnement.

**Topics créés :**

**1. Changelog topics**
```
<application-id>-<store-name>-changelog
```
- Un par state store
- Compacté
- Stocke toutes les modifications

**2. Repartition topics**
```
<application-id>-<topic-name>-repartition
```
- Créé si changement de clé
- Stockage temporaire
- Suppression possible après compaction

**3. Topics de sortie**
```
user-defined (vous les créez)
```

**Configuration :**
```
topic.cleanup.policy=compact  (changelog)
topic.cleanup.policy=delete   (repartition)
```

**Nommage et ordre :**
```
Materialized.as("store-name")
→ Topic: <app-id>-store-name-changelog
```

**Impact :**
- Chaque state store = un topic supplémentaire
- Crée une complexité visuelle dans Kafka
- À monitorer

---

### Comment gérer les large state stores ?

**Q:** Quand externaliser l'état hors de Kafka ?

**A:** Parfois, le state store devient trop gros pour RocksDB local.

**Symptômes :**
- Temps de restore > 10 minutes
- Disque saturé
- Rebalances très lents
- Startup lents après crash

**Options :**

**1. Externaliser vers Redis**
```
Avantages : rapide, scalable
Inconvénients : perte EOS, latence réseau
```

**2. Externaliser vers Cassandra**
```
Avantages : scalable, durable
Inconvénients : complexité, coûts
```

**3. Snapshot + S3**
```
Avantages : backup, versioning
Inconvénients : latence au restore
```

**⚠️ Trade-offs critiques :**
```
RocksDB local : EOS ✅, state local ✅, scalabilité ❌
Redis externe : EOS ❌, latence ❌, scalabilité ✅
```

**Recommandation :**
```
Avant d'externaliser :
1. Réduire la durée de vie (TTL)
2. Augmenter les partitions
3. Utiliser standby replicas
4. Optimiser le state store

Si vraiment nécessaire :
→ Externaliser + Processor API custom
→ Mais perdre EOS
```

---

### Qu'est-ce que le stream-time et le watermark ?

**Q:** Comment Kafka Streams gère le temps en streaming ?

**A:** Kafka Streams maintient un stream-time interne pour coordonner les fenêtres.

**Stream-time :**
- Timestamp maximum vu jusqu'à présent
- Avance avec les événements
- Détermine la fermeture des fenêtres

**Watermark (implicite) :**
- Contrairement à Flink, **pas de watermark explicite**
- Mais comportement similaire
- Stream-time = watermark implicite

**Mécanisme :**
```
Message avec timestamp T
→ stream-time = max(stream-time, T)
→ Fenêtres avec stream-time < T fermées
→ État supprimé
```

**Fermeture de fenêtre :**
```
window_end + grace_period < stream-time
→ fenêtre fermée
→ résultat final émis
→ state supprimé
```

**Impact sur le processing :**
- Event time = source de vérité
- Processeurs ne voient pas stream-time directement
- Contrôlé via event-time des messages

---

### Comment gérer le consumer lag en Kafka Streams ?

**Q:** Comment monitorer et réduire le lag ?

**A:** Le lag indique le retard par rapport au dernier message du topic.

**Définition :**
```
Lag = Latest Offset - Current Offset
```

**Causes d'un lag élevé :**
1. Processing lent (logique complexe)
2. State restore long (new instance)
3. Rebalance fréquents
4. Erreurs de processing

**Monitoring :**
```
Métriques JMX :
- kafka.streams.record-lag
- kafka.streams.record-lag-max
```

**Outils :**
- Burrow (LinkedIn)
- Confluent Control Center
- Prometheus exporters

**Comment réduire le lag :**

1. **Optimiser la logique**
   - Éviter opérations coûteuses
   - Filtrer tôt

2. **Augmenter le parallélisme**
   - Plus de partitions
   - Plus d'instances

3. **Optimiser le state store**
   - Réduire la taille
   - Utiliser standby replicas

4. **Monitoring et alertes**
   - Lag > X = alert
   - Trend alerting (lag augmente)

---

### Comment utiliser Processor API pour du custom processing ?

**Q:** Quand et comment utiliser la Processor API ?

**A:** La Processor API offre un contrôle bas niveau quand la DSL n'est pas assez flexible.

**Cas d'usage :**
- Logique complexe multi-état
- Custom punctuation
- Gestion manuelle du state
- Performance critiques

**Exemple : compteur avec TTL manuel**
```java
public class CustomCounterProcessor extends AbstractProcessor<String, String> {
  private KeyValueStore<String, Long> store;
  
  @Override
  public void init(ProcessorContext context) {
    super.init(context);
    store = context.getStateStore("counter-store");
    
    // Punctuation chaque minute
    context.schedule(
      Duration.ofMinutes(1),
      WALL_CLOCK_TIME,
      timestamp -> cleanupExpired()
    );
  }
  
  @Override
  public void process(String key, String value) {
    Long count = store.get(key);
    count = (count == null) ? 1 : count + 1;
    store.put(key, count);
    context.forward(key, count);
  }
  
  private void cleanupExpired() {
    try (KeyValueIterator<String, Long> iter = store.all()) {
      while (iter.hasNext()) {
        // Logique de nettoyage
      }
    }
  }
}
```

**DSL vs Processor API :**
```
DSL : 95% des cas, recommandée
Processor API : 5% spécifiques, quand DSL échoue
```

---

### Comment Kafka Streams gère les clés null ?

**Q:** Que se passe-t-il avec les messages sans clé ?

**A:** 

**Messages sans clé :**
```
null key → partition aléatoire
```

**Impact :**
- `groupByKey()` : non-partitionnné, non garantie locality
- `join()` : impossible
- `aggregate()` : sans state par clé

**Exemple problématique :**
```java
stream.filter(...)           // key peut être null
      .groupByKey()          // Undefined behavior
      .count()               // Résultats imprévisibles
```

**Solution :**
```java
stream
  .selectKey((k, v) -> v.getUserId())  // Créer une clé
  .groupByKey()
  .count()
```

**Quand accepter null :**
- Broadcasts (pas de partitionnement voulu)
- Sink sans partitionnement important

---

### Comment utiliser GlobalKTable avec de gros datasets ?

**Q:** Quand GlobalKTable devient-elle non viable ?

**A:** GlobalKTable a des limites de scalabilité.

**Taille acceptable :**
- < 100 MB : facile
- 100 MB - 1 GB : gérable
- > 1 GB : problématique

**Problèmes avec gros datasets :**
1. **Mémoire consommée :**
   - Chaque instance a une copie complète
   - Multiplicité des instances = multiplicité du coût

2. **Temps de chargement :**
   - Démarrage lent
   - Rebalance long

3. **Bande passante :**
   - Chargement initial coûteux
   - Replication vers chaque instance

**Alternatives :**
1. **Regular KTable + co-partition**
   - Partitionnée
   - Pas de charge complète
   - Nécessite co-partition

2. **Redis / Cache externe**
   - Lookup par clé différente
   - Perte EOS
   - Latence réseau

3. **Diviser en sub-GlobalKTables**
   - GlobalKTable par région/segment
   - Plus complexe

**Recommandation :**
```
< 100 MB : GlobalKTable ✅
100 MB - 500 MB : KTable + co-partition
> 500 MB : Cache externe ou alternative
```

---

### Comment implémenter un back-pressure dans Kafka Streams ?

**Q:** Comment ralentir le processing si la destination est lente ?

**A:** Le back-pressure n'est pas natif dans Kafka Streams. Vous devez l'implémenter.

**Approches :**

**1. Throttling à la source**
```java
stream.filter((k, v) -> {
  if (isDestinationSaturated()) {
    // Traiter moins de messages
    return random.nextDouble() < 0.5;  // Process 50%
  }
  return true;
});
```

**2. Retry avec délai**
```java
try {
  forwardToDownstream(message);
} catch (DestinationUnavailableException e) {
  Thread.sleep(1000);
  retry();
}
```

**3. Pause du consumer**
```java
context.pause();  // Dans Processor API
Thread.sleep(5000);
context.resume();
```

**4. Queue de buffer**
```
Kafka Streams → Buffer Local → Destination
```

**⚠️ Trade-offs :**
- Back-pressure = latence
- Peut causer rebalance (timeout)
- Complexité accrue

**Meilleure pratique :**
```
Dimensionner correctement la destination
Monitorer le lag
Alerter plutôt que back-pressure
```

---

### Quels patterns éviter en Kafka Streams ?

**Q:** Quelles erreurs courantes commettent les développeurs ?

**A:** 

**❌ Anti-patterns courants :**

**1. State store sans windowing**
```java
stream.groupByKey().count()  // ❌ Grandit indéfiniment
```
**✅ Correction :**
```java
stream.groupByKey()
      .windowedBy(TimeWindows.ofSize(...))
      .count()
```

**2. Repartition implicite massif**
```java
stream.selectKey(...).selectKey(...).selectKey(...)  // ❌ Plusieurs repartitions
```
**✅ Correction :**
```java
stream.selectKey((k, v) -> {
  // Calculer la clé finale une fois
  return finalKey(v);
})
```

**3. Side effects sans idempotence**
```java
stream.peek((k, v) -> database.insert(v))  // ❌ Doublon possible
```
**✅ Correction :**
```java
stream.peek((k, v) -> database.upsert(v))  // Idempotent
```

**4. Oublier les exceptions**
```java
stream.map(v -> parseJson(v))  // ❌ NPE non gérée
```
**✅ Correction :**
```java
stream.map(v -> {
  try {
    return parseJson(v);
  } catch (Exception e) {
    return null;  // ou DLT
  }
})
```

**5. Consumer group partagé**
```java
// App 1 et App 2 : même application.id
→ ❌ Compétition, pas parallélisation
```
**✅ Correction :**
```
application.id = "app1-unique"
application.id = "app2-unique"
```

---

### Comment implémenter un reprocessing complet ?

**Q:** Comment rejouer tous les données depuis le début ?

**A:** Le reprocessing peut être nécessaire pour changements de logique.

**Approches :**

**1. Reset des offsets (in-place)**
```bash
kafka-streams-application-reset.sh \
  --application-id orders-stream \
  --input-topics orders \
  --bootstrap-servers localhost:9092
```
**Risques :**
- Downtime
- State corrompu possible
- Résultats doublons

**2. Nouveau consumer group (recommandé)**
```
App v1 : application.id = "orders-v1"
App v2 : application.id = "orders-v2"  // Nouveau
```
- Reprocess indépendant
- Pas de downtime
- Double trafic temporaire

**3. Reprocess partiel (offset spécifique)**
```java
Properties props = new Properties();
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "...");
props.put("auto.offset.reset", "earliest");  // Depuis début
```

**Processus recommandé :**
```
1. Déployer v2 avec nouveau application.id
2. Laisser v2 reprocess
3. Comparer outputs v1 vs v2
4. Switch downstream quand prêt
5. Retirer v1
```

**Timing :**
- Input topic : pas de rétention limite
- Changelog : peut être tronqué
- Coûteux en temps et I/O

---

### Comment gérer les timeouts et les deadlocks ?

**Q:** Que faire quand une opération prend trop longtemps ?

**A:** 

**Timeouts possibles :**

**1. Session timeout**
```
session.timeout.ms=10000
```
- Consumer se considère down
- Rebalance forcé
- Trop bas = faux rebalance

**2. Processing timeout (custom)**
```java
long start = System.currentTimeMillis();
process(message);
if (System.currentTimeMillis() - start > 5000) {
  log.error("Processing too slow!");
}
```

**3. Coordination timeout**
```
connections.max.idle.ms=...
```

**Deadlocks possibles :**

**1. Circular dependencies**
```
Stream A → State of Stream B
Stream B → State of Stream A
→ Deadlock potentiel
```

**2. Blocking I/O dans processor**
```java
process(message) {
  database.query(...).block()  // ❌ Bloque le thread
}
```

**3. State store exhaustion**
```
State rempli → RocksDB saturé → Stall
```

**Solutions :**
- Timeouts courts
- Non-blocking I/O
- Monitoring des threads
- Circuit breakers

---

### Comment optimiser la latence end-to-end ?

**Q:** Quels sont les bottlenecks latence ?

**A:** 

**Sources de latence :**

1. **Commit interval**
   ```
   commit.interval.ms=30000 (par défaut)
   → Latence max = 30s
   ```
   **Optimisation :** réduire à 5-10s

2. **Cache flushing**
   ```
   cache.max.bytes.buffering
   → Remplissage = latence ajoutée
   ```
   **Optimisation :** petit cache + petit commit

3. **RocksDB write**
   ```
   Async vers state store
   → Quelques ms
   ```

4. **Network I/O**
   ```
   Production vers Kafka
   acks=all → latence
   acks=1 → rapide
   ```

5. **Processing logic**
   ```
   Votre code métier
   → Plus important à optimiser
   ```

**Profiling :**
```java
long start = System.nanoTime();
process(message);
long latency = System.nanoTime() - start;

metrics.recordLatency(latency);
```

---

## Résumé Rapide (Entretien)

| Concept | Réponse Clé |
|---------|-------------|
| **Kafka Streams** | Librairie de stream processing stateful |
| **KStream vs KTable** | KStream = événements, KTable = état |
| **GlobalKTable** | Petite table répliquée, pas de repartition |
| **Repartition** | Design de clé optimal dès la production |
| **State store** | RocksDB local + changelog Kafka |
| **TTL** | Windows ou retention explicite |
| **EOS** | Transactions Kafka + producers idempotents |
| **Windowing** | Découpe le temps pour agrégation |
| **Grace period** | Tolère les événements en retard |
| **Migration** | Nouveau application.id toujours |
| **Standby replicas** | Passifs, deviennent actifs en crash |
| **GlobalKTable** | Pas de co-partition requise |
| **Task model** | Unit logique = partition set |
| **Rebalance** | Redistribution + restore |
| **TopologyTestDriver** | Test sans Kafka |
| **Dead Letter Topic** | Capture erreurs |
| **Stream-time** | Watermark implicite |
| **Processor API** | Custom bas niveau |
| **Back-pressure** | À implémenter |
| **Anti-patterns** | State sans window, side effects |

---

**Généré à partir du guide complet Kafka Streams - Niveau Senior** 
**Questions supplémentaires ajoutées pour couverture complète**
