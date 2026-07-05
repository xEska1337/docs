# Monitoring Stack Deployment

This guide covers the deployment of a centralized monitoring and logging stack using **Grafana, Prometheus, Loki, SNMP Exporter, and Grafana Alloy**. It also details how to onboard various homelab devices into the monitoring ecosystem.

---

## Part 1: Core Stack Installation

### 1. LXC Preparation

Create a new Debian LXC in Proxmox and update the system:

```bash
apt update && apt upgrade -y

```

### 2. Install Docker

Install the Docker engine and its dependencies.

??? info "Docker Official Docs"
    [Docker Engine Installation on Debian](https://docs.docker.com/engine/install/debian/)

```bash
# Add Docker's official GPG key:
apt update
apt install ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

apt update
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3. Folder Structure & Permissions

Create the required directory tree for persistent data and configurations:

```text
monitoring/
├── docker-compose.yml
├── grafana/
├── alloy/
│   └── config.alloy
├── prometheus/
│   ├── data/
│   └── prometheus.yml
└── loki/
    ├── data/
    └── loki-config.yaml

```

Set the proper ownership permissions so the containers can write to their respective directories:

```bash
chown -R 472:472 ./grafana
chown -R 65534:65534 ./prometheus/data
chown -R 10001:10001 ./loki/data

```

### 4. Configuration Files

Populate the configuration files within your new directory structure.

**`prometheus/prometheus.yml`**

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: 'snmp_exporter'
    static_configs:
      - targets: ['snmp-exporter:9116']

```

**`loki/loki-config.yaml`**

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2026-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

```

**`alloy/config.alloy`**

```hcl
loki.source.syslog "network_syslog" {
  listener {
    address  = "0.0.0.0:1514"
    protocol = "udp"
  }
  
  relabel_rules = loki.relabel.syslog_tags.rules
  forward_to = [loki.write.local_loki.receiver]
}

loki.relabel "syslog_tags" {
  forward_to = []

  rule {
    source_labels = ["__syslog_message_hostname"]
    target_label  = "host"
  }
  
  rule {
    source_labels = ["__syslog_message_severity"]
    target_label  = "level"
  }

  rule {
    source_labels = ["__syslog_connection_ip_address"]
    target_label  = "source_ip"
  }

  rule {
    target_label = "job"
    replacement  = "syslog"
  }
}

loki.write "local_loki" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}

```

### 5. Docker Compose Deployment

Create the `docker-compose.yml` file to tie the stack together:

```yaml title="docker-compose.yml"
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./grafana:/var/lib/grafana
    depends_on:
      - prometheus
      - loki

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-remote-write-receiver'

  loki:
    image: grafana/loki:latest
    container_name: loki
    restart: unless-stopped
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yaml:/etc/loki/local-config.yaml:ro
      - ./loki/data:/loki
    command: -config.file=/etc/loki/local-config.yaml

  snmp-exporter:
    image: prom/snmp-exporter:latest
    container_name: snmp-exporter
    restart: unless-stopped
    ports:
      - "9116:9116"

  syslog-receiver:
    image: grafana/alloy:latest
    container_name: syslog-receiver
    restart: unless-stopped
    ports:
      - "1514:1514/udp"
    volumes:
      - ./alloy/config.alloy:/etc/alloy/config.alloy:ro
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - /etc/alloy/config.alloy

```

### 6. Start the Stack & Initialize Grafana

Start the containers in detached mode:

```bash
docker compose up -d

```

1. Navigate to `http://<YOUR_MONITORING_STACK_IP>:3000` to access Grafana.
2. Log in with the default credentials (`admin` / `admin`).
3. Go to **Connections > Data Sources**.
4. Add **Prometheus** using the internal Docker URL: `http://prometheus:9090`.
5. Add **Loki** using the internal Docker URL: `http://loki:3100`.

---

## Part 2: Adding Devices

### 1. Proxmox VE

??? info "Resources"
    * [Grafana Alloy Installation](https://grafana.com/docs/alloy/latest/set-up/install/linux/)
    * [PVE-Exporter Github](https://github.com/bigtcze/pve-exporter/)

#### Alloy & Node Metrics

SSH into your Proxmox Host (or open the node's web shell in the Proxmox GUI) to install Alloy and hardware sensors:

```bash
mkdir -p /etc/apt/keyrings
wget -O /etc/apt/keyrings/grafana.asc https://apt.grafana.com/gpg-full.key
chmod 644 /etc/apt/keyrings/grafana.asc
echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" | tee /etc/apt/sources.list.d/grafana.list

apt-get update
apt-get install alloy lm-sensors -y
usermod -aG adm,systemd-journal alloy

```

Edit the local Alloy config (`nano /etc/alloy/config.alloy`) and insert:

```hcl
prometheus.exporter.unix "proxmox" { }

prometheus.scrape "scrape_proxmox" {
  targets         = prometheus.exporter.unix.proxmox.targets
  scrape_interval = "15s"
  forward_to      = [prometheus.relabel.node_exporter_labels.receiver]
}

prometheus.relabel "node_exporter_labels" {
  forward_to = [prometheus.remote_write.local_prometheus.receiver]

  rule {
    target_label = "job"
    replacement  = "node-exporter"
  }

  rule {
    target_label = "instance"
    replacement  = constants.hostname
  }
}

prometheus.remote_write "local_prometheus" {
  endpoint {
    url = "http://<YOUR_MONITORING_STACK_IP>:9090/api/v1/write"
  }
}

loki.source.journal "proxmox_journal" {
  forward_to = [loki.relabel.proxmox_tags.receiver]
  labels = {
    job  = "proxmox-logs",
    host = constants.hostname,
  }
}

loki.relabel "proxmox_tags" {
  forward_to = [loki.write.local_loki.receiver]

  rule {
    source_labels = ["__journal__systemd_unit"]
    target_label  = "unit"
  }

  rule {
    source_labels = ["__journal__transport"]
    regex         = "kernel"
    target_label  = "unit"
    replacement   = "kernel"
  }

  rule {
    source_labels = ["__journal_priority"]
    regex         = "0|1|2|3"
    target_label  = "level"
    replacement   = "error"
  }
  
  rule {
    source_labels = ["__journal_priority"]
    regex         = "4"
    target_label  = "level"
    replacement   = "warn"
  }
  
  rule {
    source_labels = ["__journal_priority"]
    regex         = "5|6"
    target_label  = "level"
    replacement   = "info"
  }
  
  rule {
    source_labels = ["__journal_priority"]
    regex         = "7"
    target_label  = "level"
    replacement   = "debug"
  }

  rule {
    source_labels = ["level"]
    regex         = "^$"
    target_label  = "level"
    replacement   = "info"
  }
}

loki.write "local_loki" {
  endpoint {
    url = "http://<YOUR_MONITORING_STACK_IP>:3100/loki/api/v1/push"
  }
}

```

Enable and start Alloy:

```bash
systemctl daemon-reload
systemctl enable --now alloy

```

#### Proxmox API Exporter & S.M.A.R.T

1. Log into your **Proxmox Web GUI**.
2. Go to **Datacenter > Permissions > Users**. Add a user: `pve-exporter`, Realm: `Proxmox VE authentication server (pve)`.
3. Go to **Datacenter > Permissions**. Add a **User Permission**: Path: `/`, User: `pve-exporter@pve`, Role: `PVEAuditor`.
4. Go to **Datacenter > Permissions > API Tokens**. Add a token for the user, Token ID: `monitoring`, and **uncheck** Privilege Separation. Copy the Secret value.

Back in the Proxmox SSH shell, set up the exporter daemon:

```bash
# Create User
useradd --system --no-create-home --shell /usr/sbin/nologin pve-exporter

# Install Binary
wget -O /usr/local/bin/pve-exporter https://github.com/bigtcze/pve-exporter/releases/latest/download/pve-exporter-linux-amd64
chmod +x /usr/local/bin/pve-exporter

# Create Configuration
mkdir -p /etc/pve-exporter
cat > /etc/pve-exporter/config.yml << 'EOF'
proxmox:
  host: "<YOUR_PROXMOX_IP>"
  port: 8006
  token_id: "pve-exporter@pve!monitoring"
  token_secret: "<PASTE_YOUR_SECRET_HERE>"
  insecure_skip_verify: true

server:
  listen_address: ":9221"
  metrics_path: "/metrics"
EOF

chown root:pve-exporter /etc/pve-exporter/config.yml
chmod 640 /etc/pve-exporter/config.yml

# Create Systemd Service
cat > /etc/systemd/system/pve-exporter.service << 'EOF'
[Unit]
Description=Proxmox VE Exporter for Prometheus
Documentation=https://github.com/bigtcze/pve-exporter
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=pve-exporter
Group=pve-exporter
ExecStart=/usr/local/bin/pve-exporter -config /etc/pve-exporter/config.yml
Restart=on-failure
RestartSec=5
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
ReadOnlyPaths=/
ReadWritePaths=

[Install]
WantedBy=multi-user.target
EOF

# Enable and Start
systemctl daemon-reload
systemctl enable --now pve-exporter

# Add S.M.A.R.T Monitoring Script
wget -O /usr/local/bin/pve-smart-collector.sh https://raw.githubusercontent.com/bigtcze/pve-exporter/main/scripts/pve-smart-collector.sh
chmod +x /usr/local/bin/pve-smart-collector.sh
mkdir -p /var/lib/pve-exporter

echo '*/5 * * * * root /usr/local/bin/pve-smart-collector.sh' | tee /etc/cron.d/pve-smart-collector
/usr/local/bin/pve-smart-collector.sh

```

**Prometheus Config Update:**
Add this to your `prometheus.yml` on the monitoring LXC and restart the Prometheus container:

```yaml
  - job_name: 'pve'
    static_configs:
      - targets: ['<YOUR_PROXMOX_IP>:9221']

```

* **Grafana Dashboard ID:** `24550`

---

### 2. TrueNAS SCALE

??? info "TrueNAS API Exporter"
    [Unknowlars/truenas-scale-api-prometheus-exporter](https://github.com/Unknowlars/truenas-scale-api-prometheus-exporter)

#### Syslog Forwarding

1. Go to **System Settings > Advanced**.
2. Scroll down to the **Syslog** widget and click **Configure**.
3. Select Syslog level: **Info**.
4. In the **Syslog Server** field, enter `<YOUR_MONITORING_STACK_IP>:1514`.
5. Set the transport protocol to **UDP**.
6. Check the **Audit** box.

#### Deep Metrics via API

The exporter requires a dedicated API key to securely read data from the NAS.

1. Navigate to **Credentials > Users** and add a user (e.g., `api-readonly`) with a strong password.
2. Navigate to **Credentials > Local Groups**, edit the newly created `api-readonly` group, and add the **Read-Only Administrator** privilege.
3. Click the **Profile/User Icon** in the top right corner and select **My API Keys**.
4. Click **Add API Key**, name it, and ensure the username is set to `api-readonly`.

Add the exporter to your monitoring LXC's `docker-compose.yml`:

```yaml
  truenas-exporter:
    image: ghcr.io/unknowlars/truenas-scale-api-prometheus-exporter:latest
    container_name: truenas-exporter
    restart: unless-stopped
    ports:
      - "9108:9108"
    environment:
      - TRUENAS_WS_URL=wss://<YOUR_TRUENAS_IP>/api/current
      - TRUENAS_API_KEY=<YOUR_API_KEY>
      - TRUENAS_VERIFY_TLS=false

```

**Prometheus Config Update:**
Add this to `prometheus.yml` and restart Prometheus:

```yaml
  - job_name: 'truenas_exporter'
    static_configs:
      - targets: ['truenas-exporter:9108']

```

* **Grafana Dashboard ID:** `25081`

---

### 3. OPNsense Firewall

??? info "OPNsense API Exporter"
    [AthennaMind/opnsense-exporter](https://github.com/AthennaMind/opnsense-exporter)

#### Syslog Forwarding

1. Log into your **OPNsense Web GUI**.
2. Navigate to **System > Settings > Logging > Remote**.
3. Click **+ Add**.
4. Configure as follows:
* **Transport:** `UDP (IPv4)`
* **Hostname:** `<YOUR_MONITORING_STACK_IP>`
* **Port:** `1514`
* **RFC5424:** Checked (✅)
* **Description:** `Grafana Loki`



#### Basic Node Metrics

1. Go to **System > Firmware > Plugins**.
2. Search for and install `os-node_exporter`.
3. Navigate to **Services > Prometheus Exporter**.
4. Check all available boxes and apply.

#### Deep API Metrics

1. Go to **System > Access > Users**.
2. Create a dedicated user (e.g., `prometheus-scraper`). Assign required API privileges as listed in the exporter documentation.
3. Create an API key for this user. Download the `.txt` file containing the **API Key** and **API Secret**.

Add the exporter to your monitoring LXC's `docker-compose.yml`:

```yaml
  opnsense-exporter:
    image: ghcr.io/athennamind/opnsense-exporter:latest
    container_name: opnsense-exporter
    restart: unless-stopped
    ports:
      - "8080:8080"
    command:
      - "--opnsense.protocol=https"
      - "--opnsense.address=<YOUR_OPNSENSE_IP>"
      - "--exporter.instance-label=opnsense-main"
      - "--opnsense.insecure" 
      - "--exporter.disable-unbound"
    environment:
      OPNSENSE_EXPORTER_OPS_API_KEY: "<YOUR_API_KEY>"
      OPNSENSE_EXPORTER_OPS_API_SECRET: "<YOUR_API_SECRET>"

```

**Prometheus Config Update:**
Add this to `prometheus.yml` and restart Prometheus:

```yaml
  - job_name: 'opnsense-main'
    static_configs:
      - targets: ['<YOUR_OPNSENSE_IP>:9100']
  - job_name: 'opnsense-main-api'
    static_configs:
      - targets: ['opnsense-exporter:8080']

```

* **Grafana Dashboard IDs:** `21113`, `22248`

---

### 4. Windows PC

1. Open **PowerShell as Administrator**.
2. Install Grafana Alloy via winget:

```powershell
winget install GrafanaLabs.Alloy

```

3. Open the configuration file:

```powershell
notepad "C:\Program Files\GrafanaLabs\Alloy\config.alloy"

```

4. Replace the contents with:

```hcl
prometheus.exporter.windows "local_system" {
  enabled_collectors = ["cpu", "logical_disk", "net", "os", "system", "memory"]
}

prometheus.scrape "scrape_windows" {
  targets         = prometheus.exporter.windows.local_system.targets
  scrape_interval = "15s"
  forward_to      = [prometheus.remote_write.local_prometheus.receiver]
}

loki.source.windowsevent "application" {
  eventlog_name = "Application"
  forward_to    = [loki.write.local_loki.receiver]
  labels        = { job = "windows-events", host = "Windows-PC" }
}

loki.source.windowsevent "system" {
  eventlog_name = "System"
  forward_to    = [loki.write.local_loki.receiver]
  labels        = { job = "windows-events", host = "Windows-PC" }
}

loki.source.windowsevent "security" {
  eventlog_name = "Security"
  forward_to    = [loki.write.local_loki.receiver]
  labels        = { job = "windows-events", host = "Windows-PC" }
}

prometheus.remote_write "local_prometheus" {
  endpoint {
    url = "http://<YOUR_MONITORING_STACK_IP>:9090/api/v1/write"
  }
}

loki.write "local_loki" {
  endpoint {
    url = "http://<YOUR_MONITORING_STACK_IP>:3100/loki/api/v1/push"
  }
}

```

5. Restart the service to apply changes:

```powershell
Restart-Service Alloy
Get-Service Alloy

```

*(Verify it shows a "Running" status).*

* **Grafana Dashboard ID:** `16523`

---

### 5. SNMP Devices (e.g., TP-Link Switch)

#### SNMP v1/v2c Setup

In your managed switch's web interface (under SNMP config):

1. Click **+ Add**.
2. **Community Name:** Type `public`.
3. **Access Mode:** Select **Read-Only**.
4. **MIB View:** Leave as default.
5. Apply/Save.

#### Remote Logs Setup (Syslog)

In your switch's Log Server config:

1. Check the box for an active Index.
2. **Server IP:** `<YOUR_MONITORING_STACK_IP>`
3. **UDP Port:** `1514`
4. **Severity:** Set to `level_6` or `level_7` to capture standard connection logs.
5. **Status:** Enable.

**Prometheus Config Update:**
Add this to `prometheus.yml` and restart Prometheus:

```yaml
  - job_name: 'snmp'
    static_configs:
      - targets:
        - <YOUR_DEVICE_IP>
    metrics_path: /snmp
    params:
      auth: [public_v2]
      module: [if_mib]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: snmp-exporter:9116

```

* **Grafana Dashboard ID:** `11169`