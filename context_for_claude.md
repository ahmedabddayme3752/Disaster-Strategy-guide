# Project Context for Claude (Updated Architecture)

Hello Claude! I am working on the **Disaster Recovery (DR) MySQL Implementation for Banque Nationale de Mauritanie (BNM)**. Here is the updated architecture for our project.

## 📋 1. Project Description
We are implementing a **Double-Instance (Double-Conteneur)** DR architecture. Instead of dedicating one zone to one project, both **Zone 2** and **Zone 3** will now host BOTH **Click** and **Pleniged** databases.

- **Click** — Port 3306 (Ubuntu Source: `192.168.1.241`)
- **Pleniged** — Port 3307 (Windows Source: `10.168.2.34`)

**Goal:** RTO < 2h and RPO ≈ 0 using a hybrid Semi-Sync/Async approach.

## 🗺️ 2. Architecture & Zones

| Zone | IP Address | Role | MySQL Containers |
|------|------------|------|-----------------|
| **Zone 1** | `192.168.1.65` | 🔍 Monitoring | ❌ No |
| **Zone 2** | `192.168.1.66` | 🟢 **Primary DR** | Click (:3306) & Pleniged (:3307) |
| **Zone 3** | `192.168.1.64` | 🟡 **Standby DR** | Click (:3306) & Pleniged (:3307) |

### Replication Strategy
1.  **Source ➡️ Zone 2**: **Semi-Synchronous** (High Data Integrity).
2.  **Source ➡️ Zone 3**: **Asynchronous** Parallel (Performance & Redundancy).

## 📂 3. Documentation Suite
All files have been updated to reflect this port-based isolation:
- `mysql_docker_guide.md`: Multi-container `docker-compose.yml` config.
- `03_start_replication_guide.md`: Parallel replication commands.
- `05_monitoring_guide.md`: Dual `mysqld_exporter` per VM (Ports 9104 & 9105).
- `task.md`: Current progress tracker for Click and Pleniged.

## 🛠️ 4. Status
- **Click**: Replication is UP and VALIDATED on both Zone 2 and Zone 3.
- **Pleniged**: Ready for setup on port 3307.
- **Monitoring**: Ready for deployment with the new dual-exporter config.

---
**Claude, the architecture is now fully documented. Please use this "Double-Instance" context for all future requests.**
