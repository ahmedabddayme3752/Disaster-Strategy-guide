# 🏦 BNM Disaster Recovery Playbook
<<<<<<< HEAD
## Banque Nationale de Mauritanie — Plan de Continuité des Systèmes Critiques

> [!IMPORTANT]
> **Document Officiel — Direction des Systèmes d'Information (DSI)**
> Ce dépôt contient l'ensemble des procédures, configurations et guides techniques
> pour la mise en place et l'exploitation du système de **Disaster Recovery (DR)**
> des bases de données critiques de la BNM.
>
> **Statut :** 🟢 Click DR (94 tables) Opérationnel — Monitoring 7/7 Zones Actif
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

## 🛠️ Stack Technique
- **Moteurs DB :** MySQL 8.0 (Click) et MySQL 5.7 (Pleniged)
- **Isolation :** Docker Engine v24+ & Docker Compose v2.0+
- **Protocole :** Réplication basée sur les **GTID** (Global Transaction Identifiers)
- **Sécurité :** Authentification `mysql_native_password` & Chiffrement RSA
- **Filtrage :** Bypass des bases de test (`--replicate-ignore-db=click`)

---

## 📂 Bases de Données Protégées (Mirroring Total)

Nous répliquons l'intégralité des 4 bases critiques identifiées sur la Production :

| Base | Description | Système |
|:---|:---|:---|
| **`click`** | **Production Réelle Click** | Click (MySQL 8.0) |
| **`clickTest`** | Copie de test Production | Click (MySQL 8.0) |
| **`clickTestPub`** | Base de test Publique | Click (MySQL 8.0) |
| **`plenesgi_ged`**| **Production Documentaire** | Pleniged (MySQL 5.7) |

---

## 📊 État Actuel du Monitoring (7/7 Cibles Actives)

| Job Prometheus | Cible | Statut | Rôle |
|:---|:---|:---:|:---|
| `BNM_PROD_CLICK_ZONE0` | `192.168.1.241:9104` | 🟢 UP | Source (Production) |
| `BNM_DR_CLICK_ZONE2` | `192.168.1.66:9104` | 🟢 UP | DR Click Primaire |
| `BNM_DR_CLICK_ZONE3` | `192.168.1.64:9104` | 🟢 UP | DR Click Secondaire |
| `BNM_DR_PLENIGED_ZONE2` | `192.168.1.66:9105` | 🟢 UP | DR Pleniged Primaire |
| `BNM_DR_PLENIGED_ZONE3` | `192.168.1.64:9105` | 🟢 UP | DR Pleniged Secondaire |

---

## 📖 Navigation dans le Playbook

| # | Guide | Contenu | État |
|:---|:---|:---|:---|
| **01** | [Stratégie DR](01_dr_architecture_strategy.md) | Détails du GTID et RPO/RTO | ✅ |
| **02** | [Config Maître](02_master_config_guide.md) | Port 3366, users, pare-feu | ✅ |
| **03** | [Mise en Service DR](03_start_replication_guide.md) | Guide du Magic Sync v2 (Full Mirror) | ✅ |
| **04** | [Validation](04_validation_replication_guide.md) | Procédures de test "Grand Direct" | ✅ |
| **05** | [Troubleshooting](05_dr_troubleshooting_knowledge_base.md) | Base de connaissances des erreurs | ✅ |
| **06** | [Monitoring](06_monitoring_guide.md) | Guide Prometheus/Grafana complet | ✅ |
| **07** | [Guide Pleniged](pleniged_windows_dr_guide.md) | Configuration spécifique Windows 5.7 | ✅ |

---

## 🔐 Référence des Identifiants DR

| Composant | Utilisateur | Mot de passe | Notes |
|:---|:---|:---|:---|
| MySQL Click DR (root) | root | RootPassword123! | Conteneurs Zone 2/3 |
| MySQL Réplication | replicator | ReplicaPass2026! | Droits REPLICATION SLAVE |
| MySQL Monitoring Zone 0 | monitor | MonitorPass2026! | Droits lecture seule |
| Grafana | admin | GrafanaPass2026! | http://192.168.1.65:3000 |
