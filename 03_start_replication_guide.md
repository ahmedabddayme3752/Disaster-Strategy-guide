# 03 — Mise en Service de la Réplication DR

## 🎯 Rôle de ce Guide
Ce guide décrit comment déployer et démarrer la réplication MySQL sur les serveurs de secours (Zone 2 et Zone 3).

---

## 🐳 PARTIE 1 : Configuration Docker Multi-Instance

### Le fichier `docker-compose.yml`
Ce fichier est le "plan" qui décrit tous les conteneurs à faire tourner sur chaque site DR.

**Fichier complet annoté (`~/mysql-dr/docker-compose.yml`) :**

```yaml
version: '3.3'

# ============================================================
# BNM — Banque Nationale de Mauritanie
# Disaster Recovery — Multi-Instance MySQL
# Zone 2 (primbackupdr) — 192.168.1.66
# Zone 3 (secbackupdr)  — 192.168.1.64
# ============================================================

services:

  # ────────────────────────────────────────────────
  # INSTANCE 1 : Click (Paiement Mobile)
  # Source Prod : 192.168.1.241:3366
  # ────────────────────────────────────────────────
  mysql-click:
    image: mysql:8.0
    command: >
      --slave-skip-errors=1062,1396
      --replicate-ignore-db=click
    # --slave-skip-errors=1062,1396 :
    #   - 1062 = Ignorer les erreurs de doublons (données déjà présentes)
    #   - 1396 = Ignorer les erreurs de création d'utilisateurs déjà existants
    # --replicate-ignore-db=click :
    #   - Ignorer la base "click" (base de test sur la Prod, pas la base principale)
    container_name: mysql-click
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: "RootPassword123!"
    ports:
      - "3306:3306"   # Port externe:Port interne
    volumes:
      - ./click/data:/var/lib/mysql          # Données MySQL persistantes
      - ./click/config/my.cnf:/etc/mysql/my.cnf  # Configuration GTID

  # ────────────────────────────────────────────────
  # INSTANCE 2 : Pleniged (Gestion Documentaire)
  # Source Prod : 10.168.2.34:3306 (Windows)
  # ────────────────────────────────────────────────
  mysql-pleniged:
    image: mysql:8.0
    container_name: mysql-pleniged
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: "RootPassword123!"
    ports:
      - "3307:3306"   # Port 3307 pour ne pas entrer en conflit avec Click
    volumes:
      - ./pleniged/data:/var/lib/mysql
      - ./pleniged/config/my.cnf:/etc/mysql/my.cnf

  # ────────────────────────────────────────────────
  # CAPTEUR 1 : Monitoring Click → Prometheus
  # Métriques disponibles sur le port 9104
  # ────────────────────────────────────────────────
  mysql-exporter-click:
    image: prom/mysqld-exporter
    container_name: mysql-exporter-click
    command:
      - "--config.my-cnf=/cfg/.my.cnf"
      # Le fichier .my.cnf contient les credentials MySQL
      # C'est plus sécurisé que de les mettre dans les variables d'environnement
    volumes:
      - ./monitoring/click.my.cnf:/cfg/.my.cnf:ro  # :ro = lecture seule
    ports:
      - "9104:9104"   # Prometheus collecte les métriques Click ici
    depends_on:
      - mysql-click
    restart: always

  # ────────────────────────────────────────────────
  # CAPTEUR 2 : Monitoring Pleniged → Prometheus
  # Métriques disponibles sur le port 9105
  # ────────────────────────────────────────────────
  mysql-exporter-pleniged:
    image: prom/mysqld-exporter
    container_name: mysql-exporter-pleniged
    command:
      - "--config.my-cnf=/cfg/.my.cnf"
    volumes:
      - ./monitoring/pleniged.my.cnf:/cfg/.my.cnf:ro
    ports:
      - "9105:9104"   # Port 9105 pour Pleniged (9104 déjà utilisé par Click)
    depends_on:
      - mysql-pleniged
    restart: always
```

---

### Les fichiers de configuration MySQL (`my.cnf`)

**Pour Click (`./click/config/my.cnf`) :**
```ini
[mysqld]
# Identifiant unique de l'esclave Click sur Zone 2
# Zone 2 Click = 66, Zone 3 Click = 67
server-id               = 66
log_bin                 = /var/lib/mysql/mysql-bin.log
binlog_format           = ROW
gtid_mode               = ON
enforce_gtid_consistency = ON
```

**Pour Pleniged (`./pleniged/config/my.cnf`) :**
```ini
[mysqld]
# Identifiant unique de l'esclave Pleniged sur Zone 2
# Zone 2 Pleniged = 77, Zone 3 Pleniged = 78
server-id               = 77
log_bin                 = /var/lib/mysql/mysql-bin.log
binlog_format           = ROW
gtid_mode               = ON
enforce_gtid_consistency = ON
```

---

### Les fichiers de credentials Monitoring

**Pour Click (`./monitoring/click.my.cnf`) :**
```ini
[client]
user=root
password=RootPassword123!
host=mysql-click    # Nom du conteneur Docker (réseau interne)
port=3306           # Port interne du conteneur
```

**Pour Pleniged (`./monitoring/pleniged.my.cnf`) :**
```ini
[client]
user=root
password=RootPassword123!
host=mysql-pleniged
port=3306
```

> [!NOTE]
> Ces fichiers doivent avoir les permissions `644` pour être lisibles par les conteneurs :
> ```bash
> chmod 644 ~/mysql-dr/monitoring/*.my.cnf
> ```

---

## 🪄 PARTIE 2 : Le "Magic Sync" — Démarrage de la Réplication

### Prérequis
Avant de lancer la réplication, vous devez avoir :
1. ✅ Importé le Dump SQL sur l'instance esclave
2. ✅ Noté le GTID de départ (ligne `GTID_PURGED` dans le dump)

### Étape 1 : Préparer Bash (Important !)
```bash
# Désactiver l'expansion de l'historique Bash
# Sans cette commande, le "!" dans les mots de passe cause une erreur
set +H
```

### Étape 2 : Le Bloc "Magic Sync" pour Click

```bash
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "

-- ═══════════════════════════════════════════════
-- ÉTAPE 1 : Nettoyage complet des métadonnées
-- ═══════════════════════════════════════════════
STOP SLAVE;
-- Supprime toutes les configurations d'esclave précédentes
RESET SLAVE ALL;
-- Vide le journal binaire local pour repartir proprement
RESET MASTER;

-- ═══════════════════════════════════════════════
-- ÉTAPE 2 : Positionnement GTID
-- Indique au serveur qu'il possède déjà les transactions 1 à N
-- N = le chiffre trouvé dans la ligne GTID_PURGED du dump SQL
-- ═══════════════════════════════════════════════
SET GLOBAL GTID_PURGED = '816c0b5e-97e8-11ed-a73b-000c299194d8:1-215';

-- ═══════════════════════════════════════════════
-- ÉTAPE 3 : Connexion au Maître
-- ═══════════════════════════════════════════════
CHANGE MASTER TO
  MASTER_HOST='192.168.1.241',    -- IP du serveur Click Production
  MASTER_PORT=3366,               -- Port réel de la Production Click
  MASTER_USER='replicator',       -- Utilisateur dédié à la réplication
  MASTER_PASSWORD='ReplicaPass2026!',
  MASTER_AUTO_POSITION=1,         -- Utilise les GTID pour la synchronisation
  GET_MASTER_PUBLIC_KEY=1;        -- Récupère la clé RSA pour le chiffrement du mdp

-- ═══════════════════════════════════════════════
-- ÉTAPE 4 : Démarrage !
-- ═══════════════════════════════════════════════
START SLAVE;
"
```

### Étape 3 : Vérification du Statut

```bash
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "SHOW SLAVE STATUS\G" \
  | grep -E "Slave_IO_Running|Slave_SQL_Running|Seconds_Behind_Master|Executed_Gtid_Set"
```

**Résultat attendu :**
```
Slave_IO_Running: Yes          ← Connecté au Maître ✅
Slave_SQL_Running: Yes         ← Applique les transactions ✅
Seconds_Behind_Master: 0       ← Synchronisé en temps réel ✅
Executed_Gtid_Set: ...:1-33527 ← Toutes les transactions reçues ✅
```

---

## 🔧 PARTIE 3 : Résolution des Erreurs Communes

Si `Slave_SQL_Running: No`, lancez cette commande pour voir l'erreur précise :
```bash
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e \
  "SELECT * FROM performance_schema.replication_applier_status_by_worker WHERE LAST_ERROR_NUMBER > 0\G"
```

Puis consultez le **Guide 05 — Troubleshooting** pour la solution correspondante.
