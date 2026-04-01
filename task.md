# Suivi des Tâches : Architecture DR "Double-Conteneur"

## 1. Préparation des Serveurs DR (Nettoyage)
- [ ] Zone 2 (192.168.1.66) : `docker-compose down` et nettoyage des dossiers ✅
- [ ] Zone 3 (192.168.1.64) : `docker-compose down` et nettoyage des dossiers ✅

## 2. Lancement de la nouvelle Architecture (Ahmed)
- [/] Zone 2 : Lancement de `mysql-click` (:3306) et `mysql-pleniged` (:3307)
- [ ] Zone 3 : Lancement de `mysql-click` (:3306) et `mysql-pleniged` (:3307)
- [ ] Zone 2 : Configuration des utilisateurs de réplication sur les deux instances
- [ ] Zone 3 : Configuration des utilisateurs de réplication sur les deux instances

## 3. Mise en place de la Réplication (Ahmed)
- [ ] Connecter **Click** (Z2 ➡️ Source 241)
- [ ] Connecter **Pleniged** (Z2 ➡️ Source 34)
- [ ] Connecter **Zone 3 Click** ➡️ Zone 2 Click
- [ ] Connecter **Zone 3 Pleniged** ➡️ Zone 2 Pleniged

## 4. Validation & Monitoring
- [ ] Installer `mysqld_exporter` sur les 4 instances DR
- [ ] Tester l'insertion de données sur Click et voir la réplication en cascade (Source ➡️ Z2 ➡️ Z3)
- [ ] Tester l'insertion de données sur Pleniged et voir la réplication en cascade (Source ➡️ Z2 ➡️ Z3)
- [ ] Vérifier les graphiques sur Zone 1 (Port 3000)
