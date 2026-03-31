# Guide 4 — Validation de la Réplication MySQL

> 👤 **Responsable : Ahmed + Ami (en collaboration)**
> ⏱️ **À faire après :** `03_start_replication_guide.md` terminé avec succès
> 🎯 **Objectif :** Confirmer que les données se répliquent correctement en temps réel entre les sources et les VMs DR

---

## Test 1 — Vérification de l'état de la réplication

Avant tout, vérifiez que les deux réplications sont actives :

### Zone 2 (Click DR)
```bash
ssh primbackupdr@192.168.1.66
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "SHOW SLAVE STATUS\G" | grep -E "Running|Behind|Error"
```

### Zone 3 (Pleniged DR)
```bash
ssh secbackupdr@192.168.1.64
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "SHOW SLAVE STATUS\G" | grep -E "Running|Behind|Error"
```

> ✅ Les deux doivent afficher `Slave_IO_Running: Yes` et `Slave_SQL_Running: Yes`

---

## Test 2 — Validation Click (Ubuntu → Zone 2)

### Étape A : Insérer des données sur la source Click
L'ami exécute ceci sur `192.168.1.241` :
```bash
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "
USE click_test;
CREATE TABLE IF NOT EXISTS dr_validation (
  id INT AUTO_INCREMENT PRIMARY KEY,
  message VARCHAR(200),
  inserted_at DATETIME DEFAULT NOW()
);
INSERT INTO dr_validation (message) VALUES ('Test-1 : Réplication Click → Zone2 OK');
INSERT INTO dr_validation (message) VALUES ('Test-2 : RPO quasi nul confirmé');
SELECT * FROM dr_validation;
"
```

### Étape B : Vérifier immédiatement sur Zone 2
Ahmed exécute ceci sur `192.168.1.66` :
```bash
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "
SELECT * FROM click_test.dr_validation;
"
```

> ✅ **Résultat attendu :** Les deux lignes insérées sur Click apparaissent instantanément sur Zone 2.
> ❌ Si rien n'apparaît, attendez 5 secondes et réessayez. Si toujours rien → problème de réplication.

### Étape C : Mesurer le délai (Seconds_Behind_Master)
```bash
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! \
  -e "SHOW SLAVE STATUS\G" | grep Seconds_Behind_Master
```
> ✅ La valeur doit être `0` ou très proche de `0`.

---

## Test 3 — Validation Pleniged (Windows → Zone 3)

### Étape A : Insérer des données sur la source Pleniged
L'ami exécute ceci sur `10.168.2.34` (PowerShell ou CMD) :
```powershell
docker exec -it mysql-pleniged mysql -u root -pRootPassword123! -e "USE pleniged_test; CREATE TABLE IF NOT EXISTS dr_validation (id INT AUTO_INCREMENT PRIMARY KEY, message VARCHAR(200), inserted_at DATETIME DEFAULT NOW()); INSERT INTO dr_validation (message) VALUES ('Test-1 : Réplication Pleniged -> Zone3 OK'); SELECT * FROM dr_validation;"
```

### Étape B : Vérifier immédiatement sur Zone 3
Ahmed exécute ceci sur `192.168.1.64` :
```bash
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "
SELECT * FROM pleniged_test.dr_validation;
"
```

> ✅ La ligne doit apparaître quasi instantanément.

---

## Test 4 — Test de charge (optionnel mais recommandé)

Insérez 1000 lignes rapidement pour vérifier que la réplication reste stable sous charge :

### Sur Click source (`192.168.1.241`)
```bash
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "
USE click_test;
DROP PROCEDURE IF EXISTS insert_test_data;
DELIMITER //
CREATE PROCEDURE insert_test_data()
BEGIN
  DECLARE i INT DEFAULT 1;
  WHILE i <= 1000 DO
    INSERT INTO dr_validation (message) VALUES (CONCAT('Charge test ligne ', i));
    SET i = i + 1;
  END WHILE;
END //
DELIMITER ;
CALL insert_test_data();
SELECT COUNT(*) AS total_lignes FROM dr_validation;
"
```

### Vérifier le compte sur Zone 2
```bash
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "
SELECT COUNT(*) AS total_lignes FROM click_test.dr_validation;
"
```
> ✅ Le total doit être identique sur les deux serveurs (1002 lignes : 2 tests + 1000 charge).

---

## Test 5 — Vérifier le plugin Semi-Sync

Confirmez que la réplication semi-synchrone est bien active (et pas seulement asynchrone) :

### Sur la source Click (`192.168.1.241`)
```bash
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "
SHOW STATUS LIKE 'Rpl_semi_sync_master%';
"
```

> ✅ Résultats attendus :
> ```
> Rpl_semi_sync_master_status      : ON
> Rpl_semi_sync_master_clients     : 1     ← Zone 2 est bien connectée
> Rpl_semi_sync_master_yes_tx      : > 0   ← Transactions confirmées en semi-sync
> ```

---

## Tableau de résultats — À remplir

| Test | Click → Zone 2 | Pleniged → Zone 3 | Statut |
|------|--------------|-----------------|--------|
| Slave_IO_Running | [ ] Yes | [ ] Yes | |
| Slave_SQL_Running | [ ] Yes | [ ] Yes | |
| Seconds_Behind_Master | [ ] 0 | [ ] 0 | |
| Données test apparaissent | [ ] Oui | [ ] Oui | |
| Semi-sync clients = 1 | [ ] Oui | [ ] Oui | |
| Test charge 1000 lignes | [ ] OK | [ ] OK | |

> ✅ Si toutes les cases sont cochées → Réplication validée avec succès !

**Suite :** Passez au guide `05_monitoring_guide.md` pour configurer Prometheus et Grafana sur Zone 1.
