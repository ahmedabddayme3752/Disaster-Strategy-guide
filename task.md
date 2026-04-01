# Suivi des Tâches : Architecture DR "Double-Conteneur"

## 1. Préparation des Serveurs DR (Nettoyage)
- [x] Zone 2 (192.168.1.66) : `docker-compose down` et nettoyage des dossiers ✅
- [x] Zone 3 (192.168.1.64) : `docker-compose down` et nettoyage des dossiers ✅

## 2. Lancement de la nouvelle Architecture (Ahmed)
- [x] Zone 2 : Lancement de `mysql-click` (:3306) et `mysql-pleniged` (:3307)
- [x] Zone 3 : Lancement de `mysql-click` (:3306) et `mysql-pleniged` (:3307)
- [x] Zone 2 : Configuration des utilisateurs de réplication sur les deux instances
- [x] Zone 3 : Configuration des utilisateurs de réplication sur les deux instances

## 3. Mise en place de la Réplication (Ahmed)
- [x] Connecter **Zone 2 Click** ➡️ Source 241 (Semi-Sync) ✅
- [ ] Connecter **Zone 2 Pleniged** ➡️ Source 34 (Semi-Sync)
- [x] Connecter **Zone 3 Click** ➡️ Source 241 (Asynchrone) ✅
- [ ] Connecter **Zone 3 Pleniged** ➡️ Source 34 (Asynchrone)

## 4. Validation & Monitoring
- [ ] Installer `mysqld_exporter` sur les 4 instances DR
- [ ] Tester l'insertion de données sur Click et voir la réplication en cascade (Source ➡️ Z2 ➡️ Z3)
- [ ] Tester l'insertion de données sur Pleniged et voir la réplication en cascade (Source ➡️ Z2 ➡️ Z3)
- [ ] Vérifier les graphiques sur Zone 1 (Port 3000)
