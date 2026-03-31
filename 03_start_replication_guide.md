# Guide 3 — Démarrage de la Réplication MySQL

> 👤 **Responsable : Ahmed**
> ⏱️ **À faire après :** `machines_source_guide.md` ET `mysql_docker_guide.md` tous les deux terminés
> 🎯 **Objectif :** Connecter les VMs DR aux machines sources et démarrer la réplication Semi-Synchrone

---

## Prérequis — Checklist avant de commencer

Vérifiez que tout est prêt avant de lancer cette étape :

- [ ] Zone 2 (`192.168.1.66`) — MySQL Docker en statut `Up` ✅
- [ ] Zone 3 (`192.168.1.64`) — MySQL Docker en statut `Up` ✅
- [ ] Click source (`192.168.1.241`) — MySQL Docker en statut `Up` ✅
- [ ] Pleniged source (`10.168.2.34`) — MySQL Docker en statut `Up` ✅
- [ ] L'ami vous a transmis : `File` et `Position` de `SHOW MASTER STATUS` pour les deux sources
- [ ] Port 3306 ouvert dans les firewalls (des deux côtés)

---

## Étape 0 — Tester la connectivité réseau

Avant tout, vérifiez que Zone 2 peut parler à Click, et Zone 3 peut parler à Pleniged.

### Depuis Zone 2 → vers Click
```bash
ssh primbackupdr@192.168.1.66

# Test de connectivité vers Click
sudo docker exec -it mysql-server mysql \
  -h 192.168.1.241 \
  -u replicator \
  -pReplicaPass2026! \
  -e "SHOW MASTER STATUS;"
```

### Depuis Zone 3 → vers Pleniged
```bash
ssh secbackupdr@192.168.1.64

# Test de connectivité vers Pleniged
sudo docker exec -it mysql-server mysql \
  -h 10.168.2.34 \
  -u replicator \
  -pReplicaPass2026! \
  -e "SHOW MASTER STATUS;"
```

> ✅ Si vous voyez un résultat avec `File` et `Position` → la connexion est établie, continuez.
> ❌ Si vous voyez `ERROR 2003 (HY000): Can't connect` → problème réseau ou firewall, ne continuez pas.

---

## Étape 1 — Démarrer la réplication sur Zone 2 (→ Click)

Connectez-vous à Zone 2 :
```bash
ssh primbackupdr@192.168.1.66
cd ~/mysql-dr
```

Lancez la commande complète :
```bash
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "
-- Activer le plugin semi-sync côté esclave
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET PERSIST rpl_semi_sync_slave_enabled = 1;

-- Pointer vers la source Click
CHANGE MASTER TO
  MASTER_HOST='192.168.1.241',
  MASTER_USER='replicator',
  MASTER_PASSWORD='ReplicaPass2026!',
  MASTER_AUTO_POSITION=1;

-- Démarrer la réplication
START SLAVE;
"
```

### Vérifier l'état immédiatement
```bash
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "SHOW SLAVE STATUS\G"
```

Cherchez ces trois lignes :
```
Slave_IO_Running: Yes        ← doit être Yes
Slave_SQL_Running: Yes       ← doit être Yes
Seconds_Behind_Master: 0     ← doit être 0 (ou très petit)
```

---

## Étape 2 — Démarrer la réplication sur Zone 3 (→ Pleniged)

Connectez-vous à Zone 3 :
```bash
ssh secbackupdr@192.168.1.64
cd ~/mysql-dr
```

Lancez la commande complète :
```bash
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "
-- Activer le plugin semi-sync côté esclave
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET PERSIST rpl_semi_sync_slave_enabled = 1;

-- Pointer vers la source Pleniged
CHANGE MASTER TO
  MASTER_HOST='10.168.2.34',
  MASTER_USER='replicator',
  MASTER_PASSWORD='ReplicaPass2026!',
  MASTER_AUTO_POSITION=1;

-- Démarrer la réplication
START SLAVE;
"
```

### Vérifier l'état immédiatement
```bash
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "SHOW SLAVE STATUS\G"
```

---

## Résolution des erreurs courantes

### ❌ Erreur : `Slave_IO_Running: Connecting`
La réplication essaie de se connecter mais ne peut pas. Vérifiez :
```bash
# 1. Vérifier que le firewall source autorise la connexion
# 2. Tester la connexion réseau (voir Étape 0)
# 3. Vérifier le message d'erreur exact :
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! \
  -e "SHOW SLAVE STATUS\G" | grep "Last_IO_Error"
```

### ❌ Erreur : `Got fatal error 1236 from master`
Le point de départ GTID est incohérent. Réinitialisez :
```bash
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "
STOP SLAVE;
RESET SLAVE ALL;
RESET MASTER;
"
# Puis relancez la commande CHANGE MASTER TO de l'étape 1 ou 2
```

### ❌ Erreur : `Slave_SQL_Running: No`
Une requête SQL a échoué lors de l'application. Vérifiez :
```bash
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! \
  -e "SHOW SLAVE STATUS\G" | grep "Last_SQL_Error"
```

---

## ✅ Résultat attendu final

Une fois les deux réplications démarrées avec succès :

| VM DR | Source | `Slave_IO_Running` | `Slave_SQL_Running` | `Seconds_Behind_Master` |
|-------|--------|--------|-------|------|
| Zone 2 (`192.168.1.66`) | Click (`192.168.1.241`) | `Yes` | `Yes` | `0` |
| Zone 3 (`192.168.1.64`) | Pleniged (`10.168.2.34`) | `Yes` | `Yes` | `0` |

**Suite :** Passez au guide `04_validation_replication_guide.md` pour valider que les données se répliquent correctement.
