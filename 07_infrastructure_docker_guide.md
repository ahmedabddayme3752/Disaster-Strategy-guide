# 07 — Infrastructure Complète & Images Docker
## Guide de chaque Serveur, Conteneur et leur Rôle

> [!NOTE]
> Ce guide décrit **exactement** ce qui tourne sur chaque machine du réseau BNM DR.
> Chaque image Docker est expliquée avec son objectif précis dans notre stratégie.

---

## 🗺️ Vue Globale du Réseau BNM DR

| Zone | IP | OS | Rôle Principal |
|:---|:---|:---|:---|
| **Zone 0** | `192.168.1.241` | Ubuntu Linux | Production Click (MySQL Natif) |
| **Zone 0 bis** | `10.168.2.34` | Windows 10 | Production Pleniged (MySQL Natif) |
| **Zone 1** | `192.168.1.65` | Ubuntu Linux | Monitoring Centralisé |
| **Zone 2** | `192.168.1.66` | Ubuntu Linux | DR Primaire (Site de Secours Principal) |
| **Zone 3** | `192.168.1.64` | Ubuntu Linux | DR Secondaire (Site de Secours Redondant) |

---

## 🏦 ZONE 0 — Serveur de Production Click (192.168.1.241)

### Rôle dans la Stratégie
C'est la **source de toutes les données**. Toutes les transactions bancaires, paiements mobiles et rechargements sont écrits ici en premier. Aucune donnée ne peut exister sur le DR sans avoir existé ici.

### Infrastructure
```
192.168.1.241 (Ubuntu Linux)
│
├── ⚙️  MySQL 8.0 — NATIF (pas dans Docker)
│       Port: 3366  ← Port non-standard pour sécurité
│       Bases: click, clickTest, clickTestPub, mail_otp
│       Config: /etc/mysql/mysql.conf.d/mysqld.cnf
│
└── 🐳 Conteneur Docker: mysqld-exporter-prod
        Image: prom/mysqld-exporter:latest
        Réseau: --network=host  ← Partage le réseau hôte (nécessaire pour atteindre MySQL natif)
        Port exposé: 9104
        Objectif: Lire les statistiques MySQL et les publier en HTTP pour Prometheus
        Config: ~/monitoring-prod/monitor.my.cnf
```

### Pourquoi `--network=host` sur Zone 0 ?
> Sans cette option, le conteneur Docker essaierait de se connecter à `127.0.0.1:3366` de son propre réseau interne (qui ne contient pas MySQL). Avec `--network=host`, le conteneur "voit" le réseau de la machine physique et peut donc atteindre le MySQL natif.

### Images Docker sur Zone 0

| Image | Conteneur | Port | Objectif |
|:---|:---|:---:|:---|
| `prom/mysqld-exporter` | `mysqld-exporter-prod` | `9104` | Capteur de métriques pour la Production |

---

## 📊 ZONE 1 — Serveur de Monitoring (192.168.1.65)

### Rôle dans la Stratégie
C'est le **cerveau de surveillance**. Il ne stocke aucune donnée bancaire. Son seul rôle est d'observer tous les autres serveurs et d'alerter l'équipe DSI en cas de problème. C'est le cockpit de pilotage du DR.

### Infrastructure
```
192.168.1.65 (Ubuntu Linux — monotordr)
│
├── 🐳 Conteneur: prometheus
│       Image: prom/prometheus:latest
│       Port: 9090
│       Config: ~/monitoring/prometheus.yml
│       Règles: ~/monitoring/alert_rules.yml
│       Objectif: Collecte les métriques de tous les exporters toutes les 10 secondes
│
├── 🐳 Conteneur: grafana
│       Image: grafana/grafana:latest
│       Port: 3000
│       Objectif: Affiche le Dashboard BNM et les alertes en temps réel
│       Dashboard: "BNM — Disaster Recovery Dashboard"
│
└── 🐳 Conteneur: alertmanager
        Image: prom/alertmanager:latest
        Port: 9093
        Config: ~/monitoring/alertmanager.yml
        Objectif: Reçoit les alertes de Prometheus et les envoie par email/Slack
```

### Flux de données de monitoring :
```
MySQL (Zone 0/2/3)
    → mysqld-exporter (expose /metrics)
    → Prometheus (collecte toutes les 10s)
    → Grafana (affiche le dashboard)
    → Alertmanager (envoie alertes si seuil dépassé)
    → Email DSI / Slack
```

### Images Docker sur Zone 1

| Image | Conteneur | Port | Objectif |
|:---|:---|:---:|:---|
| `prom/prometheus` | `prometheus` | `9090` | Collecte et stockage des métriques |
| `grafana/grafana` | `grafana` | `3000` | Visualisation et alertes UI |
| `prom/alertmanager` | `alertmanager` | `9093` | Routage des alertes vers les équipes |

---

## 🟢 ZONE 2 — DR Primaire (192.168.1.66)

### Rôle dans la Stratégie
C'est le **premier site de secours**. Si la Production tombe, c'est ici que l'on bascule en **PRIORITÉ**. Les données sont synchronisées en temps réel grâce au protocole GTID. Le délai de synchronisation (Lag) doit rester à **0 seconde** en permanence.

### Infrastructure
```
192.168.1.66 (Ubuntu Linux — primbackupdr)
│
├── 🐳 Conteneur: mysql-click
│       Image: mysql:8.0
│       Port: 3306  (externe) → 3306 (interne)
│       Rôle MySQL: ESCLAVE (slave) de 192.168.1.241:3366
│       Bases répliquées: click, clickTest, clickTestPub, mail_otp
│       Volume données: ./click/data:/var/lib/mysql
│       Config GTID: ./click/config/my.cnf
│       Objectif: Miroir temps réel de la base Click de production
│
├── 🐳 Conteneur: mysql-pleniged
│       Image: mysql:5.7  ← Doit correspondre à la version Windows !
│       Port: 3307  (externe) → 3306 (interne)
│       Rôle MySQL: ESCLAVE (slave) de 10.168.2.34:3306
│       Bases répliquées: plenesgi_ged
│       Volume données: ./pleniged/data:/var/lib/mysql
│       Config GTID: ./pleniged/config/my.cnf
│       Objectif: Miroir temps réel de la base Pleniged Windows
│
├── 🐳 Conteneur: mysql-exporter-click
│       Image: prom/mysqld-exporter:latest
│       Port: 9104  (externe) → 9104 (interne)
│       Config: ./monitoring/click.my.cnf
│       Objectif: Expose les métriques de mysql-click à Prometheus
│
└── 🐳 Conteneur: mysql-exporter-pleniged
        Image: prom/mysqld-exporter:latest
        Port: 9105  (externe) → 9104 (interne)
        Config: ./monitoring/pleniged.my.cnf
        Objectif: Expose les métriques de mysql-pleniged à Prometheus
```

### Images Docker sur Zone 2

| Image | Conteneur | Port | Objectif |
|:---|:---|:---:|:---|
| `mysql:8.0` | `mysql-click` | `3306` | Esclave Click (miroir Production) |
| `mysql:5.7` | `mysql-pleniged` | `3307` | Esclave Pleniged (miroir Windows) |
| `prom/mysqld-exporter` | `mysql-exporter-click` | `9104` | Capteur métriques Click |
| `prom/mysqld-exporter` | `mysql-exporter-pleniged` | `9105` | Capteur métriques Pleniged |

---

## 🔵 ZONE 3 — DR Secondaire (192.168.1.64)

### Rôle dans la Stratégie
C'est le **deuxième site de secours** (redondance). Il est identique à Zone 2 dans sa structure. Si Zone 2 est aussi touchée par le sinistre, Zone 3 prend le relais. Les deux zones fonctionnent en parallèle : c'est la **défense en profondeur** de la BNM.

### Infrastructure
```
192.168.1.64 (Ubuntu Linux — secbackupdr)
│
├── 🐳 Conteneur: mysql-click
│       Image: mysql:8.0
│       Port: 3306
│       Rôle MySQL: ESCLAVE de 192.168.1.241:3366
│       Objectif: Deuxième miroir Click (redondance)
│
├── 🐳 Conteneur: mysql-pleniged
│       Image: mysql:5.7
│       Port: 3307
│       Rôle MySQL: ESCLAVE de 10.168.2.34:3306
│       Objectif: Deuxième miroir Pleniged (redondance)
│
├── 🐳 Conteneur: mysql-exporter-click
│       Image: prom/mysqld-exporter:latest
│       Port: 9104
│       Objectif: Capteur métriques Click pour Zone 3
│
└── 🐳 Conteneur: mysql-exporter-pleniged
        Image: prom/mysqld-exporter:latest
        Port: 9105
        Objectif: Capteur métriques Pleniged pour Zone 3
```

### Images Docker sur Zone 3

| Image | Conteneur | Port | Objectif |
|:---|:---|:---:|:---|
| `mysql:8.0` | `mysql-click` | `3306` | Deuxième esclave Click |
| `mysql:5.7` | `mysql-pleniged` | `3307` | Deuxième esclave Pleniged |
| `prom/mysqld-exporter` | `mysql-exporter-click` | `9104` | Capteur métriques Click |
| `prom/mysqld-exporter` | `mysql-exporter-pleniged` | `9105` | Capteur métriques Pleniged |

---

## 🔄 Comment la Réplication GTID fonctionne

```
PRODUCTION (.241 ou .34)
│
│  1️⃣  L'application écrit une transaction :
│      INSERT INTO click.clients VALUES (...)
│
│  2️⃣  MySQL génère un GTID unique :
│      816c0b5e-97e8-11ed-a73b-000c299194d8:55883
│
│  3️⃣  MySQL écrit dans le Binary Log (binlog)
│
▼
────────────── Réseau TCP ──────────────
│
▼
Site DR (Zone 2 ou Zone 3)
│
│  4️⃣  L'esclave lit le binlog du maître
│      (IO Thread)
│
│  5️⃣  L'esclave exécute la même transaction
│      (SQL Thread)
│
│  6️⃣  Seconds_Behind_Master = 0 ✅
│      (Synchronisation parfaite)
│
▼
mysqld-exporter → Prometheus → Grafana → DSI informée
```

---

## ⚠️ Note Importante — Bug Docker Compose 1.29

> [!WARNING]
> Sur les Zones 2 et 3, Docker Compose version 1.29 a un bug connu (**KeyError: 'ContainerConfig'**). Si un conteneur est modifié et qu'on tente de le recréer, l'erreur se produit. **Solution :** Toujours faire `sudo docker rm -f NOM_CONTENEUR` avant `docker-compose up -d`.
> Voir le Guide 05 pour la procédure complète.
