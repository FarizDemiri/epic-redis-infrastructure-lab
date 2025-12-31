# Redis Monitoring Setup Runbook

Production monitoring for HIPAA-compliant Redis cluster.

---

## Architecture Overview

```
Redis Nodes (3x)
    ↓
Redis Exporter (port 9121) ← Scrapes metrics via TLS
    ↓
Prometheus (scrapes every 30s)
    ↓
Grafana (dashboards)
```

---

## Phase 1: Redis Exporter Installation

### Install on Each Redis Node

**SSH to each instance:**

```bash
# Node 1a
ssh -i redis-cluster-key.pem ubuntu@3.149.234.87

# Node 1b
ssh -i redis-cluster-key.pem ubuntu@18.191.137.36

# Node 1c
ssh -i redis-cluster-key.pem ubuntu@3.16.125.60
```

**Install Redis Exporter (on each node):**

```bash
# Download latest Redis Exporter
cd ~
wget https://github.com/oliver006/redis_exporter/releases/download/v1.55.0/redis_exporter-v1.55.0.linux-amd64.tar.gz

# Extract
tar xzf redis_exporter-v1.55.0.linux-amd64.tar.gz
cd redis_exporter-v1.55.0.linux-amd64

# Move to /usr/local/bin
sudo mv redis_exporter /usr/local/bin/
sudo chmod +x /usr/local/bin/redis_exporter

# Verify
redis_exporter --version
```

**Expected output:** `redis_exporter v1.55.0`

---

### Configure Redis Exporter Service

**Create systemd service (on each node):**

```bash
sudo nano /etc/systemd/system/redis_exporter.service
```

**Paste:**

```ini
[Unit]
Description=Redis Exporter
After=network.target redis-server.service

[Service]
Type=simple
User=ubuntu
Environment="REDIS_ADDR=127.0.0.1:6379"
Environment="REDIS_USER=admin_user"
Environment="REDIS_PASSWORD=AdminPassword123!"
ExecStart=/usr/local/bin/redis_exporter \
  --redis.addr=redis://127.0.0.1:6379 \
  --redis.password=AdminPassword123! \
  --redis.user=admin_user \
  --tls-client-cert-file=/home/ubuntu/redis-certs/redis-server-cert.pem \
  --tls-client-key-file=/home/ubuntu/redis-certs/redis-server-key.pem \
  --tls-ca-cert-file=/home/ubuntu/redis-certs/ca-cert.pem \
  --web.listen-address=:9121

Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Start the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable redis_exporter
sudo systemctl start redis_exporter
sudo systemctl status redis_exporter
```

**Expected:** `Active: active (running)`

**Test metrics endpoint:**

```bash
curl http://localhost:9121/metrics | head -20
```

**Expected:** See metrics like `redis_up 1`, `redis_commands_total`, etc.

---

## Phase 2: Update Security Group

**Add rule for Prometheus to scrape exporters:**

```
AWS Console → EC2 → Security Groups → redis-cluster-sg

Add Inbound Rule:
Type: Custom TCP
Port: 9121
Source: 10.0.0.0/16  (VPC CIDR - for now)
Description: Redis Exporter metrics
```

**Why 10.0.0.0/16:** Allows Prometheus instance (in same VPC) to scrape metrics.

---

## Phase 3: Deploy Prometheus Instance

### Launch Monitoring Instance

```
AWS Console → EC2 → Launch Instance

Name: prometheus-grafana
AMI: Ubuntu Server 24.04 LTS
Instance type: t2.micro
Key pair: redis-cluster-key (reuse existing)
Network:
  VPC: redis-vpc
  Subnet: redis-subnet-1a (same AZ as node 1)
  Auto-assign public IP: Enable
Security group: Create new → prometheus-sg

Inbound rules:
  SSH (22): My IP
  Prometheus (9090): My IP
  Grafana (3000): My IP

Storage: 8 GB gp3

Launch instance
```

**Note the public IP:** `<PROMETHEUS_PUBLIC_IP>`

---

### Install Prometheus

**SSH to prometheus instance:**

```bash
ssh -i redis-cluster-key.pem ubuntu@<PROMETHEUS_PUBLIC_IP>
```

**Install Prometheus:**

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Download Prometheus
cd ~
wget https://github.com/prometheus/prometheus/releases/download/v2.48.0/prometheus-2.48.0.linux-amd64.tar.gz

# Extract
tar xzf prometheus-2.48.0.linux-amd64.tar.gz
cd prometheus-2.48.0.linux-amd64

# Move binaries
sudo mv prometheus promtool /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo mv consoles console_libraries prometheus.yml /etc/prometheus/

# Create prometheus user
sudo useradd --no-create-home --shell /bin/false prometheus
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

**Configure Prometheus:**

```bash
sudo nano /etc/prometheus/prometheus.yml
```

**Replace contents:**

```yaml
global:
  scrape_interval: 30s
  evaluation_interval: 30s

scrape_configs:
  - job_name: 'redis-cluster'
    static_configs:
      - targets:
          - '10.0.1.102:9121'  # redis-node-1a private IP
          - '10.0.2.27:9121'   # redis-node-1b private IP
          - '10.0.3.205:9121'  # redis-node-1c private IP
        labels:
          cluster: 'redis-prod'
          environment: 'production'
```

**IMPORTANT:** Verify the private IPs match your actual Redis nodes!

**Create systemd service:**

```bash
sudo nano /etc/systemd/system/prometheus.service
```

**Paste:**

```ini
[Unit]
Description=Prometheus
After=network.target

[Service]
Type=simple
User=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

Restart=always

[Install]
WantedBy=multi-user.target
```

**Start Prometheus:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

**Expected:** `Active: active (running)`

**Verify targets:**

Open browser: `http://<PROMETHEUS_PUBLIC_IP>:9090/targets`

**Expected:** All 3 Redis exporters showing `UP` (green)

---

## Phase 4: Install Grafana

**On prometheus instance:**

```bash
# Add Grafana repository
sudo apt-get install -y software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

# Install Grafana
sudo apt-get update
sudo apt-get install grafana -y

# Start Grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

**Expected:** `Active: active (running)`

**Access Grafana:**

Open browser: `http://<PROMETHEUS_PUBLIC_IP>:3000`

**Default login:**

- Username: `admin`
- Password: `admin`
- (You'll be prompted to change it)

---

## Phase 5: Configure Grafana Data Source

**In Grafana UI:**

1. Click **⚙️ Configuration** (gear icon) → **Data Sources**
2. Click **Add data source**
3. Select **Prometheus**
4. Configure:
   - **URL:** `http://localhost:9090`
   - **Access:** Server (default)
5. Click **Save & Test**

**Expected:** ✅ "Data source is working"

---

## Phase 6: Import Redis Dashboard

**Method 1: Use Pre-built Dashboard**

1. Click **+** → **Import**
2. Enter dashboard ID: `11835` (Redis Dashboard for Prometheus)
3. Click **Load**
4. Select Prometheus data source
5. Click **Import**

**Expected:** Beautiful dashboard showing:

- Redis uptime
- Memory usage
- Commands per second
- Connected clients
- Keyspace hits/misses

**Method 2: Create Custom Dashboard**

See [Grafana Custom Dashboards](#grafana-custom-dashboards) section below.

---

## Verification Tests

### Test 1: Exporter Metrics

**On any Redis node:**

```bash
curl http://localhost:9121/metrics | grep redis_up
```

**Expected:** `redis_up 1`

### Test 2: Prometheus Scraping

**Query Prometheus:**

```bash
curl 'http://<PROMETHEUS_IP>:9090/api/v1/query?query=redis_up'
```

**Expected:** JSON response with `"value": [timestamp, "1"]` for each instance

### Test 3: Grafana Visualization

1. Open Grafana dashboard
2. Look for "Memory Usage" panel
3. **Expected:** Graph showing memory usage for all 3 nodes

### Test 4: Simulate Load

**Generate traffic:**

```bash
redis-cli -c --tls \
  --cert ~/redis-certs/redis-server-cert.pem \
  --key ~/redis-certs/redis-server-key.pem \
  --cacert ~/redis-certs/ca-cert.pem \
  --user admin_user --pass AdminPassword123!

# In redis-cli:
> SET test:1 "value1"
> SET test:2 "value2"
> GET test:1
> GET test:2
```

**Watch Grafana:** Commands per second should increase

---

## Key Metrics Explained

### redis_up

- **Meaning:** Redis instance availability (1=up, 0=down)
- **Alert threshold:** `== 0` (CRITICAL)

### redis_memory_used_bytes

- **Meaning:** Current memory consumption
- **Alert threshold:** `> 80%` of max memory

### redis_commands_processed_total

- **Meaning:** Total commands executed (counter)
- **Use:** Calculate rate: `rate(redis_commands_processed_total[1m])`
- **Typical:** 1000-10000 ops/sec for moderate load

### redis_keyspace_hits_total / redis_keyspace_misses_total

- **Meaning:** Cache hits vs misses
- **Calculate hit ratio:** `hits / (hits + misses)`
- **Good:** >70% hit rate

### redis_connected_clients

- **Meaning:** Number of active connections
- **Typical:** 10-100 for application usage
- **Alert threshold:** Sudden spike or drop

### redis_cluster_state

- **Meaning:** Cluster health (1=ok, 0=fail)
- **Alert threshold:** `== 0` (CRITICAL)

---

## Troubleshooting

### Exporter Shows "Connection Refused"

**Symptom:** `redis_up 0` or exporter logs show connection errors

**Causes:**

1. Redis not running
2. Wrong TLS certificates
3. Wrong ACL password

**Fix:**

```bash
# Check Redis is running
sudo redis-server /etc/redis/redis-cluster.conf &

# Test connection manually
redis-cli --tls \
  --cert ~/redis-certs/redis-server-cert.pem \
  --key ~/redis-certs/redis-server-key.pem \
  --cacert ~/redis-certs/ca-cert.pem \
  --user admin_user --pass AdminPassword123! \
  PING
```

### Prometheus Targets Are "Down"

**Symptom:** Targets page shows red "DOWN" status

**Causes:**

1. Security group blocking port 9121
2. Wrong private IP in prometheus.yml
3. Exporter not running

**Fix:**

```bash
# Verify security group allows 9121 from VPC
# Check private IPs match
aws ec2 describe-instances --filters "Name=tag:Name,Values=redis-node-*" \
  --query 'Reservations[*].Instances[*].[Tags[?Key==`Name`].Value|[0],PrivateIpAddress]'

# Verify exporter is running on each node
sudo systemctl status redis_exporter
```

### Grafana Shows "No Data"

**Symptom:** Dashboard panels empty

**Causes:**

1. Data source not configured
2. Prometheus not scraping
3. Wrong time range selected

**Fix:**

```bash
# Test Prometheus data source
curl http://localhost:9090/api/v1/query?query=redis_up

# Check time range (top-right in Grafana)
# Set to "Last 5 minutes"
```

---

## Cost Management

**Running costs:**

| Service | Instance | Cost/Hour | Cost/Month (730 hrs) |
|---------|----------|-----------|---------------------|
| Redis nodes (3) | t2.micro | $0.012 x 3 | $26 |
| Prometheus/Grafana | t2.micro | $0.012 | $9 |
| **Total** | | **$0.048** | **$35** |

**Storage:** ~$3/month (EBS volumes)

**To save money:**

```bash
# Stop Prometheus/Grafana when not actively monitoring
aws ec2 stop-instances --instance-ids <PROMETHEUS_INSTANCE_ID>

# Stop Redis nodes when not in use
aws ec2 stop-instances --instance-ids <NODE_1A_ID> <NODE_1B_ID> <NODE_1C_ID>
```

---

## Next Steps

1. **Set up alerting** (Prometheus Alert Manager)
2. **Add more dashboards** (Performance, Resources)
3. **Integrate with PagerDuty/Slack** (production alerting)
4. **Add CloudWatch integration** (AWS-native metrics)
5. **Configure backup monitoring** (EBS snapshot status)

---

**Production monitoring complete!** Your Redis cluster now has enterprise-grade observability. 📊
