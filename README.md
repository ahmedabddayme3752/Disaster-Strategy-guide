# 🏦 Projet DR MySQL — Banque Nationale de Mauritanie (BNM)

> **Direction Système d'Information — BNM**
> **Statut :** 🟢 En cours d'exécution
> **Go-Live cible :** Fin Avril 2026

---

## 📋 Description du projet

Ce projet vise à mettre en place une architecture complète de **Disaster Recovery (DR)** pour les bases de données MySQL des systèmes critiques de la BNM :

- **Click** — Système de paiement mobile (Ubuntu Linux, `192.168.1.241`)
- **Pleniged** — Système de gestion documentaire (Windows 10, `10.168.2.34`)

En cas de panne majeure (serveur, réseau, salle des serveurs), l'objectif est de basculer automatiquement vers un site secondaire en moins de **2 heures** avec une perte de données quasi nulle (**RPO ≈ 0**).

---

## 👥 Équipe

| Rôle | Nom | Responsabilité |
|------|-----|----------------|
| **Titulaire** | Ing. Betah Lab | Supervision technique, validation |
| **Titulaire** | Dr. Fatimetou Abdou | Tests failover/fallback, validation RPO/RTO |
| **Stagiaire MySQL** | Ahmed Abd Dayem Ahmed Bouha | Configuration VMs DR (Zone 2, Zone 3) |
| **Stagiaire Oracle** | Ethmane Mahmoud Ethmane | Track Oracle (séparé de ce projet) |

---

## 🗺️ Architecture des Zones

```
┌─────────────────────────────────────────────────────────────────────┐
│                        RÉSEAU BNM                                   │
│                                                                     │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐      │
│   │   Zone 1     │     │   Zone 2     │     │   Zone 3     │      │
│   │  Monitoring  │     │  MySQL DR    │     │  MySQL DR    │      │
│   │              │     │  (Primary)   │     │  (Standby)   │      │
│   │ 192.168.1.65 │     │192.168.1.66  │     │192.168.1.64  │      │
│   │ monotordr    │     │primbackupdr  │     │secbackupdr   │      │
│   └──────┬───────┘     └──────▲───────┘     └──────▲───────┘      │
│          │                    │ Semi-Sync           │ Semi-Sync    │
│          │ Surveille          │ Réplication         │ Réplication  │
│          ▼                    │                     │              │
│   ┌──────────────┐            │                     │              │
│   │  Prometheus  │     ┌──────┴───────┐     ┌──────┴───────┐      │
│   │  + Grafana   │     │  Click       │     │  Pleniged    │      │
│   │  Alertmanager│     │  Ubuntu      │     │  Windows 10  │      │
│   └──────────────┘     │192.168.1.241 │     │ 10.168.2.34  │      │
│                        │  (Source)    │     │  (Source)    │      │
│                        └──────────────┘     └──────────────┘      │
│                                                                     │
│   Zone 4 (Remote DR géo-séparé) — À configurer ultérieurement     │
└─────────────────────────────────────────────────────────────────────┘
```

### Détail des zones

| Zone | IP | Serveur | Rôle | MySQL |
|------|----|---------|------|-------|
| **Zone 1** | `192.168.1.65` | `monotordr` | 🔍 Monitoring (Prometheus, Grafana) | ❌ Non |
| **Zone 2** | `192.168.1.66` | `primbackupdr` | 🟢 DR Primary | ✅ Docker |
| **Zone 3** | `192.168.1.64` | `secbackupdr` | 🟡 DR Standby | ✅ Docker |
| **Zone 4** | TBD | TBD | 🔴 Remote DR géo-séparé | ⏳ Plus tard |
| **Click** | `192.168.1.241` | Machine Ubuntu | 📤 Source Production | ✅ Docker |
| **Pleniged** | `10.168.2.34` | Machine Windows 10 | 📤 Source Production | ✅ Docker |

---

## 🎯 Objectifs RPO / RTO

| Système | RPO | RTO | Méthode |
|---------|-----|-----|---------|
| Click DB | ≈ 0 | < 2h | Réplication MySQL Semi-Synchrone |
| Pleniged DB | ≈ 0 | < 2h | Réplication MySQL Semi-Synchrone |

---

## 📁 Fichiers du projet

| Fichier | Description | Destinataire |
|---------|-------------|--------------|
| `README.md` | Ce fichier — vue d'ensemble du projet | Toute l'équipe |
| `server_vm_zone.txt` | Inventaire des IPs, utilisateurs et mots de passe des serveurs | Toute l'équipe |
| `mysql_architecture.md` | Diagramme Mermaid de l'architecture DR MySQL | Toute l'équipe |
| `mysql_docker_guide.md` | Guide technique — Configuration des **VMs DR** (Zone 2 & Zone 3) avec Docker | **Ahmed** |
| `machines_source_guide.md` | Guide technique — Configuration des **machines sources** (Click & Pleniged) avec Docker | **Ami de Ahmed** |
| `mysql_dr_plan.tex` | Document LaTeX complet (en français) — Architecture + Plan d'exécution + Runbook | Direction SI |
| `mysql_dr_implementation.md` | Résumé Markdown des épiques et tâches Jira | Équipe projet |
| `mysql_dr_implementation.tex` | Version LaTeX du plan d'implémentation | Direction SI |
| `main.tex` | Document LaTeX principal — Plan DR global BNM | Direction SI |

---

## 🚀 Guide de démarrage rapide

### Division du travail — Travailler en parallèle

Le projet se divise en **deux tracks parallèles** :

#### 👤 Ahmed (VMs DR)
1. Lire **`mysql_docker_guide.md`**
2. Configurer Zone 2 (`192.168.1.66`) comme MySQL Primary DR
3. Configurer Zone 3 (`192.168.1.64`) comme MySQL Standby DR

#### 👥 Ami de Ahmed (Machines Sources)
1. Lire **`machines_source_guide.md`**
2. Configurer Click Ubuntu (`192.168.1.241`) avec Docker — `server-id = 10`
3. Configurer Pleniged Windows (`10.168.2.34`) avec Docker — `server-id = 20`

#### 🔗 Étape finale (ensemble)
Une fois les deux tracks terminés, démarrer la réplication :
- Click (`192.168.1.241`) → Zone 2 (`192.168.1.66`)
- Pleniged (`10.168.2.34`) → Zone 3 (`192.168.1.64`)

---

## 🔁 Flux de réplication

```
Click Source (server-id=10)    ──Semi-Sync TCP 3306──▶  Zone 2 DR (server-id=2)
Pleniged Source (server-id=20) ──Semi-Sync TCP 3306──▶  Zone 3 DR (server-id=3)
Zone 2 DR                      ──Async (plus tard)───▶  Zone 4 Remote DR
Zone 3 DR                      ──Async (plus tard)───▶  Zone 4 Remote DR
```

---

## ⚠️ Règles réseau à ouvrir (Firewall)

| Source | Destination | Port | Usage |
|--------|-------------|------|-------|
| `192.168.1.241` (Click) | `192.168.1.66` (Zone 2) | TCP 3306 | Réplication MySQL |
| `10.168.2.34` (Pleniged) | `192.168.1.64` (Zone 3) | TCP 3306 | Réplication MySQL |
| `192.168.1.66` (Zone 2) | `192.168.1.65` (Zone 1) | TCP 9104 | Prometheus scrape |
| `192.168.1.64` (Zone 3) | `192.168.1.65` (Zone 1) | TCP 9104 | Prometheus scrape |

---

## 📞 Procédure d'urgence (résumé)

En cas de panne de la machine source :

1. Aller sur la **VM DR correspondante** (Zone 2 pour Click, Zone 3 pour Pleniged)
2. Vérifier l'état de la réplication :
   ```bash
   sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "SHOW SLAVE STATUS\G"
   ```
3. Promouvoir le DR en Primary :
   ```bash
   sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "STOP SLAVE; RESET SLAVE ALL; SET GLOBAL read_only=0;"
   ```
4. Rediriger le trafic applicatif vers l'IP de la VM DR
5. Contacter **Ing. Betah Lab** et ouvrir un ticket Jira d'incident

> 📖 Voir le **Runbook complet** dans `mysql_dr_plan.tex` — Section 8.
