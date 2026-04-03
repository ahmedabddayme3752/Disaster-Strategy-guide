---
description: How to update the BNM Disaster Recovery context when resuming work
---

# When starting a new conversation about BNM DR

## Step 1: Read the full context
Always read `.agents/bnm_dr_context.md` first. This file contains:
- Complete network topology (5 zones, all IPs)
- All credentials (MySQL, Grafana, OS users)
- Docker infrastructure per zone
- Current status of all replications
- Key file locations
- Known bugs and solutions

## Step 2: Read the task list
Read `task.md` to see what is done and what still needs to be done.

## Step 3: Ask Ahmed what he wants to do
After reading context, ask: "Which zone or task would you like to work on today?"

## Key rules
- Always use `set +H` before any command with `!` in passwords
- Always do `sudo docker rm -f CONTAINER` before `docker-compose up -d` (ContainerConfig bug)
- MySQL root password on DR: `RootPassword123!`
- Grafana admin password: `GrafanaPass2026!`
- Never ignore databases - replicate ALL bases (no --replicate-ignore-db)
- All monitoring is centralized on Zone 1 (192.168.1.65)
