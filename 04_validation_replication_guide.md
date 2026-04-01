# Guide 4 — Validation de la Réplication (Architecture Double-Instance)

Ce guide permet de vérifier que les données de Click et Pleniged arrivent bien sur vos deux serveurs de secours (Zone 2 et Zone 3).

---

## 1. Vérification de l'état des 4 réplications

Exécutez ces commandes pour vérifier que tous les tuyaux sont ouverts (**`Yes` / `Yes`**) :

### Sur Zone 2 (`192.168.1.66`)
```bash
# Vérifier Click
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "SHOW SLAVE STATUS\G" | grep -E "Running|Behind|Master_Host"
# Vérifier Pleniged
sudo docker exec -it mysql-pleniged mysql -u root -pRootPassword123! -e "SHOW SLAVE STATUS\G" | grep -E "Running|Behind|Master_Host"
```

### Sur Zone 3 (`192.168.1.64`)
```bash
# Vérifier Click
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "SHOW SLAVE STATUS\G" | grep -E "Running|Behind|Master_Host"
# Vérifier Pleniged
sudo docker exec -it mysql-pleniged mysql -u root -pRootPassword123! -e "SHOW SLAVE STATUS\G" | grep -E "Running|Behind|Master_Host"
```

---

## 2. Test fonctionnel : Click (Ubuntu ➡️ DR)

### A. Insérer sur la Source (`.241`)
Demandez à votre ami d'insérer une ligne :
```bash
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "INSERT INTO click_test.dr_test (message) VALUES ('Validation Click Parallel DR - $(date)');"
```

### B. Vérifier sur Zone 2 et Zone 3
```bash
# Sur Zone 2
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "SELECT * FROM click_test.dr_test ORDER BY id DESC LIMIT 1;"

# Sur Zone 3
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "SELECT * FROM click_test.dr_test ORDER BY id DESC LIMIT 1;"
```

---

## 3. Test fonctionnel : Pleniged (Windows ➡️ DR)

### A. Insérer sur la Source (`.34`)
Demandez à votre ami d'insérer une ligne (PowerShell) :
```powershell
docker exec -it mysql-pleniged mysql -u root -pRootPassword123! -e "INSERT INTO pleniged_test.dr_test (message) VALUES ('Validation Pleniged Parallel DR');"
```

### B. Vérifier sur Zone 2 et Zone 3
```bash
# Sur Zone 2
sudo docker exec -it mysql-pleniged mysql -u root -pRootPassword123! -e "SELECT * FROM pleniged_test.dr_test ORDER BY id DESC LIMIT 1;"

# Sur Zone 3
sudo docker exec -it mysql-pleniged mysql -u root -pRootPassword123! -e "SELECT * FROM pleniged_test.dr_test ORDER BY id DESC LIMIT 1;"
```

---

## 4. Vérification de la Synchronisation (Semi-Sync vs Async)

### Sur la Source Click (`.241`)
```bash
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "SHOW STATUS LIKE 'Rpl_semi_sync_master_clients';"
```
> ✅ **Résultat attendu : `1`**. 
> Seule la Zone 2 est enregistrée comme client Semi-Sync. La Zone 3 est invisible ici car elle est en mode Asynchrone (meilleur pour la performance).
