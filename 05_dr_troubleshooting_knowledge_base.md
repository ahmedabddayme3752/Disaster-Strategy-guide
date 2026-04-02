# 05 — Base de Connaissances DR (Troubleshooting)

## 🧠 Mémoire Technique du Projet
Lors de la mise en place du DR pour la BNM, nous avons résolu plusieurs obstacles critiques. Ce document sert de mémoire pour les futurs administrateurs.

---

## 🐋 DOCKER & COMPOSE

### 1 — Erreur `KeyError: 'ContainerConfig'`
- **Symptôme :** Impossible de recréer un conteneur après une modification du `docker-compose.yml`.
- **Cause :** Bug de Docker Compose v1.x qui garde une trace corrompue de l'ID du conteneur.
- **Solution (Nuclear Fix) :**
    ```bash
    sudo docker-compose rm -f -s mysql-click
    sudo docker rm -f <ID_SPECIFIQUE_DE_L_ERREUR>
    sudo docker-compose up -d mysql-click
    ```

---

## 🔑 AUTHENTIFICATION & SÉCURITÉ

### 1 — Erreur `caching_sha2_password` / `Secure connection`
- **Symptôme :** L'esclave reste en `Connecting` avec une erreur d'authentification.
- **Cause :** MySQL 8.0 exige un canal chiffré pour le nouveau plugin par défaut.
- **Solution :** Utiliser `mysql_native_password` pour l'utilisateur `replicator` ET ajouter `GET_MASTER_PUBLIC_KEY=1` dans la commande `CHANGE MASTER`.

### 2 — Erreur `Access denied for root@localhost` (Native)
- **Symptôme :** Impossible de modifier la Prod avec l'utilisateur `root`.
- **Cause :** Ubuntu protège le compte root via `auth_socket`.
- **Solution :** Utiliser le compte de maintenance système `debian-sys-maint` trouvé dans `/etc/mysql/debian.cnf`.

---

## 🔌 RÉSEAUX & PORTS

### 1 — Conflit de Port 3306 (Docker de Test)
- **Symptôme :** L'esclave se connecte mais ne voit aucune donnée (UUID différent).
- **Cause :** Un conteneur Docker "fantôme" sur la Prod occupait le port 3306 à la place du vrai MySQL.
- **Solution :** Arrêter le Docker de test (`docker stop`) et forcer le vrai MySQL à écouter sur le port libre (ou utiliser un port alternatif comme **3366**).

---

## 🔄 RÉPLICATION GTID

### 1 — Erreur `Replica failed to initialize metadata structure` (Error 1872)
- **Symptôme :** `START SLAVE` échoue avec une erreur de structure.
- **Cause :** Les fichiers `relay-log.info` sont corrompus après plusieurs redémarrages forcés.
- **Solution :** Faire un `RESET SLAVE ALL;` complet pour vider les vieux fichiers et reconfigurer la connexion de zéro.

### 2 — Erreur `Unknown database 'click'` (Error 1049)
- **Symptôme :** L'esclave s'arrête car il ne trouve pas une base de test présente sur la Prod.
- **Solution :** Ajouter `--replicate-ignore-db=click` dans le `command:` de Docker Compose pour filtrer les données inutiles. 🚀🏦💰🏁
