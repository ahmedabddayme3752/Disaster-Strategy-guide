# 08 — Annuaire des Utilisateurs & Accès
## Tous les Comptes Nécessaires pour Administrer le Système DR BNM

> [!CAUTION]
> Ce document est **STRICTEMENT CONFIDENTIEL**. Il ne doit être accessible qu'aux membres
> autorisés de la Direction des Systèmes d'Information (DSI) de la BNM.
> Ne jamais partager par email non chiffré ou messagerie non sécurisée.

---

## 🗂️ Catégories d'Utilisateurs

Le système DR BNM implique **4 catégories** d'utilisateurs :

| Catégorie | Description |
|:---|:---|
| **Utilisateurs OS (Système)** | Comptes Linux sur chaque serveur |
| **Utilisateurs MySQL** | Comptes dans la base de données |
| **Utilisateurs Grafana** | Comptes pour le Dashboard de monitoring |
| **Utilisateurs Application** | Comptes spécifiques pour les applications métier |

---

## 🖥️ CATÉGORIE 1 — Utilisateurs Système (Linux OS)

Ces comptes permettent de se connecter par SSH aux serveurs.

| Serveur | Zone | IP | Utilisateur OS | Accès | Usage |
|:---|:---:|:---|:---|:---|:---|
| Production Click | Zone 0 | `192.168.1.241` | `connect` | SSH | Administration production |
| Monitoring | Zone 1 | `192.168.1.65` | `monotordr` | SSH | Administration monitoring |
| DR Primaire | Zone 2 | `192.168.1.66` | `primbackupdr` | SSH | Administration DR principal |
| DR Secondaire | Zone 3 | `192.168.1.64` | `secbackupdr` | SSH | Administration DR secondaire |

### Connexion SSH :
```bash
# Zone 0 - Production
ssh connect@192.168.1.241

# Zone 1 - Monitoring
ssh monotordr@192.168.1.65

# Zone 2 - DR Primaire
ssh primbackupdr@192.168.1.66

# Zone 3 - DR Secondaire
ssh secbackupdr@192.168.1.64
```

> [!NOTE]
> Les mots de passe SSH ne sont pas listés ici par sécurité. Ils sont gérés par l'administrateur système.

---

## 🗄️ CATÉGORIE 2 — Utilisateurs MySQL

### 2.1 — Zone 0 : Production Click (192.168.1.241 — MySQL 8.0, Port 3366)

| Utilisateur | Hôte | Mot de passe | Plugin Auth | Droits | Usage |
|:---|:---|:---|:---|:---|:---|
| `root` | `localhost` | *via sudo* | `auth_socket` | ALL | Admin complet (connexion locale uniquement) |
| `debian-sys-maint` | `localhost` | *voir `/etc/mysql/debian.cnf`* | `mysql_native_password` | ALL | Maintenance système Ubuntu |
| `nio` | `%` | *voir app config* | `caching_sha2_password` | Applicatif | Utilisateur de l'application Click |
| `replicator` | `%` | `ReplicaPass2026!` | `mysql_native_password` | REPLICATION SLAVE | Réplication vers Zone 2 & 3 |
| `monitor` | `127.0.0.1` | `MonitorPass2026!` | `mysql_native_password` | PROCESS, REPLICATION CLIENT, SELECT | Lecture métriques (mysqld-exporter) |

### Créer/Vérifier les utilisateurs DR sur Zone 0 :
```bash
set +H
# Connexion via debian-sys-maint (toujours disponible)
mysql -u debian-sys-maint -p'MOT_DE_PASSE_MAINTENANCE' -P 3366 -e "
-- Voir tous les utilisateurs
SELECT user, host, plugin FROM mysql.user;

-- Vérifier que replicator existe
SELECT user, host FROM mysql.user WHERE user='replicator';

-- Vérifier que monitor existe
SELECT user, host FROM mysql.user WHERE user='monitor';
"
```

### Trouver le mot de passe `debian-sys-maint` :
```bash
sudo cat /etc/mysql/debian.cnf
# Chercher la ligne : password = XXXXXXXXXXXX
```

---

### 2.2 — Zone 2 : DR Primaire (192.168.1.66 — MySQL 8.0, Port 3306)

| Utilisateur | Hôte | Mot de passe | Plugin Auth | Droits | Usage |
|:---|:---|:---|:---|:---|:---|
| `root` | `%` | `RootPassword123!` | `mysql_native_password` | ALL | Admin complet Docker |
| `replicator` | Hérité | — | — | — | Utilisé côté maître uniquement |

### Connexion rapide :
```bash
# Depuis la machine Zone 2
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!'

# Avec commande directe
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "SHOW DATABASES;"
```

---

### 2.3 — Zone 3 : DR Secondaire (192.168.1.64 — MySQL 8.0, Port 3306)

| Utilisateur | Hôte | Mot de passe | Plugin Auth | Droits | Usage |
|:---|:---|:---|:---|:---|:---|
| `root` | `%` | `RootPassword123!` | `mysql_native_password` | ALL | Admin complet Docker |

### Connexion rapide :
```bash
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!'
```

---

### 2.4 — Pleniged (Zones 2 & 3 — MySQL 5.7, Port 3307)

| Utilisateur | Hôte | Mot de passe | Plugin Auth | Droits | Usage |
|:---|:---|:---|:---|:---|:---|
| `root` | `%` | `RootPassword123!` | `mysql_native_password` | ALL | Admin complet Docker |

### Connexion rapide :
```bash
sudo docker exec -it mysql-pleniged mysql -u root -p'RootPassword123!'
```

---

### 2.5 — Utilisateurs MySQL dédié Monitoring (sur les Zones DR)

Ces fichiers `.my.cnf` sont utilisés par les conteneurs `mysqld-exporter` pour se connecter **sans mot de passe en clair dans la commande**.

**Zone 2 — `~/mysql-dr/monitoring/click.my.cnf` :**
```ini
[client]
user=root
password=RootPassword123!
host=mysql-click
port=3306
```

**Zone 2 — `~/mysql-dr/monitoring/pleniged.my.cnf` :**
```ini
[client]
user=root
password=RootPassword123!
host=mysql-pleniged
port=3306
```

**Zone 0 — `~/monitoring-prod/monitor.my.cnf` :**
```ini
[client]
user=monitor
password=MonitorPass2026!
host=127.0.0.1
port=3366
```

---

## 📊 CATÉGORIE 3 — Utilisateurs Grafana (192.168.1.65:3000)

| Utilisateur | Mot de passe | Rôle | Accès |
|:---|:---|:---|:---|
| `admin` | `GrafanaPass2026!` | Administrateur | Tout (créer dashboards, gérer alertes, utilisateurs) |

### Connexion :
```
URL      : http://192.168.1.65:3000
Login    : admin
Password : GrafanaPass2026!
```

### Dashboard BNM :
```
Nom : BNM — Disaster Recovery Dashboard v6
UID : bnm-dr-mysql
URL : http://192.168.1.65:3000/d/bnm-dr-mysql
```

### Créer un utilisateur "lecture seule" pour les auditeurs DSI :
1. **Administration** → **Users and access** → **Users** → **New User**
2. Rôle : **Viewer** (ne peut pas modifier, seulement regarder)
3. Exemple : `auditeur@bnm.mr` / `AuditPass2026!`

---

## 🔑 CATÉGORIE 4 — Tableau de Bord Récapitulatif

| Zone | Serveur | Connexion | Utilisateur | Mot de passe |
|:---|:---|:---|:---|:---|
| **Zone 0** | Linux SSH | `ssh connect@192.168.1.241` | `connect` | *Admin OS* |
| **Zone 0** | MySQL Click | `mysql -h 192.168.1.241 -P 3366` | `monitor` | `MonitorPass2026!` |
| **Zone 0** | MySQL Click | `mysql -h 192.168.1.241 -P 3366` | `replicator` | `ReplicaPass2026!` |
| **Zone 1** | Linux SSH | `ssh monotordr@192.168.1.65` | `monotordr` | *Admin OS* |
| **Zone 1** | Grafana | `http://192.168.1.65:3000` | `admin` | `GrafanaPass2026!` |
| **Zone 1** | Prometheus | `http://192.168.1.65:9090` | — | Pas d'auth (réseau interne) |
| **Zone 2** | Linux SSH | `ssh primbackupdr@192.168.1.66` | `primbackupdr` | *Admin OS* |
| **Zone 2** | MySQL Click DR | `docker exec mysql-click mysql -u root` | `root` | `RootPassword123!` |
| **Zone 2** | MySQL Pleniged DR | `docker exec mysql-pleniged mysql -u root` | `root` | `RootPassword123!` |
| **Zone 3** | Linux SSH | `ssh secbackupdr@192.168.1.64` | `secbackupdr` | *Admin OS* |
| **Zone 3** | MySQL Click DR | `docker exec mysql-click mysql -u root` | `root` | `RootPassword123!` |
| **Zone 3** | MySQL Pleniged DR | `docker exec mysql-pleniged mysql -u root` | `root` | `RootPassword123!` |

---

## 🔐 Procédure de Rotation des Mots de Passe (Recommandation Sécurité)

> [!TIP]
> Il est recommandé de changer les mots de passe **tous les 90 jours**. Voici la procédure.

### Changer le mot de passe `replicator` :
```sql
-- 1. Sur la Production (Zone 0)
ALTER USER 'replicator'@'%' IDENTIFIED BY 'NouveauMotDePasse2026!';
FLUSH PRIVILEGES;

-- 2. Sur chaque esclave (Zone 2 & 3) — Mettre à jour la connexion de réplication
STOP SLAVE;
CHANGE MASTER TO MASTER_PASSWORD='NouveauMotDePasse2026!';
START SLAVE;
```

### Changer le mot de passe Grafana :
1. Grafana → **Profil utilisateur** (icône en bas à gauche) → **Change Password**
2. Ou via ligne de commande : `docker exec grafana grafana-cli admin reset-admin-password NouveauMotDePasse2026!`
