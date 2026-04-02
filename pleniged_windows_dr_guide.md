# 🪟 Guide DR — Pleniged (Windows 10 Physique) — RÉALITÉ TERRAIN

> [!IMPORTANT]
> **Informations réelles découvertes lors de l'exploration :**
> - MySQL **5.7.33** installé nativement dans `C:\PLENISOFTS\` (géré par **Laragon**)
> - Pas d'accès root → On utilise **`plenesgi_ged`** qui a `ALL PRIVILEGES ON *.*`
> - **Une base** à répliquer : `plenesgi_ged`
> - Laragon gère le démarrage/arrêt de MySQL (icône dans la barre des tâches)

---

## 📋 Référence Rapide

| Paramètre | Valeur |
|---|---|
| **IP Source** | `10.168.2.34` |
| **Port MySQL** | `3306` |
| **MySQL version** | `5.7.33` |
| **Binaire mysql.exe** | `C:\PLENISOFTS\bin\mysql\mysql-5.7.33-winx64\bin\mysql.exe` |
| **my.ini** | `C:\PLENISOFTS\bin\mysql\mysql-5.7.33-winx64\my.ini` |
| **Données** | `C:\PLENISOFTS\data\mysql\` |
| **Utilisateur admin** | `plenesgi_ged` / `453a,IThaAqV` |
| **User réplication** | `replicator` / `ReplicaPass2026!` |
| **Bases à répliquer** | `plenesgi_ged` |
| **Lanceur MySQL** | Laragon (`C:\PLENISOFTS\laragon.exe`) |

---

# ══════════════════════════════════════════
# PARTIE A — SOURCE WINDOWS (10.168.2.34)
# ══════════════════════════════════════════

## A1 — ✅ Utilisateur de réplication créé

> Cette étape est **déjà faite**. L'utilisateur `replicator` a été créé.

Pour vérifier :
```powershell
$mysql = "C:\PLENISOFTS\bin\mysql\mysql-5.7.33-winx64\bin\mysql.exe"
& $mysql -u plenesgi_ged "-p453a,IThaAqV" -e "SELECT User, Host FROM mysql.user WHERE User='replicator';"
```

---

## A2 — ✅ Modifier le `my.ini`

> Cette étape est **déjà faite**. Vérifier le contenu :

```powershell
Get-Content "C:\PLENISOFTS\bin\mysql\mysql-5.7.33-winx64\my.ini"
```

Le fichier doit contenir dans la section `[mysqld]` :
```ini
server-id                = 20
log_bin                  = mysql-bin
binlog_format            = ROW
gtid_mode                = ON
enforce_gtid_consistency = ON
log_slave_updates        = ON
binlog_do_db             = plenesgi_ged
expire_logs_days         = 7
```

---

## A3 — 🔧 Redémarrer MySQL via Laragon

> [!IMPORTANT]
> Les changements du `my.ini` ne sont actifs **qu'après redémarrage**. Il faut ~2 minutes d'interruption.

### Option 1 — Interface graphique (recommandé)
1. Cherche l'icône **Laragon** dans la barre des tâches (coin bas-droit, systray)
2. Clic droit sur l'icône → **MySQL** → **Stop**
3. Attendre 5 secondes
4. Clic droit → **MySQL** → **Start**

### Option 2 — PowerShell (Admin)
```powershell
# Arrêter mysqld
Stop-Process -Name "mysqld" -Force
Write-Host "MySQL arrêté. Attente 5 secondes..." -ForegroundColor Yellow
Start-Sleep -Seconds 5

# Relancer via Laragon (si Laragon ne relance pas automatiquement)
Start-Process "C:\PLENISOFTS\laragon.exe"
Start-Sleep -Seconds 10

# Vérifier que MySQL est bien relancé
Get-Process mysqld -ErrorAction SilentlyContinue | Select-Object Name, Id, StartTime
```

---

## A4 — 🔧 Vérifier que GTID et BinLog sont actifs

```powershell
$mysql = "C:\PLENISOFTS\bin\mysql\mysql-5.7.33-winx64\bin\mysql.exe"

& $mysql -u plenesgi_ged "-p453a,IThaAqV" -e "
SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'server_id';
SHOW VARIABLES LIKE 'binlog_format';
SHOW MASTER STATUS;
"
```

**Résultat attendu :**
```
gtid_mode     = ON      ✅
log_bin       = ON      ✅
server_id     = 20      ✅
binlog_format = ROW     ✅
```

> [!CAUTION]
> Si `gtid_mode = OFF` ou `log_bin = OFF` → le redémarrage n'a pas fonctionné.
> Vérifier que le `my.ini` est bien modifié et relancer Laragon.

---

## A5 — 🔧 Exporter LA base (Dump Initial)

```powershell
$mysqldump = "C:\PLENISOFTS\bin\mysql\mysql-5.7.33-winx64\bin\mysqldump.exe"

# Dump de plenesgi_ged
& $mysqldump -u root -p'RootBNM2026!' `
  --databases plenesgi_ged `
  --single-transaction `
  --routines `
  --triggers `
  --set-gtid-purged=ON `
  --no-tablespaces `
  --result-file="C:\plenesgi_ged_dump.sql"

Write-Host "Dump plenesgi_ged terminé !" -ForegroundColor Green
Get-Item "C:\plenesgi_ged_dump.sql" | Select-Object Name, @{N='Taille';E={[math]::Round($_.Length/1MB,2)}}
```

> ✅ Envoie le fichier `C:\plenesgi_ged_dump.sql` à Ahmed.

---

## A6 — 🔧 Ouvrir le Firewall Windows

```powershell
# Autoriser Zone 2 (Async)
New-NetFirewallRule `
  -DisplayName "MySQL DR - Zone 2 Async" `
  -Direction Inbound -Protocol TCP -LocalPort 3306 `
  -RemoteAddress 192.168.1.66 -Action Allow

# Autoriser Zone 3 (Async)
New-NetFirewallRule `
  -DisplayName "MySQL DR - Zone 3 Async" `
  -Direction Inbound -Protocol TCP -LocalPort 3306 `
  -RemoteAddress 192.168.1.64 -Action Allow

# Autoriser Zone 1 (Monitoring)
New-NetFirewallRule `
  -DisplayName "Prometheus Monitoring Zone1" `
  -Direction Inbound -Protocol TCP -LocalPort 9104 `
  -RemoteAddress 192.168.1.65 -Action Allow

# Vérifier
Get-NetFirewallRule | Where-Object {$_.DisplayName -like "MySQL DR*" -or $_.DisplayName -like "Prometheus*"} |
    Select-Object DisplayName, Enabled
```

---

## A7 — 🔧 Récupérer le GTID à communiquer à Ahmed

```powershell
$mysql = "C:\PLENISOFTS\bin\mysql\mysql-5.7.33-winx64\bin\mysql.exe"
& $mysql -u root -p'RootBNM2026!' -e "SHOW MASTER STATUS;"
```

> 📋 Notez la valeur de **`Executed_Gtid_Set`** — c'est ce qu'Ahmed utilisera pour `GTID_PURGED`.

---

### ✅ Checklist Partie A

- [x] Utilisateur `replicator` créé
- [x] `my.ini` modifié (sans semi-sync)
- [x] MySQL redémarré avec succès (A3)
- [x] GTID=ON, log_bin=ON vérifiés (A4)
- [ ] Dump `plenesgi_ged` créé (A5)
- [ ] Firewall ouvert pour Z2, Z3, Z1 (A6)
- [ ] GTID_PURGED communiqué à Ahmed (A7)

---

# ══════════════════════════════════════════
# PARTIE B — CÔTÉ DR : Ahmed (Zone 2 & Zone 3)
# ══════════════════════════════════════════

> 👤 **Ces étapes sont réalisées sur les VMs DR.**
> Les conteneurs `mysql-pleniged` tournent sur le port **3307** et doivent désormais utiliser **MySQL 5.7** (comme la source).

---

## B1 — Tester la connectivité vers la source

```bash
# Depuis Zone 2 (192.168.1.66)
sudo docker exec -it mysql-pleniged \
  mysql -h 10.168.2.34 -P 3306 -u replicator -p'ReplicaPass2026!' \
  -e "SHOW MASTER STATUS;"

# Depuis Zone 3 (192.168.1.64)
sudo docker exec -it mysql-pleniged \
  mysql -h 10.168.2.34 -P 3306 -u replicator -p'ReplicaPass2026!' \
  -e "SHOW MASTER STATUS;"
```

> ✅ Succès → continuer | ❌ Erreur 2003 → vérifier Firewall Windows (étape A7)

---

## B2 — Trouver le GTID_PURGED dans le dump

```bash
# Chercher dans le dump
grep "GTID_PURGED" /tmp/plenesgi_ged_dump.sql

# Exemple de résultat :
# SET @@GLOBAL.GTID_PURGED='a1b2c3d4-xxxx-xxxx-xxxx-000000000001:1-523';
```

---

## B3 — Importer le dump sur les conteneurs

```bash
# Importer plenesgi_ged
sudo docker cp /tmp/plenesgi_ged_dump.sql mysql-pleniged:/tmp/
sudo docker exec -i mysql-pleniged \
  mysql -u root -p'RootPassword123!' < /tmp/plenesgi_ged_dump.sql

# Vérifier que la base est présente
sudo docker exec -it mysql-pleniged \
  mysql -u root -p'RootPassword123!' -e "SHOW DATABASES;"
```

---

## B4 — Démarrer la réplication sur Zone 2 (Async)

> ⚠️ Assure-toi que le docker-compose a bien été mis à jour avec image: mysql:5.7.33

sudo docker exec -it mysql-pleniged mysql -u root -p'RootPassword123!' -e "
-- Nettoyage
STOP SLAVE;
RESET SLAVE ALL;
RESET MASTER;

-- GTID de départ (remplacer par la valeur des dumps)
SET GLOBAL GTID_PURGED = 'XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX:1-NNN';

-- Connexion Semi-Sync vers la source Windows
CHANGE MASTER TO
  MASTER_HOST     = '10.168.2.34',
  MASTER_PORT     = 3306,
  MASTER_USER     = 'replicator',
  MASTER_PASSWORD = 'ReplicaPass2026!',
  MASTER_AUTO_POSITION = 1;

-- Démarrage
START SLAVE;
"
```

### Vérifier Zone 2
```bash
sudo docker exec -it mysql-pleniged \
  mysql -u root -p'RootPassword123!' -e "SHOW SLAVE STATUS\G" | \
  grep -E "Slave_IO_Running|Slave_SQL_Running|Seconds_Behind|Last_Error|Master_Host"
```

---

## B5 — Démarrer la réplication sur Zone 3 (Async)

```bash
# Sur Zone 3 (192.168.1.64) — même procédure, mode Async
set +H

sudo docker exec -it mysql-pleniged mysql -u root -p'RootPassword123!' -e "
STOP SLAVE;
RESET SLAVE ALL;
RESET MASTER;

SET GLOBAL GTID_PURGED = 'XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX:1-NNN';

CHANGE MASTER TO
  MASTER_HOST     = '10.168.2.34',
  MASTER_PORT     = 3306,
  MASTER_USER     = 'replicator',
  MASTER_PASSWORD = 'ReplicaPass2026!',
  MASTER_AUTO_POSITION = 1;

START SLAVE;
"
```

---

## B6 — Test de validation en temps réel

### Insérer depuis Windows (Source)
```powershell
$mysql = "C:\PLENISOFTS\bin\mysql\mysql-5.7.33-winx64\bin\mysql.exe"

& $mysql -u plenesgi_ged "-p453a,IThaAqV" -e "
USE plenesgi_ged;
CREATE TABLE IF NOT EXISTS dr_validation (
  id INT AUTO_INCREMENT PRIMARY KEY,
  message VARCHAR(100),
  created_at DATETIME DEFAULT NOW()
);
INSERT INTO dr_validation (message) VALUES ('Test DR OK');
SELECT * FROM dr_validation ORDER BY id DESC LIMIT 3;
"
```

### Vérifier sur Zone 2 ET Zone 3
```bash
sudo docker exec -it mysql-pleniged \
  mysql -u root -p'RootPassword123!' -e \
  "SELECT * FROM plenesgi_ged.dr_validation ORDER BY id DESC LIMIT 3;"
```

> ✅ Les mêmes données sur les 3 machines = **Réplication Pleniged opérationnelle !**

---

## B7 — Créer l'utilisateur Prometheus (pour le monitoring)

```bash
# Sur Zone 2 ET Zone 3
sudo docker exec -it mysql-pleniged mysql -u root -p'RootPassword123!' -e "
CREATE USER IF NOT EXISTS 'prometheus'@'192.168.1.65'
  IDENTIFIED BY 'MonitorPass2026!';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'prometheus'@'192.168.1.65';
FLUSH PRIVILEGES;
"
```

---

## 🚨 Dépannage — Problèmes Courants

| Erreur | Cause | Solution |
|---|---|---|
| `ERROR 2003: Can't connect` | Firewall Windows bloque | Vérifier A7, tester `telnet 10.168.2.34 3306` |
| `Slave_IO_Running: Connecting` | Routeur / Pare-feu Windows bloque | Vérifier A6, ou demander à l'équipe réseau une route (192.168.1.x → 10.168.2.x port 3306) |
| `KeyError: 'ContainerConfig'` | Docker Compose cache corrompu | Lancer `sudo docker rm -f mysql-pleniged` avant de refaire le `up -d` |
| `gtid_mode = OFF` après restart | my.ini pas bien sauvegardé | Relire le my.ini et redémarrer Laragon |
| `Last_Error: 1236` | GTID_PURGED incorrect | Relire le grep sur le dump et corriger |
| Plugin `.dll` non trouvé | Laragon allégé | Ignorer et faire de la réplication Asynchrone |
| `ERROR 1840: @@GLOBAL.GTID_PURGED` | Transactions résiduelles MySQL | Exécuter `RESET MASTER;` avant d'importer le dump |

---

## 📞 Tableau de Coordination

| Information | Valeur |
|---|---|
| IP Source Windows | `10.168.2.34` |
| Port Source | `3306` |
| Bases répliquées | `plenesgi_ged` |
| User réplication | `replicator` / `ReplicaPass2026!` |
| GTID_PURGED (à remplir) | *(extraire du dump)* |
| Dump ged envoyé | ☐ Oui / ☐ Non |
| Zone 2 réplication | ☐ OK / ☐ En attente |
| Zone 3 réplication | ☐ OK / ☐ En attente |
