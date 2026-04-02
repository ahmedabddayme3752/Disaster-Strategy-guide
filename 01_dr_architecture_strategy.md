# 01 — Stratégie d'Architecture DR (BNM)
## Comprendre les Choix Techniques

---

## 🎯 Pourquoi un Disaster Recovery ?

Une banque ne peut pas se permettre de perdre des données transactionnelles. Une panne de serveur, un incendie, ou une cyberattaque peut détruire des années de données.

Le DR garantit deux choses fondamentales :

| Indicateur | Signification | Notre Cible |
|:---|:---|:---|
| **RPO** (Recovery Point Objective) | Quelle est la perte de données maximale acceptable ? | **0 seconde** (Aucune perte) sur Zone 2 |
| **RTO** (Recovery Time Objective) | En combien de temps peut-on reprendre le service ? | **< 1 heure** sur Zone 2 |

---

## 🛠️ Technologie Clé : Les GTID

### Qu'est-ce qu'un GTID ?
Un **GTID** (Global Transaction Identifier) est un numéro unique attribué à chaque modification dans la base de données.

**Format :** `UUID_du_serveur:numéro_de_transaction`

**Exemple BNM :** `816c0b5e-97e8-11ed-a73b-000c299194d8:29303`
- `816c0b5e-97e8-11ed-a73b-000c299194d8` = L'identifiant unique du serveur Click Production
- `29303` = La 29 303ème transaction enregistrée sur ce serveur

### Pourquoi c'est révolutionnaire ?
Avant les GTID, pour répliquer une base, il fallait noter manuellement un numéro de fichier binaire et une position. Une seule erreur dans ces numéros et la réplication était corrompue.

Avec les GTID, le serveur esclave dit simplement : *"Je connais jusqu'à la transaction 29 303. Envoie-moi tout ce qui suit."* C'est automatique et infaillible. ✨

---

## 🐳 Pourquoi Docker pour le DR ?

### Le Problème sans Docker
Si on installe MySQL directement sur le serveur DR (en "natif"), on ne peut faire tourner **qu'une seule instance MySQL** par machine. Pour avoir Click ET Pleniged, il faudrait **deux serveurs physiques différents**.

### La Solution Docker
Docker permet de faire tourner **plusieurs instances MySQL isolées** sur la même machine physique, chacune dans son propre "conteneur" :

```
┌────────────────────────────────────────┐
│  Serveur Physique (8 CPU, 16GB RAM)    │
│                                        │
│  🐳 Conteneur 1 : mysql-click          │
│     Port: 3306                         │
│     Données: ./click/data/             │
│                                        │
│  🐳 Conteneur 2 : mysql-pleniged       │
│     Port: 3307                         │
│     Données: ./pleniged/data/          │
│                                        │
│  📡 Conteneur 3 : mysql-exporter-click │
│     Port: 9104                         │
│                                        │
│  📡 Conteneur 4 : mysql-exporter-pln   │
│     Port: 9105                         │
└────────────────────────────────────────┘
```

**Avantages :**
1. **Isolation** : Une panne de Click n'affecte pas Pleniged
2. **Économie** : Un seul serveur pour deux systèmes DR
3. **Reproductibilité** : Le même `docker-compose.yml` fonctionne à l'identique sur Zone 2 et Zone 3

---

## 📡 Stratégie de Réplication Multi-Zones

### Zone 2 — Primary DR (Chaud)
- **Type :** Réplication GTID avec Auto-Position
- **Délai (LAG) :** 0 seconde (en direct)
- **RPO :** Zéro perte de données
- **Activation en cas de sinistre :** `< 1 heure`

### Zone 3 — Secondary DR (Tiède)
- **Type :** Réplication GTID avec Auto-Position
- **Délai (LAG) :** 0 seconde (en direct)
- **RPO :** Zéro perte de données
- **Utilisation :** Secours si Zone 2 et Production sont toutes les deux perdues

---

## 🔐 Sécurité de la Réplication

### Authentification `mysql_native_password`
MySQL 8.0 utilise par défaut un plugin d'authentification avancé (`caching_sha2_password`) qui nécessite un canal SSL chiffré. Pour simplifier la connexion entre les serveurs sans SSL, nous utilisons le plugin `mysql_native_password` pour l'utilisateur de réplication.

### `GET_MASTER_PUBLIC_KEY=1`
Cette option permet à l'esclave de récupérer automatiquement la clé publique RSA du maître pour chiffrer le mot de passe lors de la première connexion. C'est un compromis entre sécurité et simplicité.

### Utilisateur dédié `replicator`
Le compte utilisé pour la réplication ne possède que les droits `REPLICATION SLAVE` — rien d'autre. Même si ce compte était compromis, il ne pourrait pas lire ni modifier les données. 🛡️
