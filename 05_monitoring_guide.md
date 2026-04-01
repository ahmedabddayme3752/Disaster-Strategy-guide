# Guide 5 — Monitoring avec Prometheus & Grafana (Architecture Double-Instance)

Ce guide explique comment surveiller vos **4 instances MySQL** (Click et Pleniged sur Zone 2 et Zone 3).

---

## Étape 1 — Préparer les utilisateurs monitoring

Sur **Zone 2** ET **Zone 3**, exécutez ces commandes pour chaque instance :

```bash
# Pour Click (Port 3306)
sudo docker exec -it mysql-click mysql -u root -pRootPassword123! -e "
CREATE USER 'prometheus'@'192.168.1.65' IDENTIFIED BY 'MonitorPass2026!';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'prometheus'@'192.168.1.65';
"

# Pour Pleniged (Port 3307)
sudo docker exec -it mysql-pleniged mysql -u root -pRootPassword123! -e "
CREATE USER 'prometheus'@'192.168.1.65' IDENTIFIED BY 'MonitorPass2026!';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'prometheus'@'192.168.1.65';
"
```

---

## Étape 2 — Installer DEUX exporteurs par VM

Sur **Zone 2** ET **Zone 3**, nous allons lancer deux exporteurs pour différencier les métriques.

### A. Exporter pour Click (Port 9104)
```bash
# Credentials
cat > ~/.my_click.cnf << EOF
[client]
user=prometheus
password=MonitorPass2026!
host=127.0.0.1
port=3306
EOF

# Service Systemd
sudo tee /etc/systemd/system/mysqld_exporter_click.service > /dev/null << EOF
[Unit]
Description=MySQL Click Exporter
After=network.target

[Service]
User=$USER
ExecStart=/usr/local/bin/mysqld_exporter \
  --config.my-cnf=/home/$USER/.my_click.cnf \
  --web.listen-address=:9104
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

### B. Exporter pour Pleniged (Port 9105)
```bash
# Credentials
cat > ~/.my_pleniged.cnf << EOF
[client]
user=prometheus
password=MonitorPass2026!
host=127.0.0.1
port=3307
EOF

# Service Systemd
sudo tee /etc/systemd/system/mysqld_exporter_pleniged.service > /dev/null << EOF
[Unit]
Description=MySQL Pleniged Exporter
After=network.target

[Service]
User=$USER
ExecStart=/usr/local/bin/mysqld_exporter \
  --config.my-cnf=/home/$USER/.my_pleniged.cnf \
  --web.listen-address=:9105
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

### C. Activer les services
```bash
sudo systemctl daemon-reload
sudo systemctl enable mysqld_exporter_click mysqld_exporter_pleniged
sudo systemctl start mysqld_exporter_click mysqld_exporter_pleniged
```

---

## Étape 3 — Configurer Prometheus (Zone 1)

Modifiez votre `prometheus.yml` sur la Zone 1 pour surveiller les 4 cibles :

```yaml
scrape_configs:
  # --- CLICK ---
  - job_name: 'mysql_z2_click'
    static_configs:
      - targets: ['192.168.1.66:9104']
  
  - job_name: 'mysql_z3_click'
    static_configs:
      - targets: ['192.168.1.64:9104']

  # --- PLENIGED ---
  - job_name: 'mysql_z2_pleniged'
    static_configs:
      - targets: ['192.168.1.66:9105']

  - job_name: 'mysql_z3_pleniged'
    static_configs:
      - targets: ['192.168.1.64:9105']
```

---

## Étape 4 — Relancer le Monitoring (Zone 1)
```bash
cd ~/monitoring
sudo docker-compose restart prometheus
```
