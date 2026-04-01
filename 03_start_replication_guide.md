# Guide 3 — Démarrage de la Réplication (Architecture Double-Instance)

Ce guide explique comment connecter vos serveurs DR (Zone 2 et Zone 3) aux sources de production.

---

## Étape 1 — Connexion de la Zone 2 (Primary DR) aux Sources

Sur **Zone 2** (`192.168.1.66`), nous allons connecter les deux conteneurs indépendamment.

### A. Connecter Click (Port 3306)
```bash
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "
-- Activer le plugin semi-sync côté esclave
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET PERSIST rpl_semi_sync_slave_enabled = 1;

-- Pointer vers la source Click
CHANGE MASTER TO
  MASTER_HOST='192.168.1.241',
  MASTER_USER='replicator',
  MASTER_PASSWORD='ReplicaPass2026!',
  MASTER_AUTO_POSITION=1,
  MASTER_SSL=0,
  GET_MASTER_PUBLIC_KEY=1;

START SLAVE;
SHOW SLAVE STATUS\G
"
```

### B. Connecter Pleniged (Port 3307)
```bash
sudo docker exec -it mysql-pleniged mysql -u root -pRootPassword123! -e "
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET PERSIST rpl_semi_sync_slave_enabled = 1;

-- Pointer vers la source Pleniged
CHANGE MASTER TO
  MASTER_HOST='10.168.2.34',
  MASTER_USER='replicator',
  MASTER_PASSWORD='ReplicaPass2026!',
  MASTER_AUTO_POSITION=1,
  MASTER_SSL=0,
  GET_MASTER_PUBLIC_KEY=1;

START SLAVE;
SHOW SLAVE STATUS\G
"
```

---

## Étape 2 — Connexion de la Zone 3 (Standby DR) en Parallèle

Dans cette version de l'architecture, la Zone 3 se connecte **directement** aux sources (comme la Zone 2), mais en mode **Asynchrone** (pour ne pas ralentir la production).

### A. Connecter Click (Port 3306)
```bash
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "
-- 1. On ne met PAS de semi-sync ici (Async pur)
SET PERSIST rpl_semi_sync_slave_enabled = 0;

-- 2. Pointer vers la source Click (241)
CHANGE MASTER TO
  MASTER_HOST='192.168.1.241',
  MASTER_USER='replicator',
  MASTER_PASSWORD='ReplicaPass2026!',
  MASTER_AUTO_POSITION=1,
  MASTER_SSL=0,
  GET_MASTER_PUBLIC_KEY=1;

START SLAVE;
SHOW SLAVE STATUS\G
"
```

### B. Connecter Pleniged (Port 3307)
```bash
sudo docker exec -it mysql-pleniged mysql -u root -pRootPassword123! -e "
-- 1. Async pur
SET PERSIST rpl_semi_sync_slave_enabled = 0;

-- 2. Pointer vers la source Pleniged (34)
CHANGE MASTER TO
  MASTER_HOST='10.168.2.34',
  MASTER_USER='replicator',
  MASTER_PASSWORD='ReplicaPass2026!',
  MASTER_AUTO_POSITION=1,
  MASTER_SSL=0,
  GET_MASTER_PUBLIC_KEY=1;

START SLAVE;
SHOW SLAVE STATUS\G
"
```

---

## ✅ Résultat Final Attendu

| VM | Conteneur | Port | Source | Type de réplication |
| :--- | :--- | :--- | :--- | :--- |
| **Zone 2** | `mysql-click` | 3306 | Click (241) | **Semi-Synchrone** (Hot Standby) |
| **Zone 2** | `mysql-pleniged` | 3307 | Pleniged (34) | **Semi-Synchrone** (Hot Standby) |
| **Zone 3** | `mysql-click` | 3306 | Click (241) | **Asynchrone** (Warm Standby) |
| **Zone 3** | `mysql-pleniged` | 3307 | Pleniged (34) | **Asynchrone** (Warm Standby) |
