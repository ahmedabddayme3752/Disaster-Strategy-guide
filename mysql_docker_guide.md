# Guide d'Implémentation MySQL DR avec Docker

## Architecture des Zones

| Zone | Serveur | IP | Rôle |
|------|---------|-----|------|
| **Zone 1** | `monotordr` | `192.168.1.65` | 🔍 **Monitoring uniquement** — Prometheus, Grafana, Alertes |
| **Zone 2** | `primbackupdr` | `192.168.1.66` | 🟢 **MySQL Primary** — Production (même ville) |
| **Zone 3** | `secbackupdr` | `192.168.1.64` | 🟡 **MySQL Standby** — Secondaire (même ville, serveur différent) |
| **Zone 4** | TBD | TBD | 🔴 **Remote DR** — Géo-séparé (à utiliser plus tard) |

> ⚠️ **Important :** Zone 1 ne reçoit PAS MySQL. C'est uniquement le serveur de monitoring. MySQL est installé sur Zone 2 (Primary) et Zone 3 (Standby).

---

## Étape 0 — Prérequis communs (Zone 2 et Zone 3 uniquement)

Sur **Zone 2 (`primbackupdr`, 192.168.1.66`)** ET **Zone 3 (`secbackupdr`, `192.168.1.64`)** :

```bash
# Installer Docker et Docker Compose
sudo apt update
sudo apt install -y docker.io docker-compose

# Créer l'arborescence pour MySQL
mkdir -p ~/mysql-dr/{data,config}
cd ~/mysql-dr
```

Créez également le fichier `docker-compose.yml` sur les deux serveurs :
```yaml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    container_name: mysql-server
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: "RootPassword123!"
    ports:
      - "3306:3306"
    volumes:
      - ./data:/var/lib/mysql
      - ./config/my.cnf:/etc/mysql/my.cnf
```

---

## Étape 1 — Zone 2 (Primary) : `primbackupdr@192.168.1.66`

### A. Fichier de configuration `~/mysql-dr/config/my.cnf`

```ini
[mysqld]
server-id = 2
log_bin = /var/lib/mysql/mysql-bin.log
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
```

> **Note :** Les plugins `semisync` sont installés dynamiquement via SQL après le démarrage (voir étape C).

### B. Démarrer le conteneur

```bash
cd ~/mysql-dr
sudo docker-compose up -d

# Attendez ~30 secondes que MySQL s'initialise
sleep 30
sudo docker ps   # Vérifiez que le statut est "Up" et non "Restarting"
```

### C. Créer l'utilisateur de réplication + activer le plugin semi-sync

```bash
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "
-- Créer l'utilisateur de réplication
CREATE USER 'replicator'@'%' IDENTIFIED BY 'ReplicaPass2026!';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';

-- Activer la réplication semi-synchrone coté Primary
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET PERSIST rpl_semi_sync_master_enabled = 1;
SET PERSIST rpl_semi_sync_master_timeout = 10000;

FLUSH PRIVILEGES;
SHOW MASTER STATUS;
"
```

---

## Étape 2 — Zone 3 (Standby) : `secbackupdr@192.168.1.64`

### A. Fichier de configuration `~/mysql-dr/config/my.cnf`

```ini
[mysqld]
server-id = 3
relay-log = /var/lib/mysql/relay-bin.log
read_only = ON
gtid_mode = ON
enforce_gtid_consistency = ON
```

### B. Démarrer le conteneur

```bash
cd ~/mysql-dr
sudo docker-compose up -d

# Attendez ~30 secondes
sleep 30
sudo docker ps   # Vérifiez que le statut est "Up"
```

### C. Activer le plugin semi-sync + démarrer la réplication vers Zone 2

```bash
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "
-- Activer la réplication semi-synchrone coté Standby
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET PERSIST rpl_semi_sync_slave_enabled = 1;

-- Pointer vers le Primary (Zone 2)
CHANGE MASTER TO
  MASTER_HOST='192.168.1.66',
  MASTER_USER='replicator',
  MASTER_PASSWORD='ReplicaPass2026!',
  MASTER_AUTO_POSITION=1,
  MASTER_SSL=0,
  GET_MASTER_PUBLIC_KEY=1;

START SLAVE;
SHOW SLAVE STATUS\G
"
```

> ✅ **Vérifiez que vous voyez :**
> - `Slave_IO_Running: Yes`
> - `Slave_SQL_Running: Yes`
> - `Seconds_Behind_Master: 0`

---

## Étape 3 — Zone 1 (Monitoring) : `monotordr@192.168.1.65`

Zone 1 n'accueille **pas** MySQL. Elle est dédiée au monitoring de la réplication sur Zone 2 et Zone 3.

> ➡️ **Veuillez consulter le fichier `05_monitoring_guide.md`** pour installer Prometheus et Grafana proprement via Docker sur cette zone. N'utilisez pas `apt install prometheus`, qui peut causer des conflits d'utilisateurs.

---

## Zone 4 — Remote DR (À venir)

Zone 4 sera configurée ultérieurement comme site de DR géo-séparé. La réplication sera de type **asynchrone** depuis la Zone 2 (Primary) ou Zone 3 (Standby). La configuration sera ajoutée à ce guide dès que l'IP et les accès sont disponibles.

---

## Résumé des flux de réplication

```
Zone 2 (Primary, 192.168.1.66)
    |
    | Semi-Sync Replication (TCP 3306)
    |
    v
Zone 3 (Standby, 192.168.1.64)

Zone 1 (Monitoring, 192.168.1.65):
    Surveille Zone 2 et Zone 3 via mysqld_exporter (port 9104)

Zone 4 (Remote DR): À configurer — réplication Async depuis Zone 2
```
