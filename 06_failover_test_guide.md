# Guide 6 — Test de Basculement (Failover) et Retour (Fallback)

Ce guide permet de tester la résilience de votre infrastructure DR Double-Instance. 

---

## 🔥 Test 1 — Panne du site Click Production (`.241`)

### A. Simuler la panne
Demandez à votre ami d'arrêter le conteneur sur la source :
```bash
sudo docker stop mysql-click
```

### B. Activation du secours sur Zone 2 (`.66`)
Sur votre machine Zone 2, promouvez le conteneur Click en mode Écriture :
```bash
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "
STOP SLAVE;
SET GLOBAL read_only = 0;
SET GLOBAL super_read_only = 0;
"
```
> ✅ **Zone 2** est maintenant le Maître pour Click. Vos applications peuvent pointer vers `192.168.1.66:3306`.

---

## ❄️ Test 2 — Retour à la normale (Fallback Click)

### A. Redémarrer la source et synchroniser depuis Zone 2
Une fois la machine `.241` réparée :
```bash
# Sur Source 241
sudo docker start mysql-click
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "
CHANGE MASTER TO MASTER_HOST='192.168.1.66', MASTER_USER='replicator', MASTER_PASSWORD='ReplicaPass2026!', MASTER_AUTO_POSITION=1;
START SLAVE;
"
```

### B. Inverser les rôles (revenir à l'état initial)
Une fois synchronisé, remettez Click (`.241`) en Maître et Zone 2 en Esclave comme avant.

---

## 🔥 Test 3 — Panne du site Pleniged Production (`.34`)

Même procédure, mais sur le port **3307** :

### A. Activation du secours sur Zone 2 (`.66`)
```bash
sudo docker exec -it mysql-pleniged mysql -u root -pRootPassword123! -e "
STOP SLAVE;
SET GLOBAL read_only = 0;
SET GLOBAL super_read_only = 0;
"
```
> ✅ **Zone 2** est maintenant le Maître pour Pleniged. Vos applications peuvent pointer vers `192.168.1.66:3307`.

---

## 📊 Tableau de bord de validation DR

| Système | Source | Backup Primary (Z2) | Backup Standby (Z3) |
| :--- | :--- | :--- | :--- |
| **Click** | `192.168.1.241:3306` | `192.168.1.66:3306` | `192.168.1.64:3306` |
| **Pleniged** | `10.168.2.34:3306` | `192.168.1.66:3307` | `192.168.1.64:3307` |
