# Suivi des Tâches : Projet DR MySQL (Click & Pleniged)

Ce document trace l'avancement global du projet. Les tâches cochées `[x]` sont terminées, celles notées `[/]` sont en cours, et `[ ]` sont à faire.

## 1. Préparation documentaire (Terminé)
- [x] Créer l'architecture réseau et les rôles (`README.md`, `server_vm_zone.txt`)
- [x] Créer les guides d'installation (`mysql_docker_guide.md`, `machines_source_guide.md`)
- [x] Créer les guides opérationnels (`03_start_replication`, `04_validation`, `05_monitoring`, `06_failover`)
- [x] Corriger les configurations (`GET_MASTER_PUBLIC_KEY=1` pour l'authentification SSL)

## 2. Configuration des VMs DR — Zone 2 & Zone 3 (Ahmed)
- [x] Installer Docker et Docker Compose
- [x] Lancer les conteneurs MySQL `mysql-server`
- [x] Gérer les configurations `my.cnf` (avec `server-id` à 2 et 3)
- [x] Créer les utilisateurs système `replicator`

## 3. Configuration des Machines Sources (L'Ami)
- [x] Connecter Click (Ubuntu - `192.168.1.241`) et installer MySQL via Docker
- [ ] Connecter Pleniged (Windows 10 - `10.168.2.34`) et installer Docker Desktop
- [x] Activer les logs binaires et le mode GTID sur Click
- [x] Créer l'utilisateur de réplication sur Click
- [x] Communiquer les coordonnées de `SHOW MASTER STATUS` pour Click

## 4. Finalisation du Monitoring — Zone 1 (Ahmed)
- [x] Résoudre le conflit de ports (arrêt de l'ancien Grafana/Alertmanager)
- [x] Lancer la stack Docker `monitoring` (Prometheus, Grafana, Alertmanager)
- [ ] *En attente* : Installer `mysqld_exporter` **sur les Serveurs Zone 2 et Zone 3** (Étape 2 du guide `05_monitoring_guide.md`)

## 5. Démarrage de la Réplication (Ahmed)
- [x] Récupérer les `File` et `Position` (ou GTID) de Click
- [x] Exécuter la commande `CHANGE MASTER TO` sur Zone 2 pointant vers Click
- [ ] Exécuter la commande `CHANGE MASTER TO` sur Zone 3 pointant vers Pleniged
- [x] Exécuter `START SLAVE;` sur la VM Zone 2 (Click)

## 6. Validation Finale & Tests (Équipe)
- [ ] Insérer des lignes de test sur Click et Pleniged (L'ami)
- [ ] Vérifier la présence en temps réel sur Zone 2 et Zone 3 (Ahmed)
- [ ] S'assurer que le tableau de bord Grafana sur `192.168.1.65:3000` est tout vert
- [ ] **Tests hors-prod :** Simuler un Crash et exécuter le Failover (Guide 06)
