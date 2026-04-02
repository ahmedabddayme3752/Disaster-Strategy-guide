# 🏦 BNM Disaster Recovery Playbook
## Banque Nationale de Mauritanie — Plan de Continuité des Systèmes Critiques

> [!IMPORTANT]
> **Document Officiel — Direction des Systèmes d'Information (DSI)**
> Ce dépôt contient l'ensemble des procédures, configurations et guides techniques
> pour la mise en place et l'exploitation du système de **Disaster Recovery (DR)**
> des bases de données critiques de la BNM.
>
> **Statut :** 🟢 Click DR Opérationnel — Monitoring 7/7 Zones Actif
> **Go-Live cible :** Fin Avril 2026

---

## 🎯 Objectif du Projet

La BNM opère deux systèmes critiques dont la perte de données serait catastrophique :

| Système | Rôle | Serveur Source |
|:---|:---|:---|
| **Click** | Paiement mobile et transactions financières | Ubuntu Linux — `192.168.1.241:3366` |
| **Pleniged** | Gestion documentaire et archivage | Windows 10 — `10.168.2.34:3306` |

Ce projet garantit que **si le site de production tombe** (panne, sinistre, cyberattaque), les données bancaires sont **immédiatement disponibles** sur un site de secours.

---

## 🛠️ Stack Technique Complète

### Bases de Données
| Technologie | Version | Rôle |
|:---|:---|:---|
| **MySQL** | 8.0 | Moteur de base de données relationnel |
| **GTID** | — | Global Transaction Identifiers — Garantit l'intégrité de la réplication |
| **Binary Log** | ROW format | Journal de toutes les modifications pour la réplication |

### Infrastructure & Conteneurisation
| Technologie | Version | Rôle |
|:---|:---|:---|
| **Docker Engine** | 24+ | Isolation des instances MySQL (Click/Pleniged séparés) |
| **Docker Compose** | 1.29+ | Orchestration multi-conteneurs sur chaque site DR |

### Monitoring (Zone 1 — 192.168.1.65)
| Technologie | Version | Port | Rôle |
|:---|:---|:---|:---|
| **mysqld-exporter** | 0.19.0 | 9104/9105 | Capteur — Collecte les métriques MySQL |
| **Prometheus** | Latest | 9090 | Collecteur — Stocke les métriques |
| **Grafana** | Latest | 3000 | Dashboard — Visualisation temps réel |
| **Alertmanager** | Latest | 9093 | Alertes — Notifie l'équipe DSI |

---

## 🏗️ Architecture Multi-Zones (Architecture Finale)

```
┌──────────────────────────────────────────────────────────────────────┐
│  ZONE 1 — MONITORING (192.168.1.65)                                  │
│  🐳 Prometheus (9090) + Grafana (3000) + Alertmanager (9093)         │
│  Dashboard : http://192.168.1.65:3000                                │
└──────────────┬──────────────────────────────────────────────────────-┘
               │ Scrape métriques (toutes les 10s)
    ┌──────────┼──────────────────────┐
    ▼          ▼                      ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ ZONE 0 — PROD    │  │ ZONE 2 — DR PRI  │  │ ZONE 3 — DR SEC  │
│ 192.168.1.241    │  │ 192.168.1.66     │  │ 192.168.1.64     │
│                  │  │                  │  │                  │
│ ⚙️  MySQL Natif  │  │ 🐳 mysql-click   │  │ 🐳 mysql-click   │
│    Port: 3366    │  │    Port: 3306    │  │    Port: 3306    │
│                  │  │ 🐳 mysql-plenigd │  │ 🐳 mysql-plenigd │
│ 🐳 exporter      │  │    Port: 3307    │  │    Port: 3307    │
│    Port: 9104    │  │ 📡 export: 9104  │  │ 📡 export: 9104  │
│ (--network=host) │  │ 📡 export: 9105  │  │ 📡 export: 9105  │
└────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
         │                     ▲                      ▲
         │    Réplication GTID │                      │
         └─────────────────────┴──────────────────────┘
```

---

## 📊 État Actuel du Monitoring (7/7 Cibles Actives)

| Job Prometheus | Cible | Statut |
|:---|:---|:---|
| `BNM_PROD_CLICK_ZONE0` | `192.168.1.241:9104` | 🟢 UP |
| `BNM_DR_CLICK_ZONE2` | `192.168.1.66:9104` | 🟢 UP |
| `BNM_DR_CLICK_ZONE3` | `192.168.1.64:9104` | 🟢 UP |
| `BNM_DR_PLENIGED_ZONE2` | `192.168.1.66:9105` | 🟢 UP |
| `BNM_DR_PLENIGED_ZONE3` | `192.168.1.64:9105` | 🟢 UP |
| `mysql_zone2_click_dr` | `192.168.1.66:9104` | 🟢 UP |
| `mysql_zone3_pleniged_dr` | `192.168.1.64:9105` | 🟢 UP |

---

## 📖 Navigation dans le Playbook

| # | Guide | Contenu | État |
|:---|:---|:---|:---|
| **01** | [Stratégie DR](01_dr_architecture_strategy.md) | RPO/RTO, GTID, pourquoi Docker | ✅ |
| **02** | [Config Maître](02_master_config_guide.md) | Port 3366, utilisateurs, pare-feu | ✅ |
| **03** | [Mise en Service DR](03_start_replication_guide.md) | docker-compose, "Magic Sync" | ✅ |
| **04** | [Validation](04_validation_replication_guide.md) | Tests "Grand Direct", intégrité | ✅ |
| **05** | [Troubleshooting](05_dr_troubleshooting_knowledge_base.md) | Toutes les erreurs résolues | ✅ |
| **06** | [Monitoring](06_monitoring_guide.md) | Prometheus, Grafana, Zone 0 | ✅ |

---

## 🚨 Procédure de Failover (Sinistre Majeur)

> [!CAUTION]
> Un Failover est une opération **IRRÉVERSIBLE** à court terme.
> Obtenir l'accord **écrit** de la DSI avant toute action.

```bash
# Étape 1 — Vérifier le dernier GTID sur Zone 2
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "SHOW SLAVE STATUS\G"

# Étape 2 — Promouvoir Zone 2 en Maître
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "
STOP SLAVE;
SET GLOBAL read_only = 0;
"

# Étape 3 — Rediriger les applications vers 192.168.1.66:3306
```

---

## 🔐 Référence des Identifiants DR

| Composant | Utilisateur | Mot de passe | Notes |
|:---|:---|:---|:---|
| MySQL Click DR (root) | root | RootPassword123! | Conteneurs Zone 2/3 |
| MySQL Réplication | replicator | ReplicaPass2026! | Droits REPLICATION SLAVE |
| MySQL Monitoring Zone 0 | monitor | MonitorPass2026! | Droits lecture seule |
| Grafana | admin | GrafanaPass2026! | Dashboard BNM |
