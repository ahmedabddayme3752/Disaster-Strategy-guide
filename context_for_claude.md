# Project Context for Claude

Hello Claude! I am working on the **Disaster Recovery (DR) MySQL Implementation for Banque Nationale de Mauritanie (BNM)**. Here is a comprehensive overview of the project, architecture, and current state to give you full context for our session.

## 📋 1. Project Description
This project implements a complete **Disaster Recovery (DR)** architecture for the critical MySQL databases at BNM:
- **Click** — Mobile Payment System (Ubuntu Linux, `192.168.1.241`)
- **Pleniged** — Document Management System (Windows 10, `10.168.2.34`)

**Goal:** In the event of a major failure (server, network, or data center crash), we must be able to automatically switch to a secondary site in under **2 hours (RTO < 2h)** with almost zero data loss (**RPO ≈ 0**). This is achieved using MySQL Semi-Synchronous Replication (native on Prod zones, Docker on DR zones) across several zones.

## 🗺️ 2. Architecture & Zones

The infrastructure is split into several network zones:

| Zone | IP Address | Server Name | Role | MySQL Installed |
|------|------------|-------------|------|-----------------|
| **Zone 1** | `192.168.1.65` | `monotordr` | 🔍 **Monitoring** (Prometheus & Grafana) | ❌ No |
| **Zone 2** | `192.168.1.66` | `primbackupdr` | 🟢 **DR Primary** | ✅ Yes (Docker) |
| **Zone 3** | `192.168.1.64` | `secbackupdr` | 🟡 **DR Standby** | ✅ Yes (Docker) |
| **Zone 4** | *TBD* | *TBD* | 🔴 **Remote DR (Geo-separated)** | ⏳ Later |
| **Prod 1** | `192.168.1.241`| `Click` (Ubuntu) | 📤 **Source Production** | ✅ Yes (Native) |
| **Prod 2** | `10.168.2.34` | `Pleniged` (Windows)|📤 **Source Production** | ✅ Yes (Native) |

### Replication Data Flow

```text
Click Source (server-id=10)    ──Semi-Sync TCP 3306──▶  Zone 2 DR (server-id=2)
Pleniged Source (server-id=20) ──Semi-Sync TCP 3306──▶  Zone 3 DR (server-id=3)
Zone 2 DR (Primary)            ──Semi-Sync TCP 3306──▶  Zone 3 DR (Standby)
```
*(Zone 4 will receive asynchronous replication later)*

## 📂 3. Available Documentation / File Tracker

My current directory contains the following guides. Feel free to ask me to provide the contents of any specific file if you need granular details:

1. `README.md` : High-level overview of the project, including team members, IP addresses, and firewall requirements.
2. `mysql_docker_guide.md` : Step-by-step guide for bringing up MySQL containers on Zone 2 & Zone 3 (Primary/Standby DR).
3. `machines_source_guide.md` : Step-by-step guide for configuring Native MySQL replication on the production source machines (Click and Pleniged).
4. `03_start_replication_guide.md` : How to initialize replication from sources to DR zones.
5. `04_validation_replication_guide.md` : Details for validating that replication is running correctly (`SHOW SLAVE STATUS`).
6. `05_monitoring_guide.md` : Installation and configuration of Prometheus, Grafana, and `mysqld_exporter` on Zone 1.
7. `06_failover_test_guide.md` : Runbook for simulating a disaster and promoting a standby node.
8. `main.tex` / `mysql_dr_plan.tex` : Formal LaTeX project documentation and implementation plans (in French).
9. `server_vm_zone.txt` : Server IPs and credential inventories.

## 🛠️ 4. Technical Details

- **MySQL Version:** 8.0 (Containerized via Docker on DR zones, Native on Prod zones)
- **Replication Type:** Semi-Synchronous (`semisync_master`/`semisync_slave`) + GTID-based replication (`gtid_mode=ON`).
- **Monitoring Architecture:** `mysqld_exporter` via Prometheus, visualized in Grafana.
- **Failover / Runbook:** Stop slave, reset slave, disable read-only mode, redirect traffic.

---

**Claude, based on this context, I am ready for your assistance. I will now give you my specific request...**
