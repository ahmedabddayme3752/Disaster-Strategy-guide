# 06 — Guide de Monitoring (Prometheus + Grafana)

## 🎯 Rôle de ce Guide
Ce guide explique le système de surveillance mis en place pour surveiller en temps réel la santé de **toutes les zones** du DR BNM, y compris la Production.

---

## 🏗️ Architecture du Monitoring

```
┌─────────────────────────────────────────────────────────────────────┐
│  ZONE 1 — Serveur de Monitoring (192.168.1.65)                      │
│                                                                     │
│  🐳 Prometheus (Port 9090)  ─── Collecte toutes les 10 secondes    │
│  🐳 Grafana    (Port 3000)  ─── Dashboard temps réel               │
│  🐳 Alertmanager (Port 9093) ── Alertes email/Slack                │
│                                                                     │
│  Fichiers clés :                                                    │
│  ~/monitoring/docker-compose.yml   ← Orchestration des conteneurs  │
│  ~/monitoring/prometheus.yml       ← Liste des cibles à surveiller │
│  ~/monitoring/bnm_dashboard_v4.json ← Dashboard Grafana BNM        │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │ Scrape (HTTP GET /metrics)
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  Zone 0 — Prod   │  │  Zone 2 — DR     │  │  Zone 3 — DR     │
│  .241 Port 9104  │  │  .66 Port 9104   │  │  .64 Port 9104   │
│  (--network=host)│  │  Port 9105       │  │  Port 9105       │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

---

## 🛠️ Technologie 1 : `mysqld-exporter` (Capteur)

### Qu'est-ce que c'est ?
`mysqld-exporter` se connecte à MySQL, lit ses statistiques internes (`SHOW GLOBAL STATUS`, `SHOW SLAVE STATUS`), et les expose en format texte sur `/metrics`.

### Métriques importantes :

| Métrique | Description |
|:---|:---|
| `mysql_up` | 1 = MySQL actif, 0 = MySQL down |
| `mysql_slave_status_seconds_behind_master` | Retard de réplication (en secondes) |
| `mysql_global_status_threads_connected` | Connexions MySQL actives |
| `mysql_global_status_questions` | Total des requêtes SQL exécutées |
| `mysql_global_status_uptime` | Durée de fonctionnement depuis le dernier démarrage |

---

## 🖥️ Zone 0 — Capteur sur la Production (Spécifique)

> [!IMPORTANT]
> Zone 0 utilise MySQL **natif** (pas Docker). On utilise `--network=host`
> pour que le conteneur exporter puisse atteindre le MySQL local.

### Créer l'utilisateur de monitoring (droits minimaux) :
```bash
# Sur Zone 0 (192.168.1.241)
mysql -u debian-sys-maint -p'MOT_DE_PASSE_MAINTENANCE' -e "
CREATE USER 'monitor'@'127.0.0.1'
  IDENTIFIED WITH mysql_native_password BY 'MonitorPass2026!';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'monitor'@'127.0.0.1';
FLUSH PRIVILEGES;
"
```

### Créer le fichier de credentials :
```bash
mkdir -p ~/monitoring-prod

cat > ~/monitoring-prod/monitor.my.cnf << 'EOF'
[client]
user=monitor
password=MonitorPass2026!
host=127.0.0.1
port=3366
EOF

chmod 644 ~/monitoring-prod/monitor.my.cnf
```

### Lancer le capteur avec `--network=host` :
```bash
# CRUCIAL : --network=host permet au conteneur d'atteindre 127.0.0.1:3366
# Sans cette option, 127.0.0.1 pointerait vers le conteneur lui-même !
sudo docker run -d \
  --name mysqld-exporter-prod \
  --restart always \
  --network=host \
  -v ~/monitoring-prod/monitor.my.cnf:/cfg/.my.cnf:ro \
  prom/mysqld-exporter \
  --config.my-cnf=/cfg/.my.cnf
```

> [!NOTE]
> Avec `--network=host`, le port 9104 est directement exposé sur la machine hôte.
> Pas besoin de `-p 9104:9104` dans ce cas !

### Ouvrir le pare-feu (uniquement pour Zone 1) :
```bash
sudo ufw allow from 192.168.1.65 to any port 9104 comment "Monitoring BNM Zone1"
sudo ufw reload
```

### Vérifier :
```bash
curl http://localhost:9104/metrics | grep mysql_up
# Résultat attendu : mysql_up 1 ✅
```

---

## 🐳 Zones 2 et 3 — Capteurs Docker (Configuration)

### Fichiers de credentials :

**`~/mysql-dr/monitoring/click.my.cnf` :**
```ini
[client]
user=root
password=RootPassword123!
host=mysql-click    # Nom du conteneur Docker (réseau interne)
port=3306
```

**`~/mysql-dr/monitoring/pleniged.my.cnf` :**
```ini
[client]
user=root
password=RootPassword123!
host=mysql-pleniged
port=3306
```

```bash
chmod 644 ~/mysql-dr/monitoring/*.my.cnf
```

### Section dans `docker-compose.yml` :
```yaml
  mysql-exporter-click:
    image: prom/mysqld-exporter
    container_name: mysql-exporter-click
    command:
      - "--config.my-cnf=/cfg/.my.cnf"
    volumes:
      - ./monitoring/click.my.cnf:/cfg/.my.cnf:ro
    ports:
      - "9104:9104"
    depends_on:
      - mysql-click
    restart: always

  mysql-exporter-pleniged:
    image: prom/mysqld-exporter
    container_name: mysql-exporter-pleniged
    command:
      - "--config.my-cnf=/cfg/.my.cnf"
    volumes:
      - ./monitoring/pleniged.my.cnf:/cfg/.my.cnf:ro
    ports:
      - "9105:9104"
    depends_on:
      - mysql-pleniged
    restart: always
```

---

## 🛠️ Technologie 2 : Prometheus

### Configuration complète (`~/monitoring/prometheus.yml`) :
```yaml
global:
  scrape_interval: 10s
  evaluation_interval: 10s

scrape_configs:

  # === ZONE 0 - PRODUCTION CLICK ===
  - job_name: 'BNM_PROD_CLICK_ZONE0'
    static_configs:
      - targets: ['192.168.1.241:9104']
        labels:
          zone: 'Zone0'
          service: 'Click_Production'

  # === ZONE 2 - DR PRIMARY ===
  - job_name: 'BNM_DR_CLICK_ZONE2'
    static_configs:
      - targets: ['192.168.1.66:9104']
        labels:
          zone: 'Zone2'
          service: 'Click'

  - job_name: 'BNM_DR_PLENIGED_ZONE2'
    static_configs:
      - targets: ['192.168.1.66:9105']
        labels:
          zone: 'Zone2'
          service: 'Pleniged'

  # === ZONE 3 - DR SECONDARY ===
  - job_name: 'BNM_DR_CLICK_ZONE3'
    static_configs:
      - targets: ['192.168.1.64:9104']
        labels:
          zone: 'Zone3'
          service: 'Click'

  - job_name: 'BNM_DR_PLENIGED_ZONE3'
    static_configs:
      - targets: ['192.168.1.64:9105']
        labels:
          zone: 'Zone3'
          service: 'Pleniged'
```

### Recharger sans interruption :
```bash
docker exec prometheus kill -HUP 1

# Vérifier les 7 cibles
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -E '"job"|"health"'
```

---

## 🛠️ Technologie 3 : Grafana

### Accès :
- **URL :** `http://192.168.1.65:3000`
- **Utilisateur :** `admin`
- **Mot de passe :** `GrafanaPass2026!`

### Ajouter la source de données Prometheus :
1. **Connections** → **Data Sources** → **Add data source** → **Prometheus**
2. **URL :** `http://prometheus:9090` *(nom du conteneur Docker)*
3. **Save & Test** → ✅

### Importer le Dashboard BNM :
1. **Dashboards** → **New** → **Import**
2. **Upload JSON file** → Sélectionner `~/monitoring/bnm_dashboard_v4.json`
3. Sélectionner la source Prometheus → **Import**

### Panneaux du Dashboard `BNM — Disaster Recovery Dashboard` :

| Panneau | Type | Requête | Description |
|:---|:---|:---|:---|
| 🏦 Production Click | Stat | `mysql_up{job='BNM_PROD_CLICK_ZONE0'}` | Vert = UP, Rouge = DOWN |
| 🟢 DR Click Zone 2 | Stat | `mysql_up{job='BNM_DR_CLICK_ZONE2'}` | Statut de l'esclave |
| 🟢 DR Click Zone 3 | Stat | `mysql_up{job='BNM_DR_CLICK_ZONE3'}` | Statut de l'esclave |
| 📄 DR Pleniged Zone 2 | Stat | `mysql_up{job='BNM_DR_PLENIGED_ZONE2'}` | Statut Pleniged |
| 📄 DR Pleniged Zone 3 | Stat | `mysql_up{job='BNM_DR_PLENIGED_ZONE3'}` | Statut Pleniged |
| ⏱️ Replication Lag | Timeseries | `mysql_slave_status_seconds_behind_master` | Retard en secondes |
| 🔌 Connexions actives | Timeseries | `mysql_global_status_threads_connected` | Par zone |
| ⚡ Requêtes/seconde | Timeseries | `rate(mysql_global_status_questions[1m])` | Activité SQL |
| 🕐 Uptime Production | Stat | `mysql_global_status_uptime` | En secondes |

> [!NOTE]
> Les panneaux Pleniged afficheront **"No data"** pour le Replication Lag
> jusqu'à la configuration de la réplication Pleniged (prochaine étape).
> C'est **normal et attendu**.

---

## 🔧 Commandes de Maintenance

```bash
# Sur Zone 1 — Vérifier tous les conteneurs monitoring
docker ps

# Recharger Prometheus après modification du prometheus.yml
docker exec prometheus kill -HUP 1

# Voir les logs Grafana
docker logs grafana --tail 30

# Tester un capteur depuis Zone 1
curl http://192.168.1.241:9104/metrics | grep mysql_up  # Zone 0
curl http://192.168.1.66:9104/metrics | grep mysql_up   # Zone 2 Click
curl http://192.168.1.66:9105/metrics | grep mysql_up   # Zone 2 Pleniged
curl http://192.168.1.64:9104/metrics | grep mysql_up   # Zone 3 Click
curl http://192.168.1.64:9105/metrics | grep mysql_up   # Zone 3 Pleniged
```
