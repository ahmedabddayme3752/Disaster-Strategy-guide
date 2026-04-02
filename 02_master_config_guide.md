# 02 — Configuration des Serveurs Maîtres (Production)

## 🎯 Rôle de ce Guide
Ce guide détaille la configuration requise sur les serveurs de **production** pour qu'ils puissent envoyer leurs données aux sites de secours (DR).

---

## 🖥️ SERVER 1 — Click (Ubuntu Linux 22.04)
**IP :** `192.168.1.241` | **Port MySQL :** `3366` | **Utilisateur Applicatif :** `nio`

### 1.1 — Fichier de Configuration MySQL

**Chemin :** `/etc/mysql/mysql.conf.d/mysqld.cnf`

```ini
[mysqld]
# ─────────────────────────────────────────────
# RÉSEAU
# ─────────────────────────────────────────────
# Port 3366 est le port RÉEL de la production Click
# (pas le 3306 standard — configuration historique de la BNM)
port                    = 3366

# Autorise les connexions depuis n'importe quelle IP
# (nécessaire pour que Zone 2 et Zone 3 puissent se connecter)
bind-address            = 0.0.0.0

# ─────────────────────────────────────────────
# RÉPLICATION GTID
# ─────────────────────────────────────────────
# Identifiant unique de CE serveur sur tout le réseau BNM
server-id               = 10

# Active le journal binaire (indispensable pour la réplication)
log_bin                 = /var/log/mysql/mysql-bin.log

# Format ROW = chaque modification de ligne est journalisée
# (plus fiable que STATEMENT pour la réplication)
binlog_format           = ROW

# Active les GTID (Global Transaction Identifiers)
gtid_mode               = ON

# Force la cohérence des GTID (empêche les transactions non-GTID)
enforce_gtid_consistency = ON
```

### 1.2 — Compte de Maintenance Système
Sur Ubuntu, le compte `debian-sys-maint` est le seul qui peut créer des utilisateurs quand le mot de passe root MySQL est inconnu. Son mot de passe se trouve dans :
```bash
sudo cat /etc/mysql/debian.cnf
# Chercher la ligne : password = XXXXXXXXXXXXXXXX
```

### 1.3 — Création de l'Utilisateur de Réplication
```bash
# Connexion avec le compte de maintenance
mysql -u debian-sys-maint -p'MOT_DE_PASSE_DEBIAN' -e "
-- Supprimer l'ancien compte si il existe
DROP USER IF EXISTS 'replicator'@'%';

-- Créer avec le plugin natif (compatible sans SSL)
CREATE USER 'replicator'@'%'
  IDENTIFIED WITH mysql_native_password
  BY 'ReplicaPass2026!';

-- Droits minimaux (lecture seule des logs de réplication)
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';

-- Appliquer les changements
FLUSH PRIVILEGES;
"
```

### 1.4 — Ouverture du Pare-feu (UFW)
```bash
# Autoriser les Zones DR à se connecter sur le port 3366
sudo ufw allow 3366/tcp
sudo ufw reload

# Vérifier que MySQL écoute bien sur 3366
sudo ss -tulpn | grep mysqld
# Résultat attendu : 0.0.0.0:3366 ... (mysqld)
```

### 1.5 — Exportation des Données (Dump Initial)
Pour initialiser les esclaves DR avec les données existantes :
```bash
# Exporter la base clickTest avec les GTID inclus
mysqldump -u nio -p \
  --databases clickTest \
  --single-transaction \
  --routines \
  --triggers \
  --set-gtid-purged=ON \
  --no-tablespaces \
  > clickTest_dump.sql

# La ligne cruciale dans ce fichier :
grep "GTID_PURGED" clickTest_dump.sql
# Exemple : SET @@GLOBAL.GTID_PURGED='816c0b5e-...:1-215';
# Ce chiffre "215" est le point de départ pour les esclaves
```

---

## 🖥️ SERVER 2 — Pleniged (Windows 10)
**IP :** `10.168.2.34` | **Port MySQL :** `3306`

### 2.1 — Fichier de Configuration MySQL

**Chemin :** `C:\ProgramData\MySQL\MySQL Server 8.0\my.ini`

```ini
[mysqld]
# Identifiant unique sur tout le réseau BNM
server-id               = 20

# Journal binaire pour la réplication
log_bin                 = mysql-bin
binlog_format           = ROW

# Activation GTID
gtid_mode               = ON
enforce_gtid_consistency = ON
```

### 2.2 — Ouverture du Pare-feu Windows (PowerShell Admin)
```powershell
# Autoriser Zone 2 et Zone 3 à se connecter
New-NetFirewallRule `
  -DisplayName "MySQL DR Replication BNM" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 3306 `
  -Action Allow
```

### 2.3 — Exportation des Données (Dump Initial)
```powershell
# Dans PowerShell
& "C:\Program Files\MySQL\MySQL Server 8.0\bin\mysqldump.exe" `
  -u root -p `
  --databases pleniged_test `
  --single-transaction `
  --routines `
  --triggers `
  --set-gtid-purged=ON `
  --no-tablespaces `
  --result-file="C:\mysql-dr\pleniged_dump.sql"
```

---

## 📊 Tableau de Référence — Paramètres de Production

| Paramètre | Click (.241) | Pleniged (.34) |
|:---|:---|:---|
| **IP** | `192.168.1.241` | `10.168.2.34` |
| **Port MySQL** | `3366` | `3306` |
| **Server-ID** | `10` | `20` |
| **UUID** | `816c0b5e-97e8-11ed-a73b-000c299194d8` | *(à récupérer)* |
| **Base principale** | `clickTest` | `pleniged_test` |
| **Utilisateur app** | `nio` | `root` |
| **Utilisateur DR** | `replicator` | `replicator` |
| **Mot de passe DR** | `ReplicaPass2026!` | `ReplicaPass2026!` |
