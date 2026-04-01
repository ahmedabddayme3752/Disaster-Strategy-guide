# Guide d'Implémentation MySQL DR avec Docker (Double-Conteneur)

## Architecture des Zones

| Zone | Serveur | IP | Rôle |
|------|---------|-----|------|
| **Zone 1** | `monotordr` | `192.168.1.65` | 🔍 **Monitoring** — Gère la surveillance de Click et Pleniged |
| **Zone 2** | `primbackupdr` | `192.168.1.66` | 🟢 **Primary DR** — Contient Click (:3306) et Pleniged (:3307) |
| **Zone 3** | `secbackupdr` | `192.168.1.64` | 🟡 **Standby DR** — Contient Click (:3306) et Pleniged (:3307) |

---

## Étape 0 — Nettoyage et Prérequis (Zone 2 et Zone 3)

Exécutez ceci sur les serveurs **Zone 2** ET **Zone 3** :

```bash
# 1. Arrêter l'ancienne configuration
cd ~/mysql-dr
sudo docker-compose down

# 2. Créer la nouvelle structure de dossiers isolée
mkdir -p ~/mysql-dr/click/{data,config}
mkdir -p ~/mysql-dr/pleniged/{data,config}

# 3. Créer le nouveau docker-compose.yml (Double Instance)
cat <<EOF > ~/mysql-dr/docker-compose.yml
version: '3.3'
services:
  mysql-click:
    image: mysql:8.0
    container_name: mysql-click
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: "RootPassword123!"
    ports:
      - "3306:3306"
    volumes:
      - ./click/data:/var/lib/mysql
      - ./click/config/my.cnf:/etc/mysql/my.cnf

  mysql-pleniged:
    image: mysql:8.0
    container_name: mysql-pleniged
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: "RootPassword123!"
    ports:
      - "3307:3306"
    volumes:
      - ./pleniged/data:/var/lib/mysql
      - ./pleniged/config/my.cnf:/etc/mysql/my.cnf
EOF
```

---

## Étape 1 — Configuration Zone 2 (Primary DR)

### A. Fichiers de configuration `my.cnf`

**Pour Click (`~/mysql-dr/click/config/my.cnf`)** :
```ini
[mysqld]
server-id = 21
log_bin = /var/lib/mysql/mysql-bin.log
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
```

**Pour Pleniged (`~/mysql-dr/pleniged/config/my.cnf`)** :
```ini
[mysqld]
server-id = 22
log_bin = /var/lib/mysql/mysql-bin.log
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
```

### B. Démarrer les deux conteneurs
```bash
cd ~/mysql-dr
sudo docker-compose up -d
sleep 30
sudo docker ps # Vous devez voir mysql-click (3306) et mysql-pleniged (3307)
```

### C. Initialisation des plugins (Semi-Sync)
```bash
# Pour Click
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "
CREATE USER 'replicator'@'%' IDENTIFIED BY 'ReplicaPass2026!';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'replicator'@'%';
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET PERSIST rpl_semi_sync_master_enabled = 1;
"

# Pour Pleniged
sudo docker exec -it mysql-pleniged mysql -u root -pRootPassword123! -e "
CREATE USER 'replicator'@'%' IDENTIFIED BY 'ReplicaPass2026!';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'replicator'@'%';
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET PERSIST rpl_semi_sync_master_enabled = 1;
"
```

---

## Étape 2 — Configuration Zone 3 (Standby DR)

### A. Fichiers de configuration `my.cnf`

**Pour Click (`~/mysql-dr/click/config/my.cnf`)** :
```ini
[mysqld]
server-id = 31
relay-log = /var/lib/mysql/relay-bin.log
read_only = ON
gtid_mode = ON
enforce_gtid_consistency = ON
```

**Pour Pleniged (`~/mysql-dr/pleniged/config/my.cnf`)** :
```ini
[mysqld]
server-id = 32
relay-log = /var/lib/mysql/relay-bin.log
read_only = ON
gtid_mode = ON
enforce_gtid_consistency = ON
```

### B. Démarrer les deux conteneurs
```bash
cd ~/mysql-dr
sudo docker-compose up -d
sleep 30
```

---

## Étape 3 — Suite du Projet

Dès que les conteneurs sont lancés, passez au guide **`03_start_replication_guide.md`** pour connecter les sources.
