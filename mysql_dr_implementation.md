# Implementation Plan: MySQL DR (Click & Pleniged)

## 1. Overview
This document outlines the step-by-step implementation for the Disaster Recovery (DR) architecture concerning the **Click** and **Pleniged** MySQL environments. It breaks down the prerequisites, current infrastructure, and executable tasks suitable for Jira import.

## 2. What We Have
**Production Environment (Zone 0):**
- **Click**: MySQL 8.0 running on Ubuntu Linux (32 GB RAM, 16 vCPUs, DB Size: 0.1 GB).
- **Pleniged DB**: MySQL 8.0 running on Windows (32 GB RAM, 16 vCPUs, DB Size: ~32 GB).
- **Pleniged FS**: File system containing images and PDFs (~4 TB).
- **Monitoring**: Observability tools including Prometheus and Grafana.

## 3. What We Need
**Disaster Recovery Environment (Target):**
- **DR VMs**: Allocation of target virtual machines mirroring production specs (Ubuntu for Click, Windows for Pleniged).
- **Network rules**: Firewall configurations allowing TCP 3306 (MySQL) and SSH/Rsync (Port 22/TCP).
- **Deployment Software**: 
  - Percona XtraBackup (for binary hot backups).
  - Rsync (for Pleniged file system sync).
- **Replication**: MySQL Semi-Synchronous replication enabling Primary to DR syncing.

## 4. Jira Epics and Tasks
Here is the granular breakdown of tasks that can be bulk-imported into Jira for the engineering team.

### Epic 1: DR Infrastructure Preparation
- **Task 1.1**: Provision DR Virtual Machine for **Click** (Ubuntu Linux, 32 GB RAM, 16 vCPUs). (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 1.2**: Provision DR Virtual Machine for **Pleniged** (Windows, 32 GB RAM, 16 vCPUs). (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 1.3**: Configure Network and Firewall to allow TCP 3306 (MySQL) from Primary to DR VMs. (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 1.4**: Configure Network and Firewall to allow SSH/Rsync traffic for Pleniged FS. (Assignee: Ahmed Abd Dayem Ahmed Bouha)

### Epic 2: Click Environment DR Setup (Ubuntu/MySQL)
- **Task 2.1**: Install MySQL 8.0 on the Click DR virtual machine. (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 2.2**: Install Percona XtraBackup on Click Primary and DR VMs. (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 2.3**: Execute preliminary full backup on Primary using XtraBackup and restore on DR VM. (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 2.4**: Configure and enable MySQL Semi-Synchronous Replication from Primary to DR. (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 2.5**: Verify Replication status, sync gaps, and replication delay (RPO check). (Assignee: Ahmed Abd Dayem Ahmed Bouha)

### Epic 3: Pleniged Environment DR Setup (Windows/MySQL)
- **Task 3.1**: Install MySQL 8.0 on the Pleniged DR virtual machine. (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 3.2**: Configure Percona XtraBackup (or verified Windows-compatible tool) on Pleniged Primary. (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 3.3**: Perform initial database backup from Primary and restore to Pleniged DR. (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 3.4**: Configure and enable MySQL Semi-Synchronous Replication from Primary to DR. (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 3.5**: Set up scheduled nightly Rsync job for the Pleniged FS (approx. 4 TB). (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 3.6**: Verify replication status and Rsync nighttime job completion logs. (Assignee: Ahmed Abd Dayem Ahmed Bouha)

### Epic 4: Testing & Handover
- **Task 4.1**: Perform a DR failover and fallback test for the Click environment. (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 4.2**: Perform a DR failover and fallback test for the Pleniged database and File System. (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 4.3**: Integrate DR VMs and replication status into Prometheus and Grafana alerting. (Assignee: Ahmed Abd Dayem Ahmed Bouha)
- **Task 4.4**: Conduct final review, secure sign-off from Team Lead, and update internal runbooks. (Assignee: Ahmed Abd Dayem Ahmed Bouha)
