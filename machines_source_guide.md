# Guide — Machines Sources avec Docker (Click & Pleniged)

> 👤 **Ce guide est destiné à l'ami qui travaille sur les machines sources (Click et Pleniged).**
> On utilise Docker sur les deux machines pour avoir un environnement propre, identique et facilement gérable.

---

## Vue d'ensemble — Architecture complète

```
[Click Ubuntu — 192.168.1.241:3306]    ──Semi-Sync──▶  [Zone 2 DR — 192.168.1.66:3306]
[Pleniged Windows — 10.168.2.34:3306]  ──Semi-Sync──▶  [Zone 3 DR — 192.168.1.64:3306]

[Zone 1 — 192.168.1.65] : Monitoring Prometheus/Grafana (surveille tout)
```

---

## MACHINE 1 — Click (Ubuntu, 192.168.1.241)

### Étape 0 — Vérifier si Docker est déjà installé

```bash
# Connexion SSH
ssh <user>@192.168.1.241

# Vérifier si Docker est déjà présent
docker --version && docker-compose --version
```

> **Si vous voyez les numéros de version** (ex: `Docker version 24.x.x`) → Docker est déjà installé, **passez directement à l'Étape 1**.
>
> **Si vous voyez `command not found`** → Installez Docker maintenant :

```bash
# Installer Docker et Docker Compose
sudo apt update
sudo apt install -y docker.io docker-compose

# Vérifier après installation
docker --version
docker-compose --version

# Ajouter votre utilisateur au groupe docker (pour éviter sudo)
sudo usermod -aG docker $USER
newgrp docker
```

```bash
# Créer l'arborescence pour MySQL
mkdir -p ~/mysql-dr/{data,config}
cd ~/mysql-dr
```

### Étape 1 — Créer le `docker-compose.yml`

```bash
nano ~/mysql-dr/docker-compose.yml
```

```yaml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    container_name: mysql-click
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: "RootPassword123!"
      MYSQL_DATABASE: "click_test"
    ports:
      - "3306:3306"
    volumes:
      - ./data:/var/lib/mysql
      - ./config/my.cnf:/etc/mysql/my.cnf
```

> `MYSQL_DATABASE: "click_test"` — Docker crée automatiquement cette base de test au premier démarrage.

### Étape 2 — Créer le fichier `~/mysql-dr/config/my.cnf`

```bash
nano ~/mysql-dr/config/my.cnf
```

```ini
[mysqld]
server-id            = 10
log_bin              = /var/lib/mysql/mysql-bin.log
binlog_format        = ROW
gtid_mode            = ON
enforce_gtid_consistency = ON
```

> ⚠️ `server-id = 10` — must be unique across the entire network.
> Les plugins semi-sync seront activés **après** le démarrage via SQL.

### Étape 3 — Démarrer le conteneur

```bash
cd ~/mysql-dr
sudo docker-compose up -d

# Attendre ~30 secondes
sleep 30
sudo docker ps   # Vérifier que le statut est "Up" et NON "Restarting"
```

### Étape 4 — Créer l'utilisateur de réplication + Semi-Sync

```bash
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "
-- Utilisateur de réplication
CREATE USER 'replicator'@'%' IDENTIFIED BY 'ReplicaPass2026!';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';

-- Activer Semi-Sync Primary
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET PERSIST rpl_semi_sync_master_enabled = 1;
SET PERSIST rpl_semi_sync_master_timeout = 10000;

FLUSH PRIVILEGES;
SHOW MASTER STATUS;
"
```

> 📋 **Notez les valeurs** affichées par `SHOW MASTER STATUS` (`File` et `Position`) — à transmettre à Ahmed.

### Étape 5 — Ouvrir le port 3306 (si UFW actif)

```bash
sudo ufw allow from 192.168.1.66 to any port 3306 comment "MySQL DR Zone2"
sudo ufw allow from 192.168.1.65 to any port 9104 comment "Prometheus monitoring"
sudo ufw status
```

### Étape 6 — Insérer des données de test (pour valider la réplication)

```bash
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "
USE click_test;
CREATE TABLE IF NOT EXISTS dr_test (
  id INT AUTO_INCREMENT PRIMARY KEY,
  message VARCHAR(100),
  created_at DATETIME DEFAULT NOW()
);
INSERT INTO dr_test (message) VALUES ('Test initial Click DR - $(date)');
SELECT * FROM dr_test;
"
```

---

## MACHINE 2 — Pleniged (Windows 10, 10.168.2.34)

> Sur Windows 10, on utilise **Docker Desktop** avec le backend **WSL2**.

### Étape 0 — Vérifier si Docker est déjà installé

Ouvrez **PowerShell** et tapez :
```powershell
docker --version
docker-compose --version
```

> **Si vous voyez les numéros de version** (ex: `Docker version 24.x.x`) → Docker Desktop est déjà installé, **passez directement à l'Étape 1**.
>
> **Si vous voyez `command not found` ou une erreur** → Installez Docker Desktop maintenant :

1. Téléchargez [Docker Desktop pour Windows](https://www.docker.com/products/docker-desktop/)
2. Lors de l'installation, activez **"Use WSL 2 instead of Hyper-V"**
3. Redémarrez si demandé
4. Vérifiez après installation :
```powershell
docker --version
docker-compose --version
# Résultat attendu : Docker version 24.x.x / docker-compose version 2.x.x
```

> ⚠️ Si Docker Desktop demande d'activer WSL2, acceptez et redémarrez Windows avant de continuer.

### Étape 1 — Créer l'arborescence

Ouvrez **PowerShell en tant qu'Administrateur** :
```powershell
New-Item -ItemType Directory -Force -Path "C:\mysql-dr\data"
New-Item -ItemType Directory -Force -Path "C:\mysql-dr\config"
Set-Location "C:\mysql-dr"
```

### Étape 2 — Créer le `docker-compose.yml`

Créez le fichier `C:\mysql-dr\docker-compose.yml` :
```yaml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    container_name: mysql-pleniged
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: "RootPassword123!"
      MYSQL_DATABASE: "pleniged_test"
    ports:
      - "3306:3306"
    volumes:
      - C:/mysql-dr/data:/var/lib/mysql
      - C:/mysql-dr/config/my.cnf:/etc/mysql/my.cnf
```

### Étape 3 — Créer le fichier `C:\mysql-dr\config\my.cnf`

```ini
[mysqld]
server-id            = 20
log_bin              = /var/lib/mysql/mysql-bin.log
binlog_format        = ROW
gtid_mode            = ON
enforce_gtid_consistency = ON
```

> ⚠️ `server-id = 20` — unique sur tout le réseau.

### Étape 4 — Démarrer le conteneur

```powershell
Set-Location "C:\mysql-dr"
docker-compose up -d

# Attendre ~30 secondes
Start-Sleep -Seconds 30
docker ps   # Vérifier que le statut est "Up"
```

### Étape 5 — Créer l'utilisateur de réplication + Semi-Sync

> ⚠️ Sur Windows, le plugin `.dll` est utilisé automatiquement par MySQL dans le conteneur Linux (l'image Docker est toujours Linux !), donc on utilise `.so` comme sur Ubuntu.

```powershell
docker exec -it mysql-pleniged mysql -u root -pRootPassword123! -e "CREATE USER 'replicator'@'%' IDENTIFIED BY 'ReplicaPass2026!'; GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%'; INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so'; SET PERSIST rpl_semi_sync_master_enabled = 1; SET PERSIST rpl_semi_sync_master_timeout = 10000; FLUSH PRIVILEGES; SHOW MASTER STATUS;"
```

> 📋 **Notez les valeurs** de `SHOW MASTER STATUS` (`File` et `Position`) — à transmettre à Ahmed.

### Étape 6 — Ouvrir le port 3306 dans le Firewall Windows

```powershell
New-NetFirewallRule `
  -DisplayName "MySQL DR Replication Zone3" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 3306 `
  -RemoteAddress 192.168.1.64 `
  -Action Allow

New-NetFirewallRule `
  -DisplayName "MySQL Monitoring Zone1" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 9104 `
  -RemoteAddress 192.168.1.65 `
  -Action Allow
```

### Étape 7 — Insérer des données de test

```powershell
docker exec -it mysql-pleniged mysql -u root -pRootPassword123! -e "USE pleniged_test; CREATE TABLE IF NOT EXISTS dr_test (id INT AUTO_INCREMENT PRIMARY KEY, message VARCHAR(100), created_at DATETIME DEFAULT NOW()); INSERT INTO dr_test (message) VALUES ('Test initial Pleniged DR'); SELECT * FROM dr_test;"
```

---

## Tableau de coordination — À transmettre à Ahmed

Une fois vos étapes terminées, remplissez ce tableau et envoyez-le à Ahmed :

| Info | Click (Ubuntu) | Pleniged (Windows) |
|------|--------------|-------------------|
| IP source | `192.168.1.241` | `10.168.2.34` |
| Nom du conteneur | `mysql-click` | `mysql-pleniged` |
| Nom de la base test | `click_test` | `pleniged_test` |
| Utilisateur réplication | `replicator` | `replicator` |
| Mot de passe | `ReplicaPass2026!` | `ReplicaPass2026!` |
| `File` (SHOW MASTER STATUS) | *(à remplir)* | *(à remplir)* |
| `Position` (SHOW MASTER STATUS) | *(à remplir)* | *(à remplir)* |

---

## Vérification finale — Tester la connexion depuis les VMs DR

Ahmed doit exécuter ces commandes depuis les VMs DR pour confirmer que la connexion passe avant de démarrer la réplication :

```bash
# Depuis Zone 2 (192.168.1.66) → vers Click
sudo docker exec -it mysql-server mysql -h 192.168.1.241 -u replicator -pReplicaPass2026! -e "SHOW MASTER STATUS;"

# Depuis Zone 3 (192.168.1.64) → vers Pleniged
sudo docker exec -it mysql-server mysql -h 10.168.2.34 -u replicator -pReplicaPass2026! -e "SHOW MASTER STATUS;"
```

Si les deux commandes retournent un résultat sans erreur, c'est bon — Ahmed peut démarrer la réplication ! ✅
