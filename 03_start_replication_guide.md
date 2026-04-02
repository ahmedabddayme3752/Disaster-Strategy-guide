# 03 — Mise en Service de la Réplication DR (Zone 2/3)

## 🐳 Rôle : Les Esclaves de Secours
Ce document détaille comment démarrer la réplication sur les conteneurs Docker MySQL 8.0 des sites de secours.

---

## 🛠️ CONFIGURATION DOCKER (Multi-Instance)
Le fichier `docker-compose.yml` doit isoler Click (Port 3306) et Pleniged (Port 3307).

### Exemple de `docker-compose.yml` pour Click :
```yaml
  mysql-click:
    image: mysql:8.0
    command: --slave-skip-errors=1062,1396 --replicate-ignore-db=click
    container_name: mysql-click
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: "RootPassword123!"
```

---

## 🪄 PROCÉDURE : "Le Magic Sync"
Cette procédure garantit un démarrage propre sans conflits de GTID.

### Étape 1 : Préparer Bash
```bash
# Désactiver l'expansion de l'historique pour autoriser les "!" dans les mots de passe
set +H
```

### Étape 2 : Lancer le bloc de synchronisation
Modifiez les variables (UUID, Port) selon l'instance (Click ou Pleniged) :

```bash
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "
-- 1. Nettoyage des métadonnées
STOP SLAVE;
RESET SLAVE ALL;
RESET MASTER;

-- 2. Positionnement (GTID de départ du Dump)
-- ⚠️ Remplacez lourdement l'UUID et la plage par ceux de votre sauvegarde
SET GLOBAL GTID_PURGED = '816c0b5e-97e8-11ed-a73b-000c299194d8:1-215';

-- 3. Connexion Sécurisée
CHANGE MASTER TO
  MASTER_HOST='192.168.1.241',    -- IP Production
  MASTER_PORT=3366,               -- Port 3366 pour Click, 3306 pour Pleniged
  MASTER_USER='replicator',
  MASTER_PASSWORD='ReplicaPass2026!',
  MASTER_AUTO_POSITION=1,         -- Magie du GTID
  GET_MASTER_PUBLIC_KEY=1;        -- Autorise l'échange de clé RSA

-- 4. Départ ! 🚀
START SLAVE;
"
```

---

## 🧐 Pourquoi utiliser `GET_MASTER_PUBLIC_KEY=1` ?
MySQL 8.0 utilise le plugin `caching_sha2_password` par défaut. Sans cette option, le maître refuse de "parler" à l'esclave car le mot de passe n'est pas envoyé par un canal chiffré. Cette option règle le problème instantanément. ✨

---

## 🔍 Vérification du rattrapage
Pour voir si l'esclave est en train de "consommer" les transactions manquantes :

```bash
sudo docker exec -it mysql-click mysql -u root -p'RootPassword123!' -e "SHOW SLAVE STATUS\G" | grep -E "Slave_IO_Running|Slave_SQL_Running|Seconds_Behind_Master|Executed_Gtid_Set"
```

> [!NOTE]
> - **Slave_IO_Running: Yes** ✅
> - **Slave_SQL_Running: Yes** ✅
> - **Seconds_Behind_Master** : Doit diminuer jusqu'à atteindre **0**.
