# 🤖 BNM DR Agent — Contexte Projet pour IA

## 🎯 Mission
Tu travailles sur l'infrastructure de **Disaster Recovery (DR)** des bases de données critiques de la **Banque Nationale de Mauritanie (BNM)**. Le projet est presque terminé. Lis ce document ENTIÈREMENT avant de répondre à quoi que ce soit.

---

## 🏗️ Architecture Réseau (5 Zones)

| Zone | IP | OS | Utilisateur SSH | Rôle |
|:---|:---|:---|:---|:---|
| **Zone 0** | `192.168.1.241` | Ubuntu Linux | `connect` | Production Click (MySQL 8.0, Port 3366) |
| **Zone 0b** | `10.168.2.34` | Windows 10 | — | Production Pleniged (MySQL 5.7, Port 3306) |
| **Zone 1** | `192.168.1.65` | Ubuntu Linux | `monotordr` | Monitoring (Prometheus + Grafana + Alertmanager) |
| **Zone 2** | `192.168.1.66` | Ubuntu Linux | `primbackupdr` | DR Primaire (mysql-click:3306, mysql-pleniged:3307) |
| **Zone 3** | `192.168.1.64` | Ubuntu Linux | `secbackupdr` | DR Secondaire (mysql-click:3306, mysql-pleniged:3307) |

---

## 🐳 Docker par Zone

### Zone 0 (Production .241)
```
prom/mysqld-exporter  → Port 9104  (--network=host, OBLIGATOIRE pour MySQL natif)
```

### Zone 1 (Monitoring .65) — Dossier: ~/monitoring/
```
prom/prometheus       → Port 9090   Config: prometheus.yml + alert_rules.yml
grafana/grafana       → Port 3000   Dashboard: "BNM — Disaster Recovery Dashboard v8"
prom/alertmanager     → Port 9093   Config: alertmanager.yml
```

### Zone 2 (DR Primaire .66) — Dossier: ~/mysql-dr/
```
mysql:8.0             → mysql-click     Port 3306  (Esclave Click de .241:3366)
mysql:5.7             → mysql-pleniged  Port 3307  (Esclave Pleniged de .34:3306)
prom/mysqld-exporter  → mysql-exporter-click     Port 9104
prom/mysqld-exporter  → mysql-exporter-pleniged  Port 9105
```

### Zone 3 (DR Secondaire .64) — Dossier: ~/mysql-dr/
```
mysql:8.0             → mysql-click     Port 3306  (Esclave Click de .241:3366)
mysql:5.7             → mysql-pleniged  Port 3307  (Esclave Pleniged de .34:3306)
prom/mysqld-exporter  → mysql-exporter-click     Port 9104
prom/mysqld-exporter  → mysql-exporter-pleniged  Port 9105
```

---

## 🗄️ Bases de Données Répliquées (MIRRORING TOTAL)

| Base | Système | Taille | Statut |
|:---|:---|:---|:---|
| `click` | Click (MySQL 8.0) | 73 Mo | ✅ Répliqué Lag=0s |
| `clickTest` | Click (MySQL 8.0) | 19 Mo | ✅ Répliqué Lag=0s |
| `clickTestPub` | Click (MySQL 8.0) | — | ✅ Répliqué Lag=0s |
| `mail_otp` | Click (MySQL 8.0) | — | ✅ Répliqué Lag=0s |
| `plenesgi_ged` | Pleniged (MySQL 5.7) | 1303 Mo | ✅ Répliqué Lag=0s |

---

## 🔐 Tous les Identifiants

### MySQL
| Utilisateur | Mot de passe | Rôle |
|:---|:---|:---|
| `root` | `RootPassword123!` | Admin Docker DR (Zones 2/3) |
| `replicator` | `ReplicaPass2026!` | Réplication GTID (maître → esclave) |
| `monitor` | `MonitorPass2026!` | Lecture seule métriques (Zone 0) |
| `debian-sys-maint` | voir `/etc/mysql/debian.cnf` | Maintenance Ubuntu (Zone 0) |

### Grafana
| Utilisateur | Mot de passe | URL |
|:---|:---|:---|
| `admin` | `GrafanaPass2026!` | http://192.168.1.65:3000 |

### Sources de données Grafana
| Nom | Type | UID | URL |
|:---|:---|:---|:---|
| `prometheus` | Prometheus | `ffhw0gqw7focgc` | http://prometheus:9090 |
| `alertmanager` | Alertmanager (Prometheus) | `dfhysky8o1vy8d` | http://alertmanager:9093 |

> ⚠️ Dans la config Alertmanager → choisir **Implementation: Prometheus** (pas Mimir/Cortex)

---

## 📁 Fichiers Importants

### Zone 1 (Monitoring)
```
~/monitoring/
├── prometheus.yml          # Config scraping (7 cibles)
├── alert_rules.yml         # Règles d'alertes Click + Pleniged
├── alertmanager.yml        # Config Alertmanager
└── bnm_dashboard_v8.json   # Dashboard Grafana (sauvegarde)
```

### Zone 2 & Zone 3 (DR)
```
~/mysql-dr/
├── docker-compose.yml              # Orchestration des 4 conteneurs
├── click/
│   ├── config/my.cnf               # Config MySQL 8.0 avec GTID
│   └── data/                       # Données MySQL Click
├── pleniged/
│   ├── config/my.cnf               # Config MySQL 5.7 avec GTID
│   └── data/                       # Données MySQL Pleniged (1303 Mo)
└── monitoring/
    ├── click.my.cnf                 # Auth exporter Click
    └── pleniged.my.cnf              # Auth exporter Pleniged
```

---

## 📊 État du Monitoring (7/7 Cibles Actives)

```yaml
# prometheus.yml scrape jobs (tous UP)
- job_name: BNM_PROD_CLICK_ZONE0     # 192.168.1.241:9104
- job_name: BNM_DR_CLICK_ZONE2       # 192.168.1.66:9104
- job_name: BNM_DR_CLICK_ZONE3       # 192.168.1.64:9104
- job_name: BNM_DR_PLENIGED_ZONE2    # 192.168.1.66:9105
- job_name: BNM_DR_PLENIGED_ZONE3    # 192.168.1.64:9105
```

---

## 🚨 Alertes Prometheus Configurées

```yaml
# alert_rules.yml (~/monitoring/)
- ProductionDown          # mysql_up{job="BNM_PROD_CLICK_ZONE0"} == 0
- ClickReplicationStopped # mysql_slave_status_slave_io_running{job=~"BNM_DR_CLICK.*"} == 0
- PlenigedReplicationStopped # mysql_slave_status_slave_io_running{job=~"BNM_DR_PLENIGED.*"} == 0
- ClickLagWarning         # seconds_behind_master > 30s
```

---

## 🔧 Commandes Essentielles

### Vérifier réplication (Zone 2 ou Zone 3)
```bash
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "SHOW SLAVE STATUS\G" | grep -E "Running:|Seconds_Behind|Last_Error"
sudo docker exec -it mysql-pleniged mysql -u root -p'RootPassword123!' -e "SHOW SLAVE STATUS\G" | grep -E "Running:|Seconds_Behind|Last_Error"
```

### Recharger config Prometheus (Zone 1)
```bash
docker exec prometheus kill -HUP 1
```

### ⚠️ Bug Docker Compose 1.29 (KeyError: ContainerConfig)
```bash
# Toujours supprimer le conteneur avant de le recréer !
sudo docker rm -f NOM_CONTENEUR
sudo docker-compose up -d
```

### Connexion avec `!` dans le mot de passe
```bash
set +H  # Désactiver l'expansion d'historique
```

---

## ✅ Ce qui est TERMINÉ

- [x] Click DR Zone 2 — Réplication GTID opérationnelle (Lag = 0s)
- [x] Click DR Zone 3 — Réplication GTID opérationnelle (Lag = 0s)
- [x] Pleniged DR Zone 2 — Import 1303 Mo + Réplication opérationnelle
- [x] Pleniged DR Zone 3 — Import 1303 Mo + Réplication opérationnelle (Lag = 0s)
- [x] Monitoring centralisé (7/7 cibles UP)
- [x] Dashboard Grafana v8 (Click + Pleniged + Métriques)
- [x] Alertes Prometheus (Production Down, Réplication Stoppée, Lag élevé)
- [x] Documentation complète (8 guides techniques)

## ⏳ Ce qui reste à faire

- [ ] Valider l'affichage des alertes Pleniged dans Grafana (config Alertmanager Implementation: Prometheus)
- [ ] Vérifier Lag Pleniged sur le Dashboard v8
- [ ] Mettre à jour le panneau Pleniged (passer de `mysql_up` à `mysql_slave_status_slave_sql_running`)

---

## 📖 Documentation (dans le dossier workspace)

| Fichier | Contenu |
|:---|:---|
| `README.md` | Vue d'ensemble complète |
| `01_dr_architecture_strategy.md` | GTID, RPO/RTO |
| `02_master_config_guide.md` | Config serveurs source |
| `03_start_replication_guide.md` | Magic Sync (procédure de réplication) |
| `04_validation_replication_guide.md` | Tests de validation |
| `05_dr_troubleshooting_knowledge_base.md` | Erreurs connues et solutions |
| `06_monitoring_guide.md` | Prometheus + Grafana |
| `07_infrastructure_docker_guide.md` | Images Docker + rôles par zone |
| `08_users_access_guide.md` | Tous les utilisateurs et accès |
| `task.md` | Liste des tâches (suivi de progression) |
