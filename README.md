# 🏦 BNM Disaster Recovery Playbook
## Banque Nationale de Mauritanie — Plan de Continuité des Systèmes Critiques

> [!IMPORTANT]
> **Document Officiel — Direction des Systèmes d'Information (DSI)**
> Ce dépôt contient l'ensemble des procédures, configurations et guides techniques
> pour la mise en place et l'exploitation du système de **Disaster Recovery (DR)**
> des bases de données critiques de la BNM.
>
> **Statut :** 🟢 Click DR Opérationnel — Monitoring Actif
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

### Monitoring
| Technologie | Version | Rôle |
|:---|:---|:---|
| **mysqld-exporter** | 0.19.0 | Capteur — Collecte les métriques MySQL et les expose via HTTP |
| **Prometheus** | Latest | Collecteur — Récupère les métriques de tous les capteurs |
| **Grafana** | Latest | Dashboard — Affiche les métriques en temps réel |
| **Alertmanager** | Latest | Alertes — Notifie l'équipe en cas de problème |

---

## 🏗️ Architecture Multi-Zones

```
┌──────────────────────────────────────────────────────────────────┐
│                    ZONE 1 - MONITORING                           │
│              Prometheus + Grafana + Alertmanager                 │
│                      192.168.1.65                                │
└────────────────────────┬─────────────────────────────────────────┘
                         │ Collecte les métriques (scrape)
           ┌─────────────┴──────────────┐
           ▼                            ▼
┌──────────────────────┐    ┌──────────────────────┐
│  ZONE 2 - DR PRIMARY │    │ ZONE 3 - DR SECONDARY│
│   primbackupdr       │    │   secbackupdr        │
│   192.168.1.66       │    │   192.168.1.64       │
│                      │    │                      │
│ 🐳 mysql-click:3306  │    │ 🐳 mysql-click:3306  │
│ 🐳 mysql-plenigd:3307│    │ 🐳 mysql-plenigd:3307│
│ 📡 exporter:9104     │    │ 📡 exporter:9104     │
│ 📡 exporter:9105     │    │ 📡 exporter:9105     │
└──────────┬───────────┘    └──────────┬───────────┘
           │ Réplication GTID          │ Réplication GTID
           └─────────────┬─────────────┘
                         ▲
           ┌─────────────┴──────────────┐
           │                            │
┌──────────────────────┐    ┌──────────────────────┐
│  Click Production    │    │ Pleniged Production  │
│  Ubuntu Linux        │    │  Windows 10          │
│  192.168.1.241:3366  │    │  10.168.2.34:3306    │
│  MySQL 8.0 (Natif)   │    │  MySQL 8.0 (Natif)   │
└──────────────────────┘    └──────────────────────┘
```

---

## 📖 Navigation dans le Playbook

| # | Guide | Contenu | État |
|:---|:---|:---|:---|
| **01** | [Stratégie DR](01_dr_architecture_strategy.md) | RPO/RTO, choix techniques, GTID expliqué | ✅ |
| **02** | [Configuration Maître](02_master_config_guide.md) | Config Production (ports, GTID, utilisateurs) | ✅ |
| **03** | [Mise en Service DR](03_start_replication_guide.md) | Déploiement Docker, "Magic Sync" | ✅ |
| **04** | [Validation Données](04_validation_replication_guide.md) | Tests "Grand Direct", intégrité | ✅ |
| **05** | [Troubleshooting](05_dr_troubleshooting_knowledge_base.md) | Solutions aux erreurs rencontrées | ✅ |
| **06** | [Monitoring](06_monitoring_guide.md) | Prometheus, Grafana, Dashboard BNM | ✅ |

---

## 🚨 Procédure de Failover (Sinistre Majeur)

> [!CAUTION]
> Un Failover est une opération **IRRÉVERSIBLE** à court terme.
> Obtenir l'accord de la DSI avant toute action.

**Étapes en cas de perte totale de la Production :**

1. **Vérifier** le dernier GTID synchronisé sur Zone 2 :
   ```bash
   sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "SHOW SLAVE STATUS\G"
   ```
2. **Promouvoir** Zone 2 en Maître :
   ```bash
   sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "STOP SLAVE; SET GLOBAL read_only=0;"
   ```
3. **Rediriger** les applications vers `192.168.1.66:3306` (Click) ou `192.168.1.66:3307` (Pleniged).
4. **Notifier** l'équipe via le canal d'urgence BNM.
