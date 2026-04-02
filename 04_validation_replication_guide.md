# 04 — Guide de Validation de la Réplication

## 🎯 Objectif : Prouver que le DR Fonctionne

Ce guide décrit comment valider que les données de la production sont bien répliquées sur les sites de secours.

---

## 🧪 TEST 1 — Le "Grand Direct" (Validation Temps Réel)

Ce test prouve que toute modification sur la production apparaît **instantanément** sur les sites DR.

### Sur le Serveur Click Production (241)

```bash
# Connexion au MySQL de production
mysql -u nio -p -e "
-- Insérer un message de test horodaté
INSERT INTO clickTest.dr_validation (test_message)
VALUES ('BNM SUCCESS - ZONE 2 SYNC OK');
"
```

### Immédiatement sur Zone 2 (primbackupdr)

```bash
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "
-- Vérifier que le message est arrivé
SELECT * FROM clickTest.dr_validation ORDER BY id DESC LIMIT 1;
"
```

**Résultat attendu :**
```
+----+------------------------------+---------------------+
| id | test_message                 | horodatage          |
+----+------------------------------+---------------------+
|  3 | BNM SUCCESS - ZONE 2 SYNC OK | 2026-04-02 12:19:29 |
+----+------------------------------+---------------------+
```

> [!IMPORTANT]
> Si ce message apparaît sur Zone 2 en moins d'1 seconde après l'insertion sur la Prod, votre **RPO = 0** est validé. C'est la preuve que votre DR est opérationnel. 🏆

---

## 🧪 TEST 2 — Intégrité des Données (Comptage)

Ce test vérifie que **toutes** les données sont présentes sur les sites DR.

### Sur Production :
```bash
mysql -u nio -p -e "SELECT COUNT(*) as total_prod FROM clickTest.dr_validation;"
```

### Sur Zone 2 et Zone 3 :
```bash
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e \
  "SELECT COUNT(*) as total_dr FROM clickTest.dr_validation;"
```

> [!NOTE]
> Les deux totaux **doivent être identiques**. Si le total DR est inférieur, il y a un retard de réplication.

---

## 🧪 TEST 3 — Vérification de la Santé de la Réplication

### Commande de surveillance complète :
```bash
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "SHOW SLAVE STATUS\G" \
  | grep -E "Slave_IO_Running|Slave_SQL_Running|Seconds_Behind_Master|Executed_Gtid_Set|Last_IO_Error|Last_SQL_Error"
```

### Interprétation :

| Champ | Valeur Attendue | Signification |
|:---|:---|:---|
| `Slave_IO_Running` | `Yes` | Connexion réseau avec la Prod établie |
| `Slave_SQL_Running` | `Yes` | Les transactions s'appliquent sur l'esclave |
| `Seconds_Behind_Master` | `0` | Synchronisé en temps réel |
| `Last_IO_Error` | *(vide)* | Aucune erreur de connexion |
| `Last_SQL_Error` | *(vide)* | Aucune erreur d'application |

---

## 🧪 TEST 4 — Vérification des Tables et Données

Ce test montre toutes les tables présentes sur les sites DR.

```bash
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "
-- Liste de toutes les bases de données
SHOW DATABASES;

-- Tables dans clickTest
USE clickTest;
SHOW TABLES;

-- Aperçu des données dans la table de validation
SELECT * FROM dr_validation ORDER BY id DESC LIMIT 5;

-- Statistiques globales
SELECT
  table_name AS 'Table',
  table_rows AS 'Lignes (approx.)',
  ROUND(data_length/1024/1024, 2) AS 'Taille (MB)'
FROM information_schema.tables
WHERE table_schema = 'clickTest';
"
```

---

## 📊 Grille d'Évaluation du DR

| Critère | Poids | Résultat |
|:---|:---|:---|
| `Slave_IO_Running: Yes` | 🔴 Critique | ✅ Validé |
| `Slave_SQL_Running: Yes` | 🔴 Critique | ✅ Validé |
| `Seconds_Behind_Master: 0` | 🟠 Important | ✅ Validé |
| Test "Grand Direct" réussi | 🔴 Critique | ✅ Validé |
| Comptage identique Prod/DR | 🟠 Important | ✅ Validé |
| Dashboard Grafana actif | 🟡 Recommandé | ✅ Validé |
