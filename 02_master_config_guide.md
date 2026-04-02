# 02 — Guide de Configuration Maître (Production)

## 🏦 Rôle : Source de Vérité (Zone 1)
Ce document détaille comment préparer les serveurs de production pour qu'ils acceptent les connexions des sites de secours (DR).

---

## 🛠️ SERVER 1 — Click (Ubuntu Linux, .241)
C'est le serveur critique pour le paiement mobile.

### 1 — Configuration du Service MySQL
Éditez le fichier `/etc/mysql/mysql.conf.d/mysqld.cnf` :

```ini
[mysqld]
# 🏁 Paramètres Réseau
port                    = 3366              # Port utilisé pour la Prod Click
bind-address            = 0.0.0.0           # Autorise les connexions réseau

# 🏁 Paramètres Réplication (Crucial !)
server-id               = 10                # Identifiant unique de la Prod
log_bin                 = /var/log/mysql/mysql-bin.log
binlog_format           = ROW               # Pour la fiabilité des données
gtid_mode               = ON                # Active les GTID
enforce_gtid_consistency = ON               # Garantie l'intégrité GTID
```

### 2 — Création de l'Utilisateur de Réplication
Connectez-vous à MySQL avec le compte de maintenance (`debian-sys-maint`) :

```bash
mysql -u debian-sys-maint -p'VOTRE_MOT_DE_PASSE_MAINTENANCE' -e "
CREATE USER 'replicator'@'%' IDENTIFIED WITH mysql_native_password BY 'ReplicaPass2026!';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;
"
```

---

## 🛠️ SERVER 2 — Pleniged (Windows 10, .34)
C'est le serveur documentaire utilisant MySQL 8.0 natif.

### 1 — Configuration `my.ini`
Trouvez le fichier `C:\ProgramData\MySQL\MySQL Server 8.0\my.ini` :

```ini
[mysqld]
server-id               = 20
log_bin                 = mysql-bin
binlog_format           = ROW
gtid_mode               = ON
enforce_gtid_consistency = ON
```

### 2 — Ouverture du Pare-feu (PowerShell Admin)
```powershell
New-NetFirewallRule -DisplayName "MySQL DR Replication" -Direction Inbound -Protocol TCP -LocalPort 3306 -Action Allow
```

---

## 📦 Procédure d'Exportation Initiale (Dump)
Pour envoyer les données existantes sur le DR sans arrêter la Prod :

### Sur Ubuntu (Click)
```bash
mysqldump -u nio -p --databases clickTest --single-transaction --routines --triggers --set-gtid-purged=ON --no-tablespaces > clickTest_dump.sql
```

### Sur Windows (Pleniged)
```powershell
& "C:\Program Files\MySQL\MySQL Server 8.0\bin\mysqldump.exe" -u root -p --databases pleniged_test --single-transaction --routines --triggers --set-gtid-purged=ON --no-tablespaces --result-file="C:\mysql-dr\pleniged_dump.sql"
```

> [!TIP]
> Envoyez ces fichiers `_dump.sql` à Ahmed pour l'importation sur les conteneurs DR. ✨🏦
