# 06 — Guide de Monitoring (Prometheus + Grafana)

## 🎯 Rôle de ce Guide
Ce guide explique le système de surveillance mis en place pour surveiller en temps réel la santé de toutes les instances MySQL DR de la BNM.

---

## 🏗️ Architecture du Monitoring

```
┌─────────────────────────────────────────────────────────────────┐
│  ZONE 1 — Serveur de Monitoring (192.168.1.65)                   │
│                                                                  │
│  🐳 Prometheus (Port 9090)  ←── Collecte les métriques          │
│  🐳 Grafana    (Port 3000)  ←── Affiche le Dashboard            │
│  🐳 Alertmanager (Port 9093) ←── Envoie les alertes             │
└────────────────────────┬────────────────────────────────────────┘
                          │ Scrape toutes les 15 secondes
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Zone 2 .66  │  │  Zone 3 .64  │  │  (Futur)     │
│ 📡 :9104     │  │ 📡 :9104     │  │              │
│ 📡 :9105     │  │ 📡 :9105     │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
```

---

## 🛠️ Technologie 1 : `mysqld-exporter`

### Qu'est-ce que c'est ?
`mysqld-exporter` est un **agent de collecte** (aussi appelé "exporter" ou "capteur"). Il se connecte à une instance MySQL, récupère ses statistiques internes, et les expose sur une URL HTTP que Prometheus peut lire.

### Comment il fonctionne ?
```
MySQL                    mysqld-exporter         Prometheus
  │                           │                      │
  │ ← Se connecte (port 3306) │                      │
  │ SHOW GLOBAL STATUS;       │                      │
  │ SHOW SLAVE STATUS;        │                      │
  │ ─────────────────────────▶│                      │
  │                           │ Expose les métriques │
  │                           │ sur /metrics (9104)  │
  │                           │ ◀──────── Scrape ────│
  │                           │                      │
```

### Configuration (`./monitoring/click.my.cnf`) :
```ini
[client]
user=root
password=RootPassword123!
host=mysql-click    # Nom du conteneur Docker
port=3306
```

### Métriques importantes collectées :
| Métrique | Description |
|:---|:---|
| `mysql_up` | 1 = MySQL actif, 0 = MySQL down |
| `mysql_slave_status_seconds_behind_master` | Retard de réplication en secondes |
| `mysql_global_status_threads_connected` | Connexions MySQL actives |
| `mysql_global_status_questions` | Nombre total de requêtes SQL |
| `mysql_global_status_uptime` | Durée de fonctionnement MySQL |

---

## 🛠️ Technologie 2 : Prometheus

### Qu'est-ce que c'est ?
Prometheus est un **système de collecte et de stockage de métriques**. Il "scrape" (récupère) régulièrement les métriques de tous les exporters et les stocke dans sa base de données temporelle (TSDB).

### Configuration (`~/monitoring/prometheus.yml`) :
```yaml
global:
  scrape_interval: 15s    # Collecter les métriques toutes les 15 secondes
  evaluation_interval: 15s

scrape_configs:
  # === BNM DISASTER RECOVERY - ZONE 2 ===
  - job_name: 'BNM_DR_CLICK_ZONE2'
    static_configs:
      - targets: ['192.168.1.66:9104']   # IP Zone 2 : Port exporter Click
        labels:
          zone: 'Zone2'
          service: 'Click'

  - job_name: 'BNM_DR_PLENIGED_ZONE2'
    static_configs:
      - targets: ['192.168.1.66:9105']   # IP Zone 2 : Port exporter Pleniged
        labels:
          zone: 'Zone2'
          service: 'Pleniged'

  # === BNM DISASTER RECOVERY - ZONE 3 ===
  - job_name: 'BNM_DR_CLICK_ZONE3'
    static_configs:
      - targets: ['192.168.1.64:9104']   # IP Zone 3 : Port exporter Click
        labels:
          zone: 'Zone3'
          service: 'Click'

  - job_name: 'BNM_DR_PLENIGED_ZONE3'
    static_configs:
      - targets: ['192.168.1.64:9105']   # IP Zone 3 : Port exporter Pleniged
        labels:
          zone: 'Zone3'
          service: 'Pleniged'
```

### Recharger la configuration (sans redémarrage) :
```bash
# Sur la machine Zone 1
docker exec prometheus kill -HUP 1

# Vérifier que toutes les cibles sont UP
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -E '"job"|"health"'
```

---

## 🛠️ Technologie 3 : Grafana

### Qu'est-ce que c'est ?
Grafana est le **tableau de bord visuel**. Il se connecte à Prometheus comme source de données et affiche les métriques sous forme de graphiques, jauges, et indicateurs en temps réel.

### Accès :
- **URL :** `http://192.168.1.65:3000`
- **Utilisateur :** `admin`
- **Mot de passe :** `GrafanaPass2026!`

### Configuration de la source de données Prometheus :
1. Menu gauche → **Connections** → **Data Sources**
2. **Add data source** → **Prometheus**
3. **URL :** `http://prometheus:9090`
   *(Nom du conteneur Docker, pas l'IP — les conteneurs sont sur le même réseau Docker)*
4. **Save & Test** → Message vert ✅

### Dashboard BNM (JSON Model) :
Le Dashboard `BNM - Disaster Recovery MySQL` surveille :

| Panneau | Requête Prometheus | Description |
|:---|:---|:---|
| **Click Zone2 Status** | `mysql_up{job='BNM_DR_CLICK_ZONE2'}` | 1 = UP, 0 = DOWN |
| **Pleniged Zone2 Status** | `mysql_up{job='BNM_DR_PLENIGED_ZONE2'}` | 1 = UP, 0 = DOWN |
| **Click Zone3 Status** | `mysql_up{job='BNM_DR_CLICK_ZONE3'}` | 1 = UP, 0 = DOWN |
| **Pleniged Zone3 Status** | `mysql_up{job='BNM_DR_PLENIGED_ZONE3'}` | 1 = UP, 0 = DOWN |
| **Replication Lag** | `mysql_slave_status_seconds_behind_master` | Retard en secondes |
| **Connexions actives** | `mysql_global_status_threads_connected` | Connexions simultanées |
| **Requêtes/seconde** | `rate(mysql_global_status_questions[5m])` | Activité SQL |

---

## 🛠️ Technologie 4 : Alertmanager

### Qu'est-ce que c'est ?
Alertmanager reçoit les alertes de Prometheus et les envoie à l'équipe (email, Slack, etc.).

### Configuration (`~/monitoring/alertmanager.yml`) :
```yaml
route:
  receiver: 'email-bnm'

receivers:
  - name: 'email-bnm'
    email_configs:
      - to: 'dsi@bnm.mr'
        from: 'alerting@bnm.mr'
        smarthost: 'smtp.bnm.mr:587'
```

---

## 🔧 Commandes de Maintenance Courantes

```bash
# Démarrer tout le stack de monitoring
cd ~/monitoring && docker-compose up -d

# Vérifier le statut de tous les conteneurs
docker ps

# Voir les logs Prometheus (débogage)
docker logs prometheus --tail 50

# Voir les logs Grafana
docker logs grafana --tail 50

# Recharger la config Prometheus sans interruption
docker exec prometheus kill -HUP 1
```
