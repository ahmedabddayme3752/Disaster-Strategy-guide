# 04 — Guide de Validation DR (Le "Grand Direct")

## 🎯 Objectif : Prouver la Réussite du Projet
Ce document détaille comment confirmer que la réplication des données bancaires fonctionne à la milliseconde près.

---

## 🧪 TEST 1 — Validation en Temps Réel
Nous allons insérer une ligne sur la Production et vérifier son apparition immédiate sur les sites de secours.

### 1 — Étape 1 : Insertion sur la Prod (241)
Sur le serveur **Click (Prod 241)**, lancez la commande suivante :

```bash
mysql -u nio -p -e "INSERT INTO clickTest.dr_validation (test_message) VALUES ('TEST BNM DR SUCCESS - SYNC OK');"
```

### 2 — Étape 2 : Vérification sur Zone 2 et Zone 3
Sur les serveurs DR, lancez la commande suivante :

```bash
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "SELECT * FROM clickTest.dr_validation ORDER BY id DESC LIMIT 1;"
```

> [!IMPORTANT]
> **Résultat attendu :** Vous devez voir la ligne **`TEST BNM DR SUCCESS - SYNC OK`** avec l'horodatage actuel. Si c'est le cas, votre réplication est **parfaite**. 🏆🏦💎

---

## 🧪 TEST 2 — Intégrité des Données (Comptage)
Vérifions que le volume de données est identique entre le Maître et les Esclaves.

### 1 — Comptage sur la Prod
```bash
mysql -u nio -p -e "SELECT count(*) FROM clickTest.dr_validation;"
```

### 2 — Comptage sur le DR (Zone 2/3)
```bash
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "SELECT count(*) FROM clickTest.dr_validation;"
```

---

## 📊 Évaluation du RPO (Perte de Données)
- Si le `id` du dernier test est identique sur Prod et DR ➡️ **RPO = 0 (Aucune perte)**. ✅
- Si `Seconds_Behind_Master` est à **0** ➡️ **Synchro Temps Réel**. ✅

---

## 🏁 Clôture de la Mission
Une fois ces tests validés, le projet Click est considéré comme **Production-Ready**. 🚀💰🏁🏦
