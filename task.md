# Suivi des Tâches : Architecture DR "Double-Conteneur"

## 1. Préparation des Serveurs DR (Nettoyage)
- [x] Zone 2 (192.168.1.66) : `docker-compose down` et nettoyage ✅
- [x] Zone 3 (192.168.1.64) : `docker-compose down` et nettoyage ✅

## 2. Lancement Architecture Double-Conteneur (Ahmed)
- [x] Zone 2 : `mysql-click` (:3306) + `mysql-pleniged` (:3307) lancés ✅
- [x] Zone 3 : `mysql-click` (:3306) + `mysql-pleniged` (:3307) lancés ✅
- [x] Zone 2 & 3 : Utilisateurs de réplication configurés ✅

---

## 3. Réplication CLICK ✅ TERMINÉ
- [x] Zone 2 Click ➡️ Source 241 (Semi-Sync) ✅
- [x] Zone 3 Click ➡️ Source 241 (Async) ✅

---

## 4. Réplication PLENIGED — EN COURS 🔧

### Infos terrain confirmées
- MySQL **5.7.33** natif, géré par **Laragon** dans `C:\PLENISOFTS\`
- Admin : `plenesgi_ged` (`RootBNM2026!`)
- Une base : `plenesgi_ged`
- Guide : `pleniged_windows_dr_guide.md`

### Partie A — Source Windows (10.168.2.34) ✅ TERMINÉ
- [x] A1 — Utilisateur `replicator` créé (ReplicaPass2026!) ✅
- [x] A2 — `my.ini` modifié (server-id=20, GTID, BinLog, 1 base) ✅
- [x] A3 — Redémarrer MySQL avec succès ✅
- [x] A4 — Vérifier GTID=ON, log_bin=ON, server_id=20 ✅
- [x] A5 — Dump de `plenesgi_ged` généré (=> 949.95 MB) ✅
- [x] A6 — Ouvrir Firewall Windows (Z2:66, Z3:64, Z1:65) ✅
- [x] A7 — GTID_PURGED identifié (`968d1d1d-a6b5-11ec-a0c0-d83bbfaee82b:1-5`) ✅

### Partie B — DR Ahmed (Zone 2 & Zone 3) EN ATTENTE
- [ ] B1 — Tester connectivité vers 10.168.2.34:3306
- [ ] B2 — Extraire GTID_PURGED du dump (`968d1d1d-a6b5-11ec-a0c0-d83bbfaee82b:1-5`)
- [ ] B3 — Importer `plenesgi_ged` sur Z2 et Z3
- [ ] B4 — Lancer réplication Async Zone 2 (Port 3307)
- [ ] B5 — Lancer réplication Async Zone 3 (Port 3307)
- [ ] B6 — Test d'insertion (Windows ➡️ Z2 + Z3)

---

## 5. Monitoring — EN ATTENTE
- [ ] Installer `mysqld_exporter` (×2 par VM) — voir `05_monitoring_guide.md`
- [ ] Créer user `prometheus` sur les 4 instances Pleniged (B7 du guide)
- [ ] Vérifier graphiques Grafana sur Zone 1 (Port 3000)

---

## 🗂️ Guides de référence
| Fichier | Usage |
|---|---|
| `pleniged_windows_dr_guide.md` | Guide complet Pleniged (terrain réel) |
| `05_monitoring_guide.md` | Monitoring Prometheus + exporteurs |
| `03_start_replication_guide.md` | Référence réplication Click |
| `06_failover_test_guide.md` | Test de basculement DR |
