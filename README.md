# 🏦 BNM Disaster Recovery Playbook
## Banque Nationale de Mauritanie — Plan de Continuité des Systèmes Critiques

> [!IMPORTANT]
> **Document Officiel — Direction des Systèmes d'Information (DSI)**
> Ce dépôt contient l'ensemble des procédures, configurations et guides techniques
> pour la mise en place et l'exploitation du système de **Disaster Recovery (DR)**
> des bases de données critiques de la BNM.
>
> **Statut :** 🟢 Click DR Opérationnel (94 tables, Lag = 0s) — 🟡 Pleniged DR En cours
> **Go-Live cible :** Fin Avril 2026

---

## 🎯 Objectif du Projet

La BNM opère deux systèmes critiques dont la perte de données serait catastrophique pour les citoyens mauritaniens :

| Système | Rôle Métier | Serveur Source | Technologie |
|:---|:---|:---|:---|
| **Click** | Paiement mobile, transactions financières, rechargements | Ubuntu Linux — `192.168.1.241:3366` | MySQL 8.0 |
| **Pleniged** | Gestion documentaire et archivage GED | Windows 10 — `10.168.2.34:3306` | MySQL 5.7 |

Ce projet garantit que **si le site de production tombe** (panne, sinistre, cyberattaque), les données bancaires sont **immédiatement disponibles** sur un site de secours avec un objectif de :
- **RPO (Recovery Point Objective)** : < 30 secondes (perte de données maximale)
- **RTO (Recovery Time Objective)** : < 5 minutes (temps de reprise maximale)

---

## 🏗️ Architecture Multi-Zones

```
                    ╔══════════════════════════════════════╗
                    ║  ZONE 1 — MONITORING (192.168.1.65)  ║
                    ║  Prometheus · Grafana · Alertmanager  ║
                    ║  Dashboard : http://192.168.1.65:3000 ║
                    ╚═══════════════════════╤══════════════╝
                                            │ Scrape toutes les 10s
              ┌─────────────────────────────┼──────────────────────────┐
              ▼                             ▼                          ▼
╔═════════════════════╗     ╔══════════════════════╗    ╔══════════════════════╗
║  ZONE 0 — PROD      ║     ║  ZONE 2 — DR PRI     ║    ║  ZONE 3 — DR SEC     ║
║  192.168.1.241      ║     ║  192.168.1.66        ║    ║  192.168.1.64        ║
║                     ║     ║                      ║    ║                      ║
║  ⚙️  MySQL Natif    ║     ║  🐳 mysql-click      ║    ║  🐳 mysql-click      ║
║     Port: 3366      ║──▶  ║     Port: 3306       ║    ║     Port: 3306       ║
║                     ║  └─▶║  🐳 mysql-pleniged   ║    ║  🐳 mysql-pleniged   ║
║  (Win) MySQL 5.7    ║     ║     Port: 3307       ║    ║     Port: 3307       ║
║  10.168.2.34:3306   ║     ║                      ║    ║                      ║
║                     ║     ║  📡 exporter: 9104   ║    ║  📡 exporter: 9104   ║
║  🐳 exporter: 9104  ║     ║  📡 exporter: 9105   ║    ║  📡 exporter: 9105   ║
╚═════════════════════╝     ╚══════════════════════╝    ╚══════════════════════╝
      │ Source des données         ▲ Réplication GTID         ▲ Réplication GTID
      └────────────────────────────┴──────────────────────────┘
```

---

## 🛠️ Stack Technique

| Technologie | Version | Rôle dans la Stratégie DR |
|:---|:---|:---|
| **MySQL 8.0** | 8.0 | Moteur de base de données principal Click (Production & DR) |
| **MySQL 5.7** | 5.7 | Moteur de base de données Pleniged (Production Windows & DR) |
| **GTID** | — | Protocol de réplication sans faille — identifie chaque transaction de manière unique |
| **Docker Engine** | 24+ | Conteneurisation des instances DR pour garantir l'isolation et la portabilité |
| **Docker Compose** | 1.29+ | Orchestration de plusieurs conteneurs sur chaque site |
| **mysqld-exporter** | 0.19 | Capteur de métriques MySQL → expose `/metrics` en HTTP |
| **Prometheus** | Latest | Système de collecte et stockage des métriques temporelles |
| **Grafana** | 12.4+ | Tableau de bord de visualisation et d'alerting |
| **Alertmanager** | Latest | Gestionnaire d'alertes (routing vers email/Slack) |

---

## 📂 Bases de Données Protégées

| Base | Système | Tables | Taille | Statut DR |
|:---|:---|:---:|:---:|:---|
| **`click`** | Click MySQL 8.0 | 94 | 73 Mo | 🟢 Répliqué (Lag=0s) |
| **`clickTest`** | Click MySQL 8.0 | 94 | 19 Mo | 🟢 Répliqué (Lag=0s) |
| **`clickTestPub`** | Click MySQL 8.0 | — | — | 🟢 Répliqué (Lag=0s) |
| **`mail_otp`** | Click MySQL 8.0 | — | — | 🟢 Répliqué (Lag=0s) |
| **`plenesgi_ged`** | Pleniged MySQL 5.7 | — | ~949 Mo | 🟡 Importé, réplication en attente |

---

## 📊 État du Monitoring (7/7 Cibles Actives)

| Job Prometheus | Cible | Statut | Rôle |
|:---|:---|:---:|:---|
| `BNM_PROD_CLICK_ZONE0` | `192.168.1.241:9104` | 🟢 UP | Production Click surveillée |
| `BNM_DR_CLICK_ZONE2` | `192.168.1.66:9104` | 🟢 UP | DR Click Primaire |
| `BNM_DR_CLICK_ZONE3` | `192.168.1.64:9104` | 🟢 UP | DR Click Secondaire |
| `BNM_DR_PLENIGED_ZONE2` | `192.168.1.66:9105` | 🟢 UP | DR Pleniged Primaire |
| `BNM_DR_PLENIGED_ZONE3` | `192.168.1.64:9105` | 🟢 UP | DR Pleniged Secondaire |

---

## 📖 Navigation dans le Playbook

| # | Guide | Contenu |
|:---|:---|:---|
| **01** | [Stratégie DR](01_dr_architecture_strategy.md) | RPO/RTO, GTID, architecture globale |
| **02** | [Config Maître](02_master_config_guide.md) | Configuration des serveurs source |
| **03** | [Mise en Service](03_start_replication_guide.md) | Magic Sync, docker-compose, démarrage |
| **04** | [Validation](04_validation_replication_guide.md) | Tests "Grand Direct", vérifications |
| **05** | [Troubleshooting](05_dr_troubleshooting_knowledge_base.md) | Toutes les erreurs connues et solutions |
| **06** | [Monitoring](06_monitoring_guide.md) | Prometheus, Grafana, alertes |
| **07** | [Infrastructure & Images Docker](07_infrastructure_docker_guide.md) | Détails de chaque serveur et conteneur |
| **08** | [Utilisateurs & Accès](08_users_access_guide.md) | Tous les utilisateurs OS, MySQL, Grafana |

---

## 🔐 Identifiants — Référence Rapide

> [!CAUTION]
> Ce tableau est confidentiel. Ne pas partager en dehors de la DSI.

| Composant | Utilisateur | Mot de passe | Accès |
|:---|:---|:---|:---|
| MySQL Click DR | `root` | `RootPassword123!` | Admin complet |
| MySQL Réplication | `replicator` | `ReplicaPass2026!` | REPLICATION SLAVE uniquement |
| MySQL Monitoring Prod | `monitor` | `MonitorPass2026!` | Lecture seule |
| Grafana | `admin` | `GrafanaPass2026!` | `http://192.168.1.65:3000` |
