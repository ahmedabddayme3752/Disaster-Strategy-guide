# 05 — Base de Connaissances (Troubleshooting)

## 🧠 Mémoire Technique du Projet BNM DR

> [!TIP]
> Cherchez l'erreur exacte dans ce document avant de faire des recherches. Toutes les erreurs rencontrées durant le projet BNM sont documentées ici avec leur solution testée et validée.

Ce document recense toutes les erreurs rencontrées durant l'implémentation du DR et leurs solutions. Il constitue la mémoire collective du projet pour les futurs administrateurs.

---

## 🐋 DOCKER & COMPOSE

### ❌ Erreur : `KeyError: 'ContainerConfig'`

**Symptôme :**
```
KeyError: 'ContainerConfig'
ERROR: for mysql-click  'ContainerConfig'
```

**Cause :**
Bug dans Docker Compose version 1.29.x. Quand le fichier `docker-compose.yml` est modifié, Compose essaie de récupérer les informations de l'ancienne image du conteneur, mais cette image (plus récente) ne contient plus le champ `ContainerConfig` dans ses métadonnées.

**Solution :**
```bash
# 1. Identifier l'ID du conteneur bloqué dans le message d'erreur
# Exemple : "b6379469cce5_mysql-click"

# 2. Supprimer le conteneur via docker-compose
sudo docker-compose rm -f -s mysql-click

# 3. Supprimer le fantôme par son ID
sudo docker rm -f b6379469cce5_mysql-click

# 4. Relancer la création
sudo docker-compose up -d --remove-orphans
```

---

## 🔑 AUTHENTIFICATION & SÉCURITÉ

### ❌ Erreur : `caching_sha2_password` / Secure Connection Required

**Symptôme :**
```
Last_IO_Error: Authentication plugin 'caching_sha2_password' reported error:
Authentication requires secure connection.
```

**Cause :**
MySQL 8.0 utilise par défaut le plugin `caching_sha2_password` qui exige un canal SSL/TLS chiffré. Sans SSL configuré entre l'esclave et le maître, l'authentification échoue.

**Solution :**
Deux actions combinées :
1. Créer l'utilisateur `replicator` avec le plugin `mysql_native_password` (ne nécessite pas de SSL)
2. Ajouter `GET_MASTER_PUBLIC_KEY=1` dans la commande `CHANGE MASTER TO`

```sql
-- Sur le Maître
CREATE USER 'replicator'@'%'
  IDENTIFIED WITH mysql_native_password BY 'ReplicaPass2026!';

-- Sur l'Esclave
CHANGE MASTER TO
  ...
  GET_MASTER_PUBLIC_KEY=1;
```

---

### ❌ Erreur : `Access denied for root@localhost`

**Symptôme :**
```
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
```

**Cause :**
Sur Ubuntu, MySQL utilise le plugin `auth_socket` pour l'utilisateur `root`. Cela signifie que `root` ne peut se connecter qu'avec `sudo mysql`, pas avec un mot de passe normal.

**Solution :**
Utiliser le compte de maintenance système `debian-sys-maint` dont le mot de passe est dans `/etc/mysql/debian.cnf`.

```bash
sudo cat /etc/mysql/debian.cnf
# Copier le mot de passe après "password = "
mysql -u debian-sys-maint -p'MOT_DE_PASSE' -e "VOS COMMANDES SQL;"
```

---

### ❌ Erreur : Bash — `!: event not found`

**Symptôme :**
```
-bash: !: event not found
```

**Cause :**
Bash interprète le caractère `!` dans les mots de passe comme une commande d'expansion d'historique.

**Solution :**
```bash
# Désactiver l'expansion d'historique pour la session
set +H

# Puis utiliser des guillemets simples pour le mot de passe
mysql -u root -p'RootPassword123!'
```

---

## 🔌 RÉSEAUX & PORTS

### ❌ Erreur : Conflit de Port 3306

**Symptôme :**
L'esclave se connecte à la production mais l'UUID renvoyé n'est pas celui du serveur de production réel.

**Cause :**
Un conteneur Docker de **test** tournait sur le serveur de production et occupait le port 3306 à la place du vrai MySQL natif. Le MySQL natif de production utilisait en réalité le port `3366`.

**Solution :**
1. Arrêter le conteneur Docker de test sur la production
2. Reconfigurer les esclaves avec le vrai port de production (`MASTER_PORT=3366`)

```bash
# Sur la Production : Vérifier quel processus écoute sur quel port
sudo ss -tulpn | grep mysqld
# Résultat attendu :
# tcp   LISTEN  0.0.0.0:3366  (mysqld, le vrai)
```

---

## 🔄 RÉPLICATION GTID

### ❌ Erreur : `Replica failed to initialize metadata` (Error 1872)

**Symptôme :**
```
ERROR 1872 (HY000): Replica failed to initialize applier metadata structure from the repository
```

**Cause :**
Les fichiers de métadonnées de réplication (`relay-log.info`, `master.info`) sont corrompus suite à plusieurs arrêts/redémarrages forcés du conteneur.

**Solution :**
Faire un nettoyage complet et recommencer la configuration :
```sql
STOP SLAVE;
RESET SLAVE ALL;   -- Supprime les fichiers corrompus
RESET MASTER;      -- Vide l'historique local
-- Reconfigurer avec CHANGE MASTER TO...
```

---

### ❌ Erreur : `Unknown database 'click'` (Error 1049)

**Symptôme :**
```
Error executing row event: 'Unknown database 'click''
```

**Cause :**
La base de données `click` (une base de test sur le serveur de production) n'existe pas sur les esclaves DR. L'esclave essaie de répliquer dessus et échoue.

**Solution :**
Deux options :
1. **Option rapide** : Créer la base vide → `CREATE DATABASE IF NOT EXISTS click;`
2. **Option propre** : Ignorer définitivement cette base dans `docker-compose.yml` → `--replicate-ignore-db=click`

---

### ❌ Erreur : `mysqld-exporter` — `permission denied` sur `.my.cnf`

**Symptôme :**
```
failed to load config from /cfg/.my.cnf: open /cfg/.my.cnf: permission denied
```

**Cause :**
Le fichier de credentials est en permissions `600` (lisible uniquement par le propriétaire). Le processus dans le conteneur Docker tourne sous un utilisateur différent.

**Solution :**
```bash
chmod 644 ~/mysql-dr/monitoring/*.my.cnf
sudo docker restart mysql-exporter-click mysql-exporter-pleniged
```

---

## 📞 Contacts d'Urgence

| Rôle | Responsabilité |
|:---|:---|
| **DBA Senior** | Validation des opérations GTID et Réplication |
| **Admin Systèmes** | Gestion des serveurs Linux et Docker |
| **DSI** | Décision de Failover |
