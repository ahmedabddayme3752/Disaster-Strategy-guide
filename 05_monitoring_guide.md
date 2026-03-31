# Guide 5 — Monitoring avec Prometheus & Grafana (Zone 1)

> 👤 **Responsable : Ahmed**
> ⏱️ **À faire après :** `04_validation_replication_guide.md` — réplication validée
> 🎯 **Objectif :** Configurer Zone 1 (`192.168.1.65`) pour surveiller en temps réel la santé de la réplication MySQL sur Zone 2 et Zone 3

---

## Architecture du monitoring

```
Zone 2 (192.168.1.66) ──port 9104──▶ ┐
                                       ├──▶ Zone 1 Prometheus (192.168.1.65:9090)
Zone 3 (192.168.1.64) ──port 9104──▶ ┘          │
                                                   ▼
                                           Grafana (192.168.1.65:3000)
                                                   │
                                                   ▼
                                           Alertmanager (Emails / SMS)
```

---

## Étape 1 — Préparer les utilisateurs monitoring sur les VMs DR

À exécuter **une fois** sur Zone 2 ET Zone 3 :

### Sur Zone 2 (`192.168.1.66`)
```bash
ssh primbackupdr@192.168.1.66
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "
CREATE USER 'prometheus'@'192.168.1.65' IDENTIFIED BY 'MonitorPass2026!';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'prometheus'@'192.168.1.65';
FLUSH PRIVILEGES;
"
```

### Sur Zone 3 (`192.168.1.64`)
```bash
ssh secbackupdr@192.168.1.64
sudo docker exec -it mysql-server mysql -u root -pRootPassword123! -e "
CREATE USER 'prometheus'@'192.168.1.65' IDENTIFIED BY 'MonitorPass2026!';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'prometheus'@'192.168.1.65';
FLUSH PRIVILEGES;
"
```

---

## Étape 2 — Installer mysqld_exporter sur Zone 2 et Zone 3

À exécuter sur **chaque VM DR** (Zone 2 et Zone 3) :

```bash
# Télécharger mysqld_exporter
cd ~
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz
tar xf mysqld_exporter-0.15.1.linux-amd64.tar.gz
sudo mv mysqld_exporter-0.15.1.linux-amd64/mysqld_exporter /usr/local/bin/

# Créer le fichier de credentials
cat > ~/.my.cnf << EOF
[client]
user=prometheus
password=MonitorPass2026!
host=127.0.0.1
port=3306
EOF
chmod 600 ~/.my.cnf

# Créer le service systemd
sudo tee /etc/systemd/system/mysqld_exporter.service > /dev/null << EOF
[Unit]
Description=MySQL Exporter for Prometheus
After=network.target

[Service]
User=$USER
ExecStart=/usr/local/bin/mysqld_exporter --config.my-cnf=/home/$USER/.my.cnf
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Démarrer l'exporteur
sudo systemctl daemon-reload
sudo systemctl enable mysqld_exporter
sudo systemctl start mysqld_exporter

# Vérifier
sudo systemctl status mysqld_exporter
curl http://localhost:9104/metrics | head -20
```

---

## Étape 3 — Installer Prometheus + Grafana sur Zone 1

Connectez-vous à Zone 1 :
```bash
ssh monotordr@192.168.1.65
mkdir -p ~/monitoring
cd ~/monitoring
```

Créez le fichier `docker-compose.yml` :
```bash
nano ~/monitoring/docker-compose.yml
```

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alerts.yml:/etc/prometheus/alerts.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: "GrafanaPass2026!"
    volumes:
      - grafana_data:/var/lib/grafana

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: always
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml

volumes:
  prometheus_data:
  grafana_data:
```

---

## Étape 4 — Configurer Prometheus

Créez le fichier `~/monitoring/prometheus.yml` :
```bash
nano ~/monitoring/prometheus.yml
```

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "alerts.yml"

scrape_configs:
  - job_name: 'mysql_zone2_click_dr'
    static_configs:
      - targets: ['192.168.1.66:9104']
        labels:
          zone: 'zone2'
          role: 'primary_dr'
          system: 'click'

  - job_name: 'mysql_zone3_pleniged_dr'
    static_configs:
      - targets: ['192.168.1.64:9104']
        labels:
          zone: 'zone3'
          role: 'standby_dr'
          system: 'pleniged'
```

---

## Étape 5 — Configurer les alertes critiques

Créez le fichier `~/monitoring/alerts.yml` :
```bash
nano ~/monitoring/alerts.yml
```

```yaml
groups:
  - name: mysql_dr_alerts
    rules:

      - alert: MySQLReplicationStopped
        expr: mysql_slave_status_slave_io_running == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "🚨 Réplication MySQL arrêtée sur {{ $labels.instance }}"
          description: "Slave_IO_Running = 0 depuis plus d'1 minute. Action immédiate requise."

      - alert: MySQLReplicationLag
        expr: mysql_slave_status_seconds_behind_master > 30
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "⚠️ Délai de réplication élevé sur {{ $labels.instance }}"
          description: "Seconds_Behind_Master = {{ $value }}s. RPO compromis."

      - alert: MySQLDown
        expr: up{job=~"mysql_zone.*"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "🚨 MySQL DR inaccessible sur {{ $labels.instance }}"
          description: "Le serveur MySQL DR ne répond plus à Prometheus."

      - alert: MySQLDiskSpaceHigh
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "⚠️ Espace disque faible sur {{ $labels.instance }}"
          description: "Moins de 20% d'espace disque disponible."
```

---

## Étape 6 — Configurer Alertmanager (notifications)

Créez le fichier `~/monitoring/alertmanager.yml` :
```bash
nano ~/monitoring/alertmanager.yml
```

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: 'bnm-team'

receivers:
  - name: 'bnm-team'
    email_configs:
      - to: 'betah.lab@bnm.mr'
        from: 'alertes-dr@bnm.mr'
        smarthost: 'smtp.bnm.mr:587'
        auth_username: 'alertes-dr@bnm.mr'
        auth_password: '<MOT_DE_PASSE_SMTP>'
        send_resolved: true
```

> ⚠️ Remplacez les adresses email et les paramètres SMTP par les vraies valeurs BNM.

---

## Étape 7 — Démarrer le stack de monitoring

```bash
cd ~/monitoring
sudo docker-compose up -d

# Vérifier que les 3 conteneurs sont Up
sudo docker ps

# Vérifier les logs Prometheus
sudo docker logs prometheus --tail 20
```

---

## Étape 8 — Accéder aux interfaces

| Service | URL | Credentials |
|---------|-----|-------------|
| **Prometheus** | `http://192.168.1.65:9090` | aucun |
| **Grafana** | `http://192.168.1.65:3000` | admin / `GrafanaPass2026!` |
| **Alertmanager** | `http://192.168.1.65:9093` | aucun |

### Configurer Grafana

1. Ouvrez `http://192.168.1.65:3000`
2. Connectez-vous avec `admin` / `GrafanaPass2026!`
3. Allez dans **Configuration → Data Sources → Add data source**
4. Choisissez **Prometheus**, URL : `http://prometheus:9090`
5. Importez le dashboard MySQL : ID **`7362`** (MySQL Overview) depuis grafana.com

---

## Vérification finale

```bash
# Vérifier que Prometheus scrape bien les cibles
curl http://192.168.1.65:9090/api/v1/targets | python3 -m json.tool | grep -E "health|job"

# Vérifier les métriques de réplication
curl -s http://192.168.1.66:9104/metrics | grep mysql_slave_status_slave_io_running
curl -s http://192.168.1.64:9104/metrics | grep mysql_slave_status_slave_io_running
```

> ✅ Résultat attendu : `mysql_slave_status_slave_io_running 1` sur les deux VMs.

**Suite :** Passez au guide `06_failover_test_guide.md` pour tester le basculement d'urgence.
