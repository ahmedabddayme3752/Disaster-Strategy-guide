# 09 — Plan de Failover & Procédures d'Urgence
## Que faire quand Zone 0 (Production) tombe en panne ?

> [!CAUTION]
> Ce document est le **manuel d'urgence** de la BNM. Il doit être imprimé et accessible même sans internet.
> Chaque membre de l'équipe DSI doit connaître les 10 premières minutes de procédure par cœur.

---

## 🚨 SCÉNARIO : Zone 0 Production Click (.241) tombe en panne

### Ce qui se passe AUTOMATIQUEMENT (sans intervention humaine)

```
T+0s    : Le serveur 192.168.1.241 tombe (panne réseau, matériel, OS...)
           │
T+10s   : mysqld-exporter ne répond plus → mysql_up{Zone0} = 0
           │
T+1min  : Prometheus détecte la panne (évaluation des règles)
           │
T+1min  : Alerte "ProductionDown" passe en état FIRING 🔴
           │
T+1min  : Alertmanager envoie la notification (email/Slack DSI)
           │
T+1min  : Grafana Dashboard → panneau "Production Click" devient ROUGE
           │
T+???   : Les sites DR Zone 2 et Zone 3 continuent à attendre des
           données de réplication... mais la Production ne répond plus.
           → Slave_IO_Running: Connecting (en attente de reconnexion)
           → Les données sur Zone 2 et Zone 3 sont FIGÉES à l'instant T+0s
           → AUCUNE nouvelle donnée n'est répliquée
           │
T+???   : Les applications Click (mobile, paiement) commencent à échouer
           car elles pointent vers 192.168.1.241:3366 (mort)
```

### ⚠️ Ce qui NE se passe PAS automatiquement

```
❌ Les applications ne basculent PAS automatiquement vers Zone 2
❌ Les utilisateurs ne sont PAS redirigés automatiquement
❌ La réplication ne "répare" pas toute seule
✋ INTERVENTION HUMAINE OBLIGATOIRE
```

---

## ⏱️ Chronologie de l'Intervention (RTO Cible : < 5 minutes)

```
T+0min  ─── 🔴 PANNE DÉTECTÉE ───────────────────────────────────
             • Alertmanager envoie l'alerte au DSI
             • Grafana montre Production = ROUGE

T+1min  ─── 🔍 VÉRIFICATION ────────────────────────────────────
             • Confirmer que c'est bien Zone 0 qui est morte
             • Vérifier que Zone 2 a bien les données à jour
             • Décision : Activer le Failover vers Zone 2 ?

T+2min  ─── ⚙️  BASCULE APPLICATIVE ──────────────────────────
             • Changer l'IP de connexion dans les apps Click
             • Pointer vers 192.168.1.66:3306 (Zone 2)
             • OU changer le DNS interne

T+3min  ─── ✅ VÉRIFICATION POST-BASCULE ─────────────────────
             • Tester une connexion depuis l'app
             • Vérifier les logs d'erreur

T+5min  ─── 📢 COMMUNICATION ────────────────────────────────
             • Notifier la direction BNM
             • Prévenir si des transactions ont été perdues (RPO)
```

---

## 🔧 Procédure de Failover Détaillée (Étape par Étape)

### PHASE 1 — Confirmer la Panne (2 minutes max)

```bash
# Depuis n'importe quelle machine du réseau
ping 192.168.1.241

# Tester la connexion MySQL directement
mysql -h 192.168.1.241 -P 3366 -u monitor -p'MonitorPass2026!' -e "SELECT 1;"

# Si les deux échouent → La Production Click est bien morte → Continuer
```

### PHASE 2 — Vérifier l'État des Esclaves DR

```bash
# Se connecter sur Zone 2 (primbackupdr@192.168.1.66)
ssh primbackupdr@192.168.1.66

# Vérifier l'état de la réplication
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "
SHOW SLAVE STATUS\G
" | grep -E "Master_Host|Slave_IO_Running|Slave_SQL_Running|Seconds_Behind|Exec_Master_Log_Pos"
```

**Interpréter le résultat :**

| Situation | Que faire |
|:---|:---|
| `Slave_IO_Running: Connecting` + `Seconds_Behind: 0` avant panne | ✅ Zone 2 est à jour → Activer Failover |
| `Slave_IO_Running: No` + `Seconds_Behind > 0` | ⚠️ Zone 2 a du retard → Évaluer la perte de données |
| `Slave_SQL_Running: No` | 🔴 Problème sur Zone 2 → Utiliser Zone 3 |

### PHASE 3 — Activer le Failover vers Zone 2

> [!WARNING]
> Avant cette étape, vous devez arrêter la réplication sur l'esclave
> pour éviter les conflits si la Production revient plus tard.

```bash
# Sur Zone 2 — Arrêter la réplication et rendre l'esclave indépendant
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "
STOP SLAVE;
RESET SLAVE ALL;  -- Oublie complètement le maître
SET GLOBAL read_only = OFF;  -- Autorise les écritures (esclave → maître)
SELECT @@read_only, @@super_read_only;
"
```

**Resultat attendu :**
```
@@read_only  @@super_read_only
0            0
```
✅ Zone 2 est maintenant un serveur **autonome** qui accepte les écritures !

### PHASE 4 — Rediriger les Applications

Vous avez **2 options** selon votre infrastructure :

#### Option A : Modifier la config de l'application Click (le plus rapide)
```bash
# Selon le fichier de config de l'application Click
# Changer la ligne de connexion de :
DB_HOST=192.168.1.241
DB_PORT=3366
# Vers :
DB_HOST=192.168.1.66
DB_PORT=3306

# Redémarrer l'application Click
```

#### Option B : Modifier le DNS interne du réseau BNM
```bash
# Sur le serveur DNS interne de la BNM
# Changer l'entrée "mysql-click.bnm.mr" de .241 vers .66
# Les applications se reconnectent automatiquement
```

### PHASE 5 — Vérification Finale

```bash
# Tester depuis Zone 2 que les données sont accessibles
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "
SELECT COUNT(*) FROM click.clients LIMIT 1;
SELECT MAX(created_at) FROM click.transactions LIMIT 1;
"
```

---

## 🔄 Procédure de Retour à la Normale (Quand Zone 0 revient)

Quand la Production .241 est réparée, il faut re-synchroniser :

```bash
# ⚠️ ATTENTION : Zone 2 a reçu de nouvelles transactions pendant la panne
# Il faut les copier vers Zone 0 avant de réactiver la réplication !

# Étape 1 : Sur Zone 2, créer un dump des nouvelles données
sudo docker exec mysql-click mysqldump -u root -p'RootPassword123!' \
  --single-transaction \
  --master-data=2 \
  click clickTest clickTestPub mail_otp > ~/failover_recovery_$(date +%Y%m%d_%H%M).sql

# Étape 2 : Transférer vers Zone 0 (quand elle revient)
scp ~/failover_recovery_*.sql connect@192.168.1.241:~/recovery/

# Étape 3 : Importer sur Zone 0
# (Sur Zone 0)
mysql -u root -P 3366 < ~/recovery/failover_recovery_*.sql

# Étape 4 : Reconfigurer la réplication (Zone 2 comme esclave de Zone 0)
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "
CHANGE MASTER TO
  MASTER_HOST='192.168.1.241',
  MASTER_PORT=3366,
  MASTER_USER='replicator',
  MASTER_PASSWORD='ReplicaPass2026!',
  MASTER_AUTO_POSITION=1;
START SLAVE;
"
```

---

## 📋 Checklist Failover Imprimable

```
□ T+0   Alerte reçue (email/Grafana)
□ T+1   Ping .241 échoue → Panne confirmée
□ T+2   Vérifier Zone 2 : SHOW SLAVE STATUS → Seconds_Behind_Master = 0
□ T+3   Zone 2 : STOP SLAVE + RESET SLAVE ALL + read_only=OFF
□ T+4   Changer IP dans config applications
□ T+5   Tester connexion applicative vers Zone 2
□ T+6   Notifier Direction BNM
□ T+7   Documenter heure de panne + heure de reprise + données perdues
□ ----  Quand Zone 0 revient : resynchroniser avant de rebrancher
```

---

## 📊 Évaluation des Bonnes Pratiques — BNM DR

### ✅ Ce qui est bien fait

| Critère | Notre Impl. | Standard |
|:---|:---:|:---:|
| Double site de secours (Zone 2 + Zone 3) | ✅ | ✅ |
| Réplication GTID (sans perte de position) | ✅ | ✅ |
| Monitoring temps réel (Prometheus + Grafana) | ✅ | ✅ |
| Alertes automatiques (Alertmanager) | ✅ | ✅ |
| Isolation par Docker | ✅ | ✅ |
| Utilisateurs à droits limités (monitor, replicator) | ✅ | ✅ |
| Documentation technique complète | ✅ | ✅ |
| RPO < 30 secondes | ✅ | ✅ |

### ⚠️ Points d'amélioration à planifier

| Criticité | Point | Impact | Effort |
|:---|:---|:---:|:---:|
| 🔴 HAUTE | Backups automatiques quotidiens (mysqldump) | Perte totale si DELETE accidentel | Moyen |
| 🔴 HAUTE | Procédure de Failover testée mensuellement | RTO inconnu en vrai sinistre | Faible |
| 🟡 MOYENNE | TLS/SSL sur la réplication | Données en clair sur le LAN | Élevé |
| 🟡 MOYENNE | Alertes email/SMS configurées et testées | Alerte non reçue = panne non détectée | Faible |
| 🟡 MOYENNE | Failover automatique (MySQL Router) | RTO 5min → RTO < 30s | Très élevé |
| 🟢 BASSE | Rétention métriques > 15 jours (Thanos) | Historique anayltique | Élevé |
| 🟢 BASSE | Chiffrement mots de passe (Vault) | Sécurité avancée | Élevé |

---

## 🔔 Plan de Notification d'Urgence

> [!IMPORTANT]
> Remplir ce tableau avec les VRAIS contacts avant le Go-Live !

| Rôle | Nom | Téléphone | Email |
|:---|:---|:---|:---|
| Responsable DSI | ____________ | ____________ | ____________ |
| Admin Système (Astreinte) | ____________ | ____________ | ____________ |
| DBA Principal | ____________ | ____________ | ____________ |
| Direction Générale | ____________ | ____________ | ____________ |

### Script de communication (à lire au téléphone) :
```
"Ici [NOM], équipe DSI BNM. Il est [HEURE].
Le serveur de production Click est en panne depuis [HEURE].
Nous avons activé le site de secours Zone 2 à [HEURE].
Le service a repris à [HEURE]. Perte de données estimée : [X] secondes.
Un rapport complet sera disponible dans 2 heures."
```

---

## 💾 Plan de Sauvegarde Automatique (À Implémenter)

```bash
# Script à mettre dans cron sur Zone 0 tous les jours à 2h du matin
# Sur Zone 0 (connect@192.168.1.241)

cat > ~/scripts/daily_backup.sh << 'ENDSCRIPT'
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M)
BACKUP_DIR=~/backups/mysql

mkdir -p $BACKUP_DIR

# Dump complet avec transaction cohérente (sans interruption)
mysqldump \
  -h 127.0.0.1 -P 3366 \
  -u debian-sys-maint -p$(grep password /etc/mysql/debian.cnf | head -1 | awk '{print $3}') \
  --single-transaction \
  --routines \
  --triggers \
  --all-databases \
  --ignore-database=information_schema \
  --ignore-database=performance_schema \
  | gzip > $BACKUP_DIR/bnm_full_$DATE.sql.gz

# Garder seulement les 7 derniers jours
find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete

echo "✅ Backup $DATE terminé : $(du -sh $BACKUP_DIR/bnm_full_$DATE.sql.gz)"
ENDSCRIPT

chmod +x ~/scripts/daily_backup.sh

# Ajouter au cron
# crontab -e
# 0 2 * * * ~/scripts/daily_backup.sh >> ~/logs/backup.log 2>&1
```

---

## 🎯 Note Globale du Projet DR BNM

```
╔══════════════════════════════════════════════════════════════╗
║  Infrastructure DR BNM — Évaluation                         ║
║                                                              ║
║  Réplication & RPO        : ██████████ 10/10  ✅ Excellent  ║
║  Monitoring & Alertes     : ████████░░  8/10  ✅ Très bien  ║
║  Documentation            : █████████░  9/10  ✅ Excellent  ║
║  Sécurité des accès       : ███████░░░  7/10  ⚠️  À renforcer║
║  Procédures d'urgence     : ██████░░░░  6/10  ⚠️  À tester  ║
║  Backups automatiques     : ███░░░░░░░  3/10  🔴 Prioritaire║
║                                                              ║
║  SCORE GLOBAL             : ████████░░ 7.5/10              ║
║  → Excellent pour une 1ère mise en production DR !          ║
╚══════════════════════════════════════════════════════════════╝
```
