# 01 — Stratégie d'Architecture DR (BNM)

## 🎯 Objectif : Sécurité des Données Bancaires
La solution repose sur une architecture à **trois zones** pour garantir la disponibilité des services critiques (Click et Pleniged). 

### 1 — Les Piliers du Projet (RPO/RTO)

| Indicateur | Définition | Zone 2 (Secondaire Focal) | Zone 3 (Tertiaire) |
| :--- | :--- | :--- | :--- |
| **RPO** | Perte de données maximale | **0 seconde (Aucune perte)** | ~5-60 secondes |
| **RTO** | Délai de reprise de service | < 1 heure | < 4 heures |

---

## 🏗️ Détails Techniques de Réplication

### Zone 2 — Réplication Semi-Synchrone (Garantie de Données)
- **Logique :** Pour chaque `COMMIT` sur la Production, le Maître attend que la Zone 2 confirme la réception de la transaction avant de répondre à l'utilisateur.
- **Bénéfice :** Si la Prod brûle instantanément, la Zone 2 possède exactement la même donnée bancaire.
- **Timeout :** Si la Zone 2 met trop de temps à répondre (ex: panne réseau), la Prod repasse en asynchrone pour ne pas bloquer les clients.

### Zone 3 — Réplication Asynchrone (Redondance Géographique)
- **Logique :** La Prod envoie les logs à la Zone 3 sans attendre de confirmation.
- **Bénéfice :** Aucune latence sur la performance de Production.
- **Utilisation :** Utile si la Zone 1 et la Zone 2 sont perdues simultanément.

---

## 🐳 Pourquoi Docker pour le DR ?
Nous avons choisi une architecture **Multi-Conteneurs** MySQL 8.0 pour les serveurs de secours :

1. **Isolation :** Click (Instance 1) et Pleniged (Instance 2) tournent sur des processus séparés. Une panne sur l'un n'affecte pas l'autre. ✨
2. **Ports Dédiés :**
    - **Port 3306 :** Sauvegarde Click.
    - **Port 3307 :** Sauvegarde Pleniged.
3. **Consommation :** Un seul serveur physique pour héberger deux répertoires de données indépendants.

---

## ✨ Mode de Transport : Les GTID
Au lieu d'utiliser les noms de fichiers binlog, nous utilisons les **GTID** (Global Transaction IDs). Chaque transaction a un numéro unique (`UUID:ID`).
- **Avantage :** Si on déplace une base, elle sait exactement où elle en est sans erreur humaine. 🏁🏦🚀
