# External Services Setup Guide - PostgreSQL, Redis, Monitoring

## 🎯 Purpose

This guide covers setting up **external stateful services** that run **outside the K3s cluster** on dedicated VMs:

- **PostgreSQL cluster** (High Availability with Patroni)
- **Redis cluster** (High Availability with Redis Sentinel)
- **Monitoring stack** (Prometheus, Grafana, AlertManager)

## 🏗️ Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│                        K3s Cluster                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│  │Nextcloud │ │  Odoo    │ │ Keycloak │ │ Mautic   │        │
│  │   Pods   │ │   Pods   │ │   Pods   │ │   Pods   │        │
│  └─────┬────┘ └─────┬────┘ └─────┬────┘ └─────┬────┘        │
│        │            │            │            │               │
└────────┼────────────┼────────────┼────────────┼───────────────┘
         │            │            │            │
         │            │            │            │
         ▼            ▼            ▼            ▼
┌────────────────────────────────────────────────────────────────┐
│              External PostgreSQL Cluster                       │
│  ┌─────────────────────┐      ┌─────────────────────┐        │
│  │  VM1: Primary       │      │  VM2: Standby       │        │
│  │  • PostgreSQL 15+   │◄────►│  • PostgreSQL 15+   │        │
│  │  • Patroni          │      │  • Patroni          │        │
│  │  • PgBouncer        │      │  • PgBouncer        │        │
│  │  • postgres_exporter│      │  • postgres_exporter│        │
│  └─────────────────────┘      └─────────────────────┘        │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│              External Redis Cluster                            │
│  ┌──────────────────┐ ┌──────────────────┐ ┌───────────────┐ │
│  │ VM1: Master      │ │ VM2: Replica     │ │ VM3: Sentinel │ │
│  │ • Redis master   │ │ • Redis replica  │ │ • Sentinel    │ │
│  │ • Sentinel       │ │ • Sentinel       │ │   only        │ │
│  │ • redis_exporter │ │ • redis_exporter │ │               │ │
│  └──────────────────┘ └──────────────────┘ └───────────────┘ │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│              External Monitoring Server                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  • Prometheus (scrapes K3s + DB + Redis)               │  │
│  │  • Grafana (dashboards)                                │  │
│  │  • AlertManager (notifications)                        │  │
│  │  • Loki (log aggregation - optional)                   │  │
│  └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│              S3 Object Storage                                 │
│  • Nextcloud files                                             │
│  • Database backups (pgBackRest)                               │
│  • Application backups                                         │
│  • Log archives                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 🗄️ Part 1: PostgreSQL High Availability Setup

### 1.1 Prerequisites

**Hardware Requirements (per VM):**
- **VM1 (Primary)**: 4 vCPU, 8GB RAM, 100GB SSD
- **VM2 (Standby)**: 4 vCPU, 8GB RAM, 100GB SSD
- **OS**: Ubuntu 22.04 LTS

**Network Requirements:**
- PostgreSQL port: **5432** (between VMs and from K3s nodes)
- Patroni REST API: **8008** (between VMs)
- etcd: **2379, 2380** (if using separate etcd cluster)

### 1.2 Install PostgreSQL 15+

Run on **both VM1 and VM2**:

```bash
# Add PostgreSQL repository
sudo apt update
sudo apt install -y wget ca-certificates
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list

# Install PostgreSQL 15 and required packages
sudo apt update
sudo apt install -y postgresql-15 postgresql-contrib-15 postgresql-15-pgvector

# Stop PostgreSQL (Patroni will manage it)
sudo systemctl stop postgresql
sudo systemctl disable postgresql

# Verify installation
psql --version
# Expected output: psql (PostgreSQL) 15.x
```

### 1.3 Install Patroni (HA Manager)

Run on **both VM1 and VM2**:

```bash
# Install Patroni and dependencies
sudo apt install -y python3-pip python3-dev libpq-dev
sudo pip3 install patroni[etcd] psycopg2-binary python-etcd

# Verify Patroni installation
patroni --version
```

### 1.4 Setup etcd (Distributed Consensus)

**Option A: Use K3s embedded etcd** (simplest for small setups)
- K3s already runs etcd on server nodes
- Connect Patroni to K3s etcd: `http://<k3s-node>:2379`

**Option B: Dedicated etcd cluster** (recommended for production)

Run on a **separate VM or on VM1**:

```bash
# Install etcd
ETCD_VER=v3.5.10
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzf etcd-${ETCD_VER}-linux-amd64.tar.gz
sudo mv etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/
rm -rf etcd-${ETCD_VER}-linux-amd64*

# Create etcd user and directories
sudo useradd --system --no-create-home etcd
sudo mkdir -p /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd

# Create etcd systemd service
sudo tee /etc/systemd/system/etcd.service > /dev/null <<EOF
[Unit]
Description=etcd key-value store
After=network.target

[Service]
User=etcd
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name etcd1 \\
  --data-dir /var/lib/etcd \\
  --listen-client-urls http://0.0.0.0:2379 \\
  --advertise-client-urls http://$(hostname -I | awk '{print $1}'):2379 \\
  --listen-peer-urls http://0.0.0.0:2380
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
EOF

# Start etcd
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd

# Verify etcd
etcdctl --endpoints=http://localhost:2379 member list
```

### 1.5 Configure Patroni

**On VM1 (Primary)**, create `/etc/patroni/patroni.yml`:

```yaml
scope: postgres-cluster
namespace: /db/
name: postgres1

restapi:
  listen: 0.0.0.0:8008
  connect_address: <VM1_IP>:8008

etcd:
  hosts: <ETCD_IP>:2379  # or K3s node IP if using K3s etcd

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 200
        shared_buffers: 2GB
        effective_cache_size: 6GB
        maintenance_work_mem: 512MB
        checkpoint_completion_target: 0.9
        wal_buffers: 16MB
        default_statistics_target: 100
        random_page_cost: 1.1
        effective_io_concurrency: 200
        work_mem: 10MB
        min_wal_size: 1GB
        max_wal_size: 4GB
        max_worker_processes: 4
        max_parallel_workers_per_gather: 2
        max_parallel_workers: 4
        max_parallel_maintenance_workers: 2

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator 0.0.0.0/0 md5
    - host all all 0.0.0.0/0 md5
    - host all all ::0/0 md5

  users:
    admin:
      password: <CHANGE_ME_ADMIN_PASSWORD>
      options:
        - createrole
        - createdb
    replicator:
      password: <CHANGE_ME_REPLICATOR_PASSWORD>
      options:
        - replication

postgresql:
  listen: 0.0.0.0:5432
  connect_address: <VM1_IP>:5432
  data_dir: /var/lib/postgresql/15/main
  bin_dir: /usr/lib/postgresql/15/bin
  authentication:
    replication:
      username: replicator
      password: <CHANGE_ME_REPLICATOR_PASSWORD>
    superuser:
      username: postgres
      password: <CHANGE_ME_POSTGRES_PASSWORD>
    rewind:
      username: replicator
      password: <CHANGE_ME_REPLICATOR_PASSWORD>

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

**On VM2 (Standby)**, create similar `/etc/patroni/patroni.yml` but change:
- `name: postgres2`
- `connect_address: <VM2_IP>:8008`
- `connect_address: <VM2_IP>:5432`

### 1.6 Create Patroni systemd service

Run on **both VM1 and VM2**:

```bash
# Set permissions
sudo chown -R postgres:postgres /var/lib/postgresql
sudo chmod 700 /var/lib/postgresql/15/main

# Create Patroni systemd service
sudo tee /etc/systemd/system/patroni.service > /dev/null <<EOF
[Unit]
Description=Patroni PostgreSQL Cluster Manager
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -HUP \$MAINPID
KillMode=process
TimeoutSec=30
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Start Patroni
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni

# Check status
sudo systemctl status patroni
```

### 1.7 Verify PostgreSQL Cluster

```bash
# Check cluster status
patronictl -c /etc/patroni/patroni.yml list

# Expected output:
# + Cluster: postgres-cluster --------+----+-----------+
# | Member    | Host       | Role    | State   | TL | Lag in MB |
# +-----------+------------+---------+---------+----+-----------+
# | postgres1 | <VM1_IP>   | Leader  | running |  1 |           |
# | postgres2 | <VM2_IP>   | Replica | running |  1 |         0 |
# +-----------+------------+---------+---------+----+-----------+

# Test failover (optional)
patronictl -c /etc/patroni/patroni.yml failover --master postgres1 --candidate postgres2
```

### 1.8 Install PgBouncer (Connection Pooling)

Run on **both VM1 and VM2**:

```bash
# Install PgBouncer
sudo apt install -y pgbouncer

# Configure PgBouncer
sudo tee /etc/pgbouncer/pgbouncer.ini > /dev/null <<EOF
[databases]
* = host=127.0.0.1 port=5432

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3
max_db_connections = 100
max_user_connections = 100
log_connections = 1
log_disconnections = 1
EOF

# Create userlist (add application users here)
sudo tee /etc/pgbouncer/userlist.txt > /dev/null <<EOF
"postgres" "<CHANGE_ME_POSTGRES_PASSWORD_MD5>"
EOF

# Start PgBouncer
sudo systemctl enable pgbouncer
sudo systemctl restart pgbouncer
```

**Applications should connect to PgBouncer port 6432, not PostgreSQL port 5432.**

### 1.9 Create Application Databases

Connect to PostgreSQL primary:

```bash
# Connect to primary node
psql -h <VM1_IP> -U postgres -p 5432

-- Create databases for each service
CREATE DATABASE nextcloud_db;
CREATE DATABASE odoo_db;
CREATE DATABASE keycloak_db;
CREATE DATABASE mautic_db;
CREATE DATABASE whmcs_db;

-- Create application user (same user for all DBs or separate users)
CREATE USER app_user WITH PASSWORD '<CHANGE_ME_APP_USER_PASSWORD>';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE nextcloud_db TO app_user;
GRANT ALL PRIVILEGES ON DATABASE odoo_db TO app_user;
GRANT ALL PRIVILEGES ON DATABASE keycloak_db TO app_user;
GRANT ALL PRIVILEGES ON DATABASE mautic_db TO app_user;
GRANT ALL PRIVILEGES ON DATABASE whmcs_db TO app_user;

\q
```

### 1.10 Setup Backups with pgBackRest

Install on **VM1 (primary)**:

```bash
# Install pgBackRest
sudo apt install -y pgbackrest

# Configure pgBackRest for S3 backups
sudo tee /etc/pgbackrest.conf > /dev/null <<EOF
[global]
repo1-type=s3
repo1-s3-endpoint=<S3_ENDPOINT>
repo1-s3-bucket=<BUCKET_NAME>
repo1-s3-region=<REGION>
repo1-s3-key=<ACCESS_KEY>
repo1-s3-key-secret=<SECRET_KEY>
repo1-retention-full=7
repo1-retention-diff=7
repo1-retention-archive=7
process-max=4
log-level-console=info
log-level-file=debug
start-fast=y

[postgres-cluster]
pg1-path=/var/lib/postgresql/15/main
pg1-port=5432
pg1-user=postgres
EOF

# Initialize repository
sudo -u postgres pgbackrest --stanza=postgres-cluster stanza-create

# Run initial full backup
sudo -u postgres pgbackrest --stanza=postgres-cluster --type=full backup

# Setup cron for automated backups
sudo tee /etc/cron.d/pgbackrest > /dev/null <<EOF
# Daily full backup at 2 AM
0 2 * * * postgres pgbackrest --stanza=postgres-cluster --type=full backup

# Hourly incremental backup
0 * * * * postgres pgbackrest --stanza=postgres-cluster --type=incr backup
EOF
```

### 1.11 Install postgres_exporter (Prometheus Metrics)

Run on **both VM1 and VM2**:

```bash
# Download and install postgres_exporter
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.15.0/postgres_exporter-0.15.0.linux-amd64.tar.gz
tar xzf postgres_exporter-0.15.0.linux-amd64.tar.gz
sudo mv postgres_exporter-0.15.0.linux-amd64/postgres_exporter /usr/local/bin/
rm -rf postgres_exporter-*

# Create postgres_exporter user in PostgreSQL
psql -h localhost -U postgres -c "CREATE USER postgres_exporter WITH PASSWORD '<EXPORTER_PASSWORD>';"
psql -h localhost -U postgres -c "ALTER USER postgres_exporter SET SEARCH_PATH TO postgres_exporter,pg_catalog;"
psql -h localhost -U postgres -c "GRANT pg_monitor TO postgres_exporter;"

# Create systemd service
sudo tee /etc/systemd/system/postgres_exporter.service > /dev/null <<EOF
[Unit]
Description=Prometheus PostgreSQL Exporter
After=network.target

[Service]
Type=simple
User=postgres
Environment="DATA_SOURCE_NAME=postgresql://postgres_exporter:<EXPORTER_PASSWORD>@localhost:5432/postgres?sslmode=disable"
ExecStart=/usr/local/bin/postgres_exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Start postgres_exporter
sudo systemctl daemon-reload
sudo systemctl enable postgres_exporter
sudo systemctl start postgres_exporter

# Verify (should expose metrics on port 9187)
curl http://localhost:9187/metrics | grep pg_up
```

---

## 🚀 Part 2: Redis High Availability Setup

### 2.1 Prerequisites

**Hardware Requirements:**
- **VM1 (Redis master + Sentinel)**: 2 vCPU, 4GB RAM, 50GB SSD
- **VM2 (Redis replica + Sentinel)**: 2 vCPU, 4GB RAM, 50GB SSD
- **VM3 (Sentinel only)**: 1 vCPU, 2GB RAM, 20GB SSD
- **OS**: Ubuntu 22.04 LTS

**Network Requirements:**
- Redis: **6379**
- Redis Sentinel: **26379**

### 2.2 Install Redis

Run on **VM1 and VM2**:

```bash
# Install Redis
sudo apt update
sudo apt install -y redis-server

# Stop Redis (we'll configure it first)
sudo systemctl stop redis-server
```

### 2.3 Configure Redis Master (VM1)

Edit `/etc/redis/redis.conf` on VM1:

```bash
sudo tee /etc/redis/redis.conf > /dev/null <<EOF
bind 0.0.0.0
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize no
supervised systemd
pidfile /var/run/redis/redis-server.pid
loglevel notice
logfile /var/log/redis/redis-server.log
databases 16

# Persistence
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis

# AOF persistence
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# Security
requirepass <CHANGE_ME_REDIS_PASSWORD>
maxclients 10000

# Memory management
maxmemory 3gb
maxmemory-policy allkeys-lru
maxmemory-samples 5

# Replication
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
replica-priority 100
EOF

# Start Redis master
sudo systemctl enable redis-server
sudo systemctl restart redis-server
sudo systemctl status redis-server
```

### 2.4 Configure Redis Replica (VM2)

Edit `/etc/redis/redis.conf` on VM2 (same as master but add replication):

```bash
# Copy master config from VM1
# Then add these lines:
sudo tee -a /etc/redis/redis.conf > /dev/null <<EOF

# Replication settings
replicaof <VM1_IP> 6379
masterauth <CHANGE_ME_REDIS_PASSWORD>
EOF

# Start Redis replica
sudo systemctl enable redis-server
sudo systemctl restart redis-server
```

### 2.5 Verify Replication

On VM1 (master):

```bash
redis-cli -a <REDIS_PASSWORD>
127.0.0.1:6379> INFO replication
# Should show:
# role:master
# connected_slaves:1
# slave0:ip=<VM2_IP>,port=6379,state=online
```

On VM2 (replica):

```bash
redis-cli -a <REDIS_PASSWORD>
127.0.0.1:6379> INFO replication
# Should show:
# role:slave
# master_host:<VM1_IP>
# master_link_status:up
```

### 2.6 Configure Redis Sentinel

Run on **VM1, VM2, VM3**:

```bash
# Create Sentinel config
sudo tee /etc/redis/sentinel.conf > /dev/null <<EOF
port 26379
daemonize no
pidfile /var/run/redis/redis-sentinel.pid
logfile /var/log/redis/redis-sentinel.log
dir /var/lib/redis

# Monitor the master
sentinel monitor mymaster <VM1_IP> 6379 2
sentinel auth-pass mymaster <CHANGE_ME_REDIS_PASSWORD>
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 10000

# Notification scripts (optional)
# sentinel notification-script mymaster /path/to/notification.sh
# sentinel client-reconfig-script mymaster /path/to/reconfig.sh
EOF

# Set permissions
sudo chown redis:redis /etc/redis/sentinel.conf
sudo chmod 640 /etc/redis/sentinel.conf

# Create systemd service for Sentinel
sudo tee /etc/systemd/system/redis-sentinel.service > /dev/null <<EOF
[Unit]
Description=Redis Sentinel
After=network.target
After=redis-server.service

[Service]
Type=notify
User=redis
Group=redis
ExecStart=/usr/bin/redis-sentinel /etc/redis/sentinel.conf --supervised systemd
ExecStop=/bin/kill -s TERM \$MAINPID
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Start Sentinel
sudo systemctl daemon-reload
sudo systemctl enable redis-sentinel
sudo systemctl start redis-sentinel
sudo systemctl status redis-sentinel
```

### 2.7 Verify Sentinel

```bash
# Check Sentinel status on any VM
redis-cli -p 26379
127.0.0.1:26379> SENTINEL masters
127.0.0.1:26379> SENTINEL slaves mymaster
127.0.0.1:26379> SENTINEL sentinels mymaster
127.0.0.1:26379> SENTINEL get-master-addr-by-name mymaster

# Should show current master IP
```

### 2.8 Test Failover

On VM1 (master):

```bash
# Stop Redis master to trigger failover
sudo systemctl stop redis-server

# Wait 10-15 seconds, then check Sentinel
redis-cli -p 26379 -h <VM2_IP>
127.0.0.1:26379> SENTINEL get-master-addr-by-name mymaster
# Should now point to VM2

# Restart VM1 - it will become a replica
sudo systemctl start redis-server
```

### 2.9 Install redis_exporter (Prometheus Metrics)

Run on **VM1 and VM2**:

```bash
# Download redis_exporter
wget https://github.com/oliver006/redis_exporter/releases/download/v1.55.0/redis_exporter-v1.55.0.linux-amd64.tar.gz
tar xzf redis_exporter-v1.55.0.linux-amd64.tar.gz
sudo mv redis_exporter-v1.55.0.linux-amd64/redis_exporter /usr/local/bin/
rm -rf redis_exporter-*

# Create systemd service
sudo tee /etc/systemd/system/redis_exporter.service > /dev/null <<EOF
[Unit]
Description=Redis Exporter for Prometheus
After=network.target

[Service]
Type=simple
User=redis
Environment="REDIS_ADDR=localhost:6379"
Environment="REDIS_PASSWORD=<CHANGE_ME_REDIS_PASSWORD>"
ExecStart=/usr/local/bin/redis_exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Start redis_exporter
sudo systemctl daemon-reload
sudo systemctl enable redis_exporter
sudo systemctl start redis_exporter

# Verify (metrics on port 9121)
curl http://localhost:9121/metrics | grep redis_up
```

### 2.10 Application Connection Configuration

Applications should connect to Redis via **Sentinel** for automatic failover:

**Python (redis-py) example:**
```python
from redis.sentinel import Sentinel

sentinel = Sentinel([
    ('<VM1_IP>', 26379),
    ('<VM2_IP>', 26379),
    ('<VM3_IP>', 26379)
], socket_timeout=0.5)

# Get master for writes
master = sentinel.master_for(
    'mymaster',
    password='<REDIS_PASSWORD>',
    socket_timeout=0.5
)

# Get slave for reads (optional)
slave = sentinel.slave_for(
    'mymaster',
    password='<REDIS_PASSWORD>',
    socket_timeout=0.5
)

# Use master for writes
master.set('foo', 'bar')

# Use slave for reads (reduces master load)
slave.get('foo')
```

**Environment variables for K3s pods:**
```yaml
env:
  - name: REDIS_SENTINELS
    value: "<VM1_IP>:26379,<VM2_IP>:26379,<VM3_IP>:26379"
  - name: REDIS_MASTER_NAME
    value: "mymaster"
  - name: REDIS_PASSWORD
    valueFrom:
      secretKeyRef:
        name: redis-secret
        key: password
```

---

## 📊 Part 3: Monitoring Stack Setup (Prometheus, Grafana)

### 3.1 Prerequisites

**Hardware Requirements:**
- **Monitoring VM**: 4 vCPU, 8GB RAM, 100GB SSD
- **OS**: Ubuntu 22.04 LTS

**Network Requirements:**
- Prometheus: **9090**
- Grafana: **3000**
- AlertManager: **9093**
- Loki: **3100** (optional)

### 3.2 Install Prometheus

```bash
# Create prometheus user
sudo useradd --system --no-create-home --shell /bin/false prometheus

# Download Prometheus
PROM_VERSION="2.48.0"
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz
tar xzf prometheus-${PROM_VERSION}.linux-amd64.tar.gz
cd prometheus-${PROM_VERSION}.linux-amd64

# Install binaries
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/

# Create directories
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

# Create Prometheus configuration
sudo tee /etc/prometheus/prometheus.yml > /dev/null <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'digital-forest'

# AlertManager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - localhost:9093

# Load rules
rule_files:
  - "/etc/prometheus/rules/*.yml"

scrape_configs:
  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # K3s cluster monitoring
  - job_name: 'k3s-nodes'
    static_configs:
      - targets:
          - '<K3S_NODE1_IP>:9100'  # node-exporter
          - '<K3S_NODE2_IP>:9100'
          - '<K3S_NODE3_IP>:9100'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

  # K3s kubelet metrics
  - job_name: 'k3s-kubelet'
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/k3s-ca.crt
      cert_file: /etc/prometheus/k3s-client.crt
      key_file: /etc/prometheus/k3s-client.key
    static_configs:
      - targets:
          - '<K3S_NODE1_IP>:10250'
          - '<K3S_NODE2_IP>:10250'
          - '<K3S_NODE3_IP>:10250'

  # PostgreSQL monitoring
  - job_name: 'postgresql'
    static_configs:
      - targets:
          - '<POSTGRES_VM1_IP>:9187'
          - '<POSTGRES_VM2_IP>:9187'

  # Redis monitoring
  - job_name: 'redis'
    static_configs:
      - targets:
          - '<REDIS_VM1_IP>:9121'
          - '<REDIS_VM2_IP>:9121'

  # kube-state-metrics (deploy in K3s)
  - job_name: 'kube-state-metrics'
    static_configs:
      - targets:
          - '<K3S_NODE1_IP>:8080'  # Adjust port based on deployment
EOF

# Create alerts directory
sudo mkdir -p /etc/prometheus/rules

# Create basic alert rules
sudo tee /etc/prometheus/rules/alerts.yml > /dev/null <<'EOF'
groups:
  - name: instances
    interval: 30s
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{ $labels.instance }} has been down for more than 5 minutes."

  - name: postgresql
    interval: 30s
    rules:
      - alert: PostgreSQLDown
        expr: pg_up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL {{ $labels.instance }} down"

      - alert: PostgreSQLReplicationLag
        expr: pg_replication_lag > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL replication lag on {{ $labels.instance }}"

  - name: redis
    interval: 30s
    rules:
      - alert: RedisDown
        expr: redis_up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Redis {{ $labels.instance }} down"

      - alert: RedisMasterSwitched
        expr: changes(redis_instance_info{role="master"}[5m]) > 0
        labels:
          severity: warning
        annotations:
          summary: "Redis master switched on {{ $labels.instance }}"
EOF

# Set permissions
sudo chown -R prometheus:prometheus /etc/prometheus

# Create systemd service
sudo tee /etc/systemd/system/prometheus.service > /dev/null <<EOF
[Unit]
Description=Prometheus Monitoring System
After=network.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \\
  --config.file=/etc/prometheus/prometheus.yml \\
  --storage.tsdb.path=/var/lib/prometheus \\
  --storage.tsdb.retention.time=30d \\
  --web.console.templates=/etc/prometheus/consoles \\
  --web.console.libraries=/etc/prometheus/console_libraries \\
  --web.listen-address=0.0.0.0:9090
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Start Prometheus
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus

# Verify
curl http://localhost:9090/-/healthy
```

### 3.3 Install Grafana

```bash
# Add Grafana repository
sudo apt install -y software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

# Install Grafana
sudo apt update
sudo apt install -y grafana

# Start Grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server

# Access Grafana at http://<MONITORING_VM_IP>:3000
# Default credentials: admin / admin (change on first login)
```

### 3.4 Configure Grafana Data Sources

1. Open Grafana web UI: `http://<MONITORING_VM_IP>:3000`
2. Login with `admin` / `admin` (change password)
3. Go to **Configuration → Data Sources → Add data source**
4. Select **Prometheus**:
   - Name: `Prometheus`
   - URL: `http://localhost:9090`
   - Click **Save & Test**

### 3.5 Import Grafana Dashboards

**Pre-built dashboards:**
1. Go to **Dashboards → Import**
2. Import these dashboard IDs:
   - **Node Exporter Full**: 1860
   - **PostgreSQL Database**: 9628
   - **Redis Dashboard**: 11835
   - **Kubernetes Cluster Monitoring**: 7249

### 3.6 Install AlertManager

```bash
# Download AlertManager
ALERT_VERSION="0.26.0"
wget https://github.com/prometheus/alertmanager/releases/download/v${ALERT_VERSION}/alertmanager-${ALERT_VERSION}.linux-amd64.tar.gz
tar xzf alertmanager-${ALERT_VERSION}.linux-amd64.tar.gz
cd alertmanager-${ALERT_VERSION}.linux-amd64

# Install binary
sudo mv alertmanager amtool /usr/local/bin/

# Create directories
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
sudo chown -R prometheus:prometheus /etc/alertmanager /var/lib/alertmanager

# Configure AlertManager
sudo tee /etc/alertmanager/alertmanager.yml > /dev/null <<'EOF'
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alerts@digitalforest.eu'
  smtp_auth_username: 'alerts@digitalforest.eu'
  smtp_auth_password: '<EMAIL_PASSWORD>'
  smtp_require_tls: true

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'email-admin'
  routes:
    - match:
        severity: critical
      receiver: 'email-admin'
      continue: true
    - match:
        severity: warning
      receiver: 'email-team'

receivers:
  - name: 'email-admin'
    email_configs:
      - to: 'admin@digitalforest.eu'
        headers:
          Subject: '[CRITICAL] {{ .GroupLabels.alertname }}'
  - name: 'email-team'
    email_configs:
      - to: 'team@digitalforest.eu'
        headers:
          Subject: '[WARNING] {{ .GroupLabels.alertname }}'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
EOF

# Set permissions
sudo chown -R prometheus:prometheus /etc/alertmanager

# Create systemd service
sudo tee /etc/systemd/system/alertmanager.service > /dev/null <<EOF
[Unit]
Description=Prometheus AlertManager
After=network.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/alertmanager \\
  --config.file=/etc/alertmanager/alertmanager.yml \\
  --storage.path=/var/lib/alertmanager \\
  --web.listen-address=0.0.0.0:9093
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Start AlertManager
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
sudo systemctl status alertmanager
```

### 3.7 Install node_exporter on All Servers

Run on **all K3s nodes, PostgreSQL VMs, Redis VMs**:

```bash
# Download node_exporter
NODE_EXP_VERSION="1.7.0"
wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXP_VERSION}/node_exporter-${NODE_EXP_VERSION}.linux-amd64.tar.gz
tar xzf node_exporter-${NODE_EXP_VERSION}.linux-amd64.tar.gz
sudo mv node_exporter-${NODE_EXP_VERSION}.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter-*

# Create systemd service
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
Type=simple
User=nobody
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Start node_exporter
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

# Verify (metrics on port 9100)
curl http://localhost:9100/metrics | head
```

---

## 🔐 Part 4: Security Configuration

### 4.1 Firewall Rules

**On PostgreSQL VMs:**
```bash
# Allow K3s nodes to connect to PostgreSQL
sudo ufw allow from <K3S_NODE1_IP> to any port 5432
sudo ufw allow from <K3S_NODE2_IP> to any port 5432
sudo ufw allow from <K3S_NODE3_IP> to any port 5432
sudo ufw allow from <K3S_NODE1_IP> to any port 6432  # PgBouncer
sudo ufw allow from <K3S_NODE2_IP> to any port 6432
sudo ufw allow from <K3S_NODE3_IP> to any port 6432

# Allow Patroni communication between PostgreSQL VMs
sudo ufw allow from <POSTGRES_VM1_IP> to any port 8008
sudo ufw allow from <POSTGRES_VM2_IP> to any port 8008

# Allow Prometheus to scrape postgres_exporter
sudo ufw allow from <MONITORING_VM_IP> to any port 9187

# Enable firewall
sudo ufw enable
```

**On Redis VMs:**
```bash
# Allow K3s nodes to connect to Redis
sudo ufw allow from <K3S_NODE1_IP> to any port 6379
sudo ufw allow from <K3S_NODE2_IP> to any port 6379
sudo ufw allow from <K3S_NODE3_IP> to any port 6379

# Allow Sentinel communication
sudo ufw allow from <REDIS_VM1_IP> to any port 26379
sudo ufw allow from <REDIS_VM2_IP> to any port 26379
sudo ufw allow from <REDIS_VM3_IP> to any port 26379

# Allow Prometheus to scrape redis_exporter
sudo ufw allow from <MONITORING_VM_IP> to any port 9121

sudo ufw enable
```

**On Monitoring VM:**
```bash
# Allow access to Prometheus from K3s nodes (for queries)
sudo ufw allow from <K3S_NODE1_IP> to any port 9090
sudo ufw allow from <K3S_NODE2_IP> to any port 9090
sudo ufw allow from <K3S_NODE3_IP> to any port 9090

# Allow Grafana access (restrict to specific IPs if possible)
sudo ufw allow 3000/tcp

# Allow Prometheus to scrape K3s nodes
sudo ufw allow from any to any port 9100  # node_exporter

sudo ufw enable
```

### 4.2 SSL/TLS for PostgreSQL

Enable SSL on PostgreSQL:

```bash
# Generate self-signed certificate (or use Let's Encrypt)
sudo -u postgres openssl req -new -x509 -days 365 -nodes -text \
  -out /var/lib/postgresql/15/main/server.crt \
  -keyout /var/lib/postgresql/15/main/server.key \
  -subj "/CN=postgres-cluster"

sudo -u postgres chmod 600 /var/lib/postgresql/15/main/server.key

# Enable SSL in postgresql.conf
echo "ssl = on" | sudo tee -a /var/lib/postgresql/15/main/postgresql.conf
echo "ssl_cert_file = 'server.crt'" | sudo tee -a /var/lib/postgresql/15/main/postgresql.conf
echo "ssl_key_file = 'server.key'" | sudo tee -a /var/lib/postgresql/15/main/postgresql.conf

# Restart PostgreSQL
sudo systemctl restart patroni
```

Update K3s connection strings to use SSL:
```
postgresql://user:password@host:5432/db?sslmode=require
```

### 4.3 Network Segmentation

**Recommended network setup:**
- **K3s Cluster**: VLAN 10 (10.0.10.0/24)
- **Database VMs**: VLAN 20 (10.0.20.0/24)
- **Redis VMs**: VLAN 30 (10.0.30.0/24)
- **Monitoring VM**: VLAN 40 (10.0.40.0/24)

Configure PlanetHoster private networking or VPN for secure communication.

---

## 🔄 Part 5: Backup and Disaster Recovery

### 5.1 PostgreSQL Backup Strategy

**Automated with pgBackRest (already configured above):**
- **Daily full backups** to S3
- **Hourly incremental backups** to S3
- **30-day retention** (configurable)

**Manual backup:**
```bash
# Full backup
sudo -u postgres pgbackrest --stanza=postgres-cluster --type=full backup

# Differential backup
sudo -u postgres pgbackrest --stanza=postgres-cluster --type=diff backup

# List backups
sudo -u postgres pgbackrest --stanza=postgres-cluster info
```

**Restore from backup:**
```bash
# Stop Patroni
sudo systemctl stop patroni

# Restore from latest backup
sudo -u postgres pgbackrest --stanza=postgres-cluster --delta restore

# Start Patroni
sudo systemctl start patroni
```

### 5.2 Redis Backup Strategy

**Automated RDB snapshots:**
- Redis automatically saves RDB snapshots based on config
- Configured in `redis.conf`: `save 900 1` (15 min if ≥1 key changed)

**Manual backup:**
```bash
# Trigger immediate save
redis-cli -a <REDIS_PASSWORD> BGSAVE

# Copy RDB file to S3
aws s3 cp /var/lib/redis/dump.rdb s3://<BUCKET>/redis-backups/dump-$(date +%Y%m%d).rdb
```

**Setup automated backups to S3:**
```bash
# Create backup script
sudo tee /usr/local/bin/redis-backup.sh > /dev/null <<'EOF'
#!/bin/bash
REDIS_PASSWORD="<CHANGE_ME>"
BUCKET="<S3_BUCKET>"
DATE=$(date +%Y%m%d-%H%M%S)

# Trigger BGSAVE
redis-cli -a $REDIS_PASSWORD BGSAVE

# Wait for save to complete
sleep 10

# Copy to S3
aws s3 cp /var/lib/redis/dump.rdb s3://$BUCKET/redis-backups/dump-$DATE.rdb

# Keep only last 30 days
aws s3 ls s3://$BUCKET/redis-backups/ | awk '{print $4}' | sort -r | tail -n +31 | \
  xargs -I {} aws s3 rm s3://$BUCKET/redis-backups/{}
EOF

sudo chmod +x /usr/local/bin/redis-backup.sh

# Add to crontab (daily at 3 AM)
echo "0 3 * * * root /usr/local/bin/redis-backup.sh" | sudo tee /etc/cron.d/redis-backup
```

### 5.3 Monitoring Data Backup

**Prometheus data:**
- Stored in `/var/lib/prometheus`
- Configured with 30-day retention
- For long-term storage, use **Thanos** or **Cortex** (advanced)

**Grafana backup:**
```bash
# Backup Grafana dashboards
sudo tar czf grafana-backup-$(date +%Y%m%d).tar.gz /var/lib/grafana /etc/grafana

# Upload to S3
aws s3 cp grafana-backup-$(date +%Y%m%d).tar.gz s3://<BUCKET>/grafana-backups/
```

---

## ✅ Part 6: Verification Checklist

### PostgreSQL Cluster
- [ ] Both PostgreSQL VMs running
- [ ] Patroni showing Leader + Replica
- [ ] Replication lag < 1MB
- [ ] PgBouncer accepting connections on port 6432
- [ ] Application databases created
- [ ] pgBackRest full backup completed
- [ ] postgres_exporter exposing metrics on port 9187
- [ ] Firewall rules configured

### Redis Cluster
- [ ] Redis master and replica running
- [ ] Redis replication working (check `INFO replication`)
- [ ] All 3 Sentinels running
- [ ] Sentinel monitoring master
- [ ] Failover test successful
- [ ] redis_exporter exposing metrics on port 9121
- [ ] Firewall rules configured

### Monitoring Stack
- [ ] Prometheus scraping all targets
- [ ] Grafana accessible on port 3000
- [ ] Data sources configured in Grafana
- [ ] Dashboards imported
- [ ] AlertManager receiving alerts from Prometheus
- [ ] Test alert sent successfully
- [ ] node_exporter running on all servers

### Connectivity
- [ ] K3s pods can connect to PostgreSQL via PgBouncer
- [ ] K3s pods can connect to Redis via Sentinel
- [ ] Prometheus can scrape all exporters
- [ ] SSL/TLS configured for PostgreSQL connections

---

## 🚀 Next Steps

1. **Update K3s application deployments** to use external services:
   - Update Nextcloud, Odoo, Keycloak, Mautic manifests
   - Add PostgreSQL and Redis connection strings
   - Create Kubernetes Secrets for passwords

2. **Deploy kube-state-metrics** in K3s for cluster metrics

3. **Configure S3 storage** for Nextcloud files

4. **Setup monitoring dashboards** in Grafana

5. **Test disaster recovery procedures**

---

## 📚 Additional Resources

- **Patroni Documentation**: https://patroni.readthedocs.io/
- **Redis Sentinel Documentation**: https://redis.io/docs/management/sentinel/
- **Prometheus Documentation**: https://prometheus.io/docs/
- **Grafana Documentation**: https://grafana.com/docs/
- **pgBackRest Documentation**: https://pgbackrest.org/

---

**Need help?** See [K3S_SETUP_GUIDE.md](K3S_SETUP_GUIDE.md) for K3s cluster setup and [ARCHITECTURE_PLAN.md](ARCHITECTURE_PLAN.md) for overall architecture details.
