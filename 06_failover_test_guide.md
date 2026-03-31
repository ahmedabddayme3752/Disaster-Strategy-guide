# Guide 6 — Test de Basculement (Failover) et Retour (Fallback)

> 👤 **Responsable : Dr. Fatimetou Abdou + Ahmed**
> ⏱️ **À faire après :** `05_monitoring_guide.md` — monitoring opérationnel
> 🎯 **Objectif :** Simuler une panne réelle et valider que le basculement vers les VMs DR fonctionne dans le délai cible (RTO < 2h)

---

> ⚠️ **ATTENTION :** Ces tests doivent être effectués en dehors des heures de production. Prévenez la Direction SI avant de commencer.

---

## Test 1 — Failover Click (Source → Zone 2)

### Phase A : Préparation

Notez l'heure de début du test :
```bash
echo "=== DÉBUT TEST FAILOVER CLICK ===" 
date
```

Vérifiez que la réplication est stable :
```bash
ssh primbackupdr@192.168.1.66
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! \
  -e "SHOW SLAVE STATUS\G" | grep -E "Running|Behind"
```
> ✅ Les deux doivent être `Yes` et `Seconds_Behind_Master: 0` avant de commencer.

### Phase B : Simulation de panne (arrêt de la source Click)

L'ami arrête MySQL sur la source Click (`192.168.1.241`) :
```bash
# Sur Click source
sudo docker stop mysql-click
# ou simuler une coupure réseau
sudo iptables -I INPUT -p tcp --dport 3306 -j DROP
```

Observez immédiatement l'alerte sur Grafana (`http://192.168.1.65:3000`).

### Phase C : Promotion de Zone 2 en Primary

Ahmed exécute sur Zone 2 (`192.168.1.66`) :
```bash
ssh primbackupdr@192.168.1.66

sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "
-- Arrêter l'esclave et détacher de la source
STOP SLAVE;
RESET SLAVE ALL;

-- Promouvoir en Primary (autoriser les écritures)
SET GLOBAL read_only = 0;
SET GLOBAL super_read_only = 0;

-- Vérifier le statut
SHOW MASTER STATUS;
SELECT @@read_only, @@super_read_only;
"
```

> ✅ `read_only` doit être `0` — Zone 2 est maintenant le Primary.

### Phase D : Rediriger le trafic applicatif

Mettez à jour le DNS ou la configuration applicative pour pointer vers `192.168.1.66:3306` au lieu de `192.168.1.241:3306`.

> ⚠️ Cette étape dépend de votre application Click. Contactez l'équipe applicative si nécessaire.

### Phase E : Mesurer le RTO

```bash
echo "=== FIN BASCULEMENT CLICK ==="
date
# Calculez la différence avec l'heure de début → RTO réel
```

> ✅ **RTO cible : < 2 heures**

### Phase F : Vérifier les données (pas de perte)

```bash
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "
SELECT COUNT(*) as total FROM click_test.dr_validation;
SELECT MAX(inserted_at) as derniere_donnee FROM click_test.dr_validation;
"
```

> Comparez avec le nombre de lignes sur la source avant la panne — elles doivent être identiques.

---

## Test 2 — Fallback Click (Retour en production)

Une fois la source Click réparée, retournez en production.

### Phase A : Redémarrer la source Click
```bash
# Sur Click source
sudo docker start mysql-click
# ou supprimer la règle iptables
sudo iptables -D INPUT -p tcp --dport 3306 -j DROP
```

### Phase B : Re-synchroniser la source depuis Zone 2

Sur Click source (`192.168.1.241`) — reconfigurer comme esclave de Zone 2 temporairement :
```bash
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "
CHANGE MASTER TO
  MASTER_HOST='192.168.1.66',
  MASTER_USER='replicator',
  MASTER_PASSWORD='ReplicaPass2026!',
  MASTER_AUTO_POSITION=1,
  MASTER_SSL=0,
  GET_MASTER_PUBLIC_KEY=1;
START SLAVE;
"

# Attendre que Seconds_Behind_Master = 0
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! \
  -e "SHOW SLAVE STATUS\G" | grep Seconds_Behind_Master
```

### Phase C : Inverser les rôles (retour initial)

Une fois la source re-synchronisée :
```bash
# Sur Click source → redevient Primary
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "
STOP SLAVE;
RESET SLAVE ALL;
SET GLOBAL read_only = 0;
"

# Sur Zone 2 → redevient Standby DR
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "
CHANGE MASTER TO
  MASTER_HOST='192.168.1.241',
  MASTER_USER='replicator',
  MASTER_PASSWORD='ReplicaPass2026!',
  MASTER_AUTO_POSITION=1,
  MASTER_SSL=0,
  GET_MASTER_PUBLIC_KEY=1;
START SLAVE;
SET GLOBAL read_only = 1;
"
```

> ✅ La réplication est revenue à son état initial : Click → Zone 2.

---

## Test 3 — Failover Pleniged (Source → Zone 3)

Répétez les mêmes étapes que le Test 1, mais pour Pleniged :

| Élément | Failover Click | Failover Pleniged |
|---------|---------------|------------------|
| Source à arrêter | `mysql-click` (`192.168.1.241`) | `mysql-pleniged` (`10.168.2.34`) |
| VM DR à promouvoir | Zone 2 (`192.168.1.66`) | Zone 3 (`192.168.1.64`) |
| Base à vérifier | `click_test.dr_validation` | `pleniged_test.dr_validation` |

---

## Tableau des résultats des tests

| Test | Heure début | Heure fin | RTO réel | RTO cible | Résultat |
|------|-------------|-----------|----------|-----------|----------|
| Failover Click → Zone 2 | | | | < 2h | [ ] ✅ / ❌ |
| Fallback Zone 2 → Click | | | | < 2h | [ ] ✅ / ❌ |
| Failover Pleniged → Zone 3 | | | | < 2h | [ ] ✅ / ❌ |
| Fallback Zone 3 → Pleniged | | | | < 2h | [ ] ✅ / ❌ |

---

## Checklist de validation finale du projet

- [ ] Réplication Click → Zone 2 : `Slave_IO_Running: Yes`
- [ ] Réplication Pleniged → Zone 3 : `Slave_IO_Running: Yes`
- [ ] Monitoring Prometheus actif sur Zone 1
- [ ] Alertes Grafana configurées (réplication arrêtée, délai > 30s)
- [ ] Test failover Click : RTO < 2h ✅
- [ ] Test fallback Click : retour en production OK ✅
- [ ] Test failover Pleniged : RTO < 2h ✅
- [ ] Test fallback Pleniged : retour en production OK ✅
- [ ] Document de validation signé par Ing. Betah Lab et Dr. Fatimetou Abdou
- [ ] Ticket Jira de clôture créé

> 🎉 **Si toutes les cases sont cochées → Le projet DR MySQL est opérationnel !**

---

## Prochaine étape : Zone 4 (Remote DR Géo-séparé)

Une fois ce projet validé, la Zone 4 (site distant géo-séparé) sera configurée avec une réplication **asynchrone** depuis Zone 2 et Zone 3. Un guide dédié sera créé quand les ressources Zone 4 seront disponibles.
