# PlanetHoster Setup Guide - Step by Step

## 📋 Prerequisites

- PlanetHoster account with Hybrid Cloud access
- Domain name registered
- SSH key pair generated
- Basic Linux/Docker knowledge

---

## 🚀 Phase 1: Initial Server Provisioning

### Step 1: Create Servers on PlanetHoster

#### Server 1: Load Balancer + Application Server (AZ1)
```
Name: app-server-01
OS: Ubuntu 22.04 LTS
vCPU: 8
RAM: 16 GB
Storage: 200 GB SSD
Location: Europe (Paris or Amsterdam)
```

#### Server 2: Load Balancer + Application Server (AZ2)
```
Name: app-server-02
OS: Ubuntu 22.04 LTS
vCPU: 8
RAM: 16 GB
Storage: 200 GB SSD
Location: Europe (Different datacenter from Server 1)
```

#### Server 3: Database Server
```
Name: db-server-01
OS: Ubuntu 22.04 LTS
vCPU: 4
RAM: 8 GB
Storage: 100 GB SSD
Location: Europe (Same as app-server-01)
```

---

## 🔒 Step 2: Initial Security Setup

### On ALL servers, run:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Create non-root user
sudo adduser deploy
sudo usermod -aG sudo deploy

# Setup SSH key authentication
sudo mkdir -p /home/deploy/.ssh
sudo nano /home/deploy/.ssh/authorized_keys
# Paste your SSH public key here

# Set correct permissions
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh

# Disable password authentication
sudo nano /etc/ssh/sshd_config
```

Edit `/etc/ssh/sshd_config`:
```
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```

```bash
# Restart SSH
sudo systemctl restart sshd

# Setup firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# Install fail2ban
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

---

## 🐳 Step 3: Install Docker & Docker Compose

### On ALL servers:

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker deploy

# Install Docker Compose
sudo apt install docker-compose-plugin -y

# Start Docker
sudo systemctl enable docker
sudo systemctl start docker

# Verify installation
docker --version
docker compose version
```

---

## 🔧 Step 4: Setup Docker Swarm (Container Orchestration)

### On app-server-01 (Manager Node):

```bash
# Initialize Docker Swarm
docker swarm init --advertise-addr <SERVER_1_PRIVATE_IP>

# Save the join token - you'll need it for other nodes
docker swarm join-token worker
```

### On app-server-02 and db-server-01 (Worker Nodes):

```bash
# Join the swarm (use the token from previous step)
docker swarm join --token <TOKEN> <SERVER_1_PRIVATE_IP>:2377
```

### Verify the cluster:

```bash
# On app-server-01
docker node ls
```

You should see all 3 nodes listed.

---

## 🌐 Step 5: Setup Load Balancers (HAProxy)

### Create HAProxy configuration directory:

```bash
# On app-server-01 and app-server-02
sudo mkdir -p /opt/haproxy
cd /opt/haproxy
```

### Create `haproxy.cfg`:

```bash
sudo nano haproxy.cfg
```

```haproxy
global
    maxconn 4096
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    option  http-server-close
    option  forwardfor except 127.0.0.0/8
    option  redispatch
    retries 3
    timeout connect 5000
    timeout client  50000
    timeout server  50000
    errorfile 400 /usr/local/etc/haproxy/errors/400.http
    errorfile 403 /usr/local/etc/haproxy/errors/403.http
    errorfile 408 /usr/local/etc/haproxy/errors/408.http
    errorfile 500 /usr/local/etc/haproxy/errors/500.http
    errorfile 502 /usr/local/etc/haproxy/errors/502.http
    errorfile 503 /usr/local/etc/haproxy/errors/503.http
    errorfile 504 /usr/local/etc/haproxy/errors/504.http

# Stats page
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats auth admin:your_secure_password_here

# HTTP to HTTPS redirect
frontend http_front
    bind *:80
    redirect scheme https code 301 if !{ ssl_fc }

# HTTPS frontend
frontend https_front
    bind *:443 ssl crt /etc/haproxy/certs/
    
    # ACLs for routing
    acl is_nextcloud hdr(host) -i nextcloud.yourplatform.eu
    acl is_odoo hdr(host) -i odoo.yourplatform.eu
    acl is_keycloak hdr(host) -i auth.yourplatform.eu
    acl is_mautic hdr(host) -i marketing.yourplatform.eu
    
    # Routing rules
    use_backend nextcloud_backend if is_nextcloud
    use_backend odoo_backend if is_odoo
    use_backend keycloak_backend if is_keycloak
    use_backend mautic_backend if is_mautic
    
    default_backend nextcloud_backend

# Backend definitions
backend nextcloud_backend
    balance roundrobin
    option httpchk GET /status.php
    http-check expect status 200
    server nextcloud1 nextcloud-1:80 check inter 5s
    server nextcloud2 nextcloud-2:80 check inter 5s backup

backend odoo_backend
    balance roundrobin
    option httpchk GET /web/health
    http-check expect status 200
    server odoo1 odoo-1:8069 check inter 5s
    server odoo2 odoo-2:8069 check inter 5s backup

backend keycloak_backend
    balance roundrobin
    option httpchk GET /health/ready
    http-check expect status 200
    server keycloak1 keycloak-1:8080 check inter 5s
    server keycloak2 keycloak-2:8080 check inter 5s backup

backend mautic_backend
    balance roundrobin
    option httpchk GET /
    http-check expect status 200
    server mautic1 mautic-1:80 check inter 5s
    server mautic2 mautic-2:80 check inter 5s backup
```

### Setup Keepalived for Virtual IP (High Availability):

```bash
# Install Keepalived
sudo apt install keepalived -y
```

### On app-server-01 (MASTER):

```bash
sudo nano /etc/keepalived/keepalived.conf
```

```conf
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0  # Change to your interface name
    virtual_router_id 51
    priority 101
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass your_secure_password
    }
    
    virtual_ipaddress {
        <YOUR_VIRTUAL_IP>/24  # Request this from PlanetHoster
    }
    
    track_script {
        chk_haproxy
    }
}
```

### On app-server-02 (BACKUP):

```bash
sudo nano /etc/keepalived/keepalived.conf
```

```conf
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0  # Change to your interface name
    virtual_router_id 51
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass your_secure_password
    }
    
    virtual_ipaddress {
        <YOUR_VIRTUAL_IP>/24  # Same as MASTER
    }
    
    track_script {
        chk_haproxy
    }
}
```

```bash
# Start Keepalived on both servers
sudo systemctl enable keepalived
sudo systemctl start keepalived
```

---

## 🗄️ Step 6: Setup PostgreSQL with Replication

### On db-server-01 (Primary):

```bash
# Install PostgreSQL
sudo apt install postgresql postgresql-contrib -y

# Configure PostgreSQL
sudo -u postgres psql
```

```sql
-- Create replication user
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'your_secure_password';

-- Create databases
CREATE DATABASE nextcloud;
CREATE DATABASE odoo;
CREATE DATABASE keycloak;
CREATE DATABASE mautic;

-- Create users
CREATE USER nextcloud_user WITH ENCRYPTED PASSWORD 'nextcloud_password';
CREATE USER odoo_user WITH ENCRYPTED PASSWORD 'odoo_password';
CREATE USER keycloak_user WITH ENCRYPTED PASSWORD 'keycloak_password';
CREATE USER mautic_user WITH ENCRYPTED PASSWORD 'mautic_password';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE nextcloud TO nextcloud_user;
GRANT ALL PRIVILEGES ON DATABASE odoo TO odoo_user;
GRANT ALL PRIVILEGES ON DATABASE keycloak TO keycloak_user;
GRANT ALL PRIVILEGES ON DATABASE mautic TO mautic_user;

\q
```

### Configure replication:

```bash
sudo nano /etc/postgresql/14/main/postgresql.conf
```

```conf
# Replication settings
wal_level = replica
max_wal_senders = 3
wal_keep_size = 64
synchronous_commit = on
```

```bash
sudo nano /etc/postgresql/14/main/pg_hba.conf
```

Add this line:
```
host    replication     replicator      <STANDBY_IP>/32         md5
```

```bash
# Restart PostgreSQL
sudo systemctl restart postgresql
```

---

## 💾 Step 7: Setup Object Storage (MinIO)

### Create MinIO deployment:

```bash
# Create directories
sudo mkdir -p /opt/minio/data
sudo mkdir -p /opt/minio/config

# Create docker-compose.yml
sudo nano /opt/minio/docker-compose.yml
```

```yaml
version: '3.8'

services:
  minio:
    image: minio/minio:latest
    container_name: minio
    restart: always
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: your_secure_password_here
    volumes:
      - ./data:/data
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

networks:
  default:
    name: sovereign_cloud
```

```bash
# Start MinIO
cd /opt/minio
docker compose up -d

# Check status
docker compose ps
```

---

## 📊 Step 8: Setup Monitoring Stack (Prometheus + Grafana)

### Create monitoring directory:

```bash
sudo mkdir -p /opt/monitoring
cd /opt/monitoring
```

### Create `docker-compose.yml`:

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
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=your_secure_password
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: always
    ports:
      - "9100:9100"

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: always
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

volumes:
  prometheus_data:
  grafana_data:

networks:
  default:
    name: sovereign_cloud
```

### Create `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'haproxy'
    static_configs:
      - targets: ['<APP_SERVER_01_IP>:8404', '<APP_SERVER_02_IP>:8404']
```

```bash
# Start monitoring stack
docker compose up -d

# Access Grafana at http://<SERVER_IP>:3000
# Default: admin / your_secure_password
```

---

## 🔐 Step 9: Setup SSL Certificates (Let's Encrypt)

```bash
# Install Certbot
sudo apt install certbot -y

# Generate certificates
sudo certbot certonly --standalone -d nextcloud.yourplatform.eu
sudo certbot certonly --standalone -d odoo.yourplatform.eu
sudo certbot certonly --standalone -d auth.yourplatform.eu
sudo certbot certonly --standalone -d marketing.yourplatform.eu

# Combine cert and key for HAProxy
sudo mkdir -p /etc/haproxy/certs
sudo cat /etc/letsencrypt/live/nextcloud.yourplatform.eu/fullchain.pem \
         /etc/letsencrypt/live/nextcloud.yourplatform.eu/privkey.pem \
         > /etc/haproxy/certs/nextcloud.yourplatform.eu.pem

# Repeat for other domains

# Setup auto-renewal
sudo crontab -e
```

Add this line:
```
0 0 1 * * certbot renew --deploy-hook "cat /etc/letsencrypt/live/*/fullchain.pem /etc/letsencrypt/live/*/privkey.pem > /etc/haproxy/certs/*.pem && systemctl reload haproxy"
```

---

## 📦 Step 10: Deploy Core Services

### Create main application directory:

```bash
sudo mkdir -p /opt/sovereign-cloud
cd /opt/sovereign-cloud
```

### Create Docker Swarm stack file `stack.yml`:

```yaml
version: '3.8'

services:
  # Redis for session management
  redis-master:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - backend
    deploy:
      placement:
        constraints:
          - node.role == manager

  # Keycloak - SSO/IAM
  keycloak-1:
    image: quay.io/keycloak/keycloak:latest
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=your_secure_password
      - KC_DB=postgres
      - KC_DB_URL=jdbc:postgresql://db-server-01:5432/keycloak
      - KC_DB_USERNAME=keycloak_user
      - KC_DB_PASSWORD=keycloak_password
      - KC_HOSTNAME=auth.yourplatform.eu
      - KC_PROXY=edge
    command: start
    networks:
      - backend
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s

  # Nextcloud
  nextcloud-1:
    image: nextcloud:latest
    environment:
      - POSTGRES_HOST=db-server-01
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=nextcloud_user
      - POSTGRES_PASSWORD=nextcloud_password
      - NEXTCLOUD_ADMIN_USER=admin
      - NEXTCLOUD_ADMIN_PASSWORD=your_secure_password
      - REDIS_HOST=redis-master
      - NEXTCLOUD_TRUSTED_DOMAINS=nextcloud.yourplatform.eu
      - OVERWRITEPROTOCOL=https
    volumes:
      - nextcloud_data:/var/www/html
    networks:
      - backend
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s

  # Odoo
  odoo-1:
    image: odoo:17
    environment:
      - HOST=db-server-01
      - USER=odoo_user
      - PASSWORD=odoo_password
    volumes:
      - odoo_data:/var/lib/odoo
    networks:
      - backend
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s

  # Mautic
  mautic-1:
    image: mautic/mautic:latest
    environment:
      - MAUTIC_DB_HOST=db-server-01
      - MAUTIC_DB_NAME=mautic
      - MAUTIC_DB_USER=mautic_user
      - MAUTIC_DB_PASSWORD=mautic_password
    volumes:
      - mautic_data:/var/www/html
    networks:
      - backend
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s

volumes:
  redis_data:
  nextcloud_data:
  odoo_data:
  mautic_data:

networks:
  backend:
    driver: overlay
```

### Deploy the stack:

```bash
docker stack deploy -c stack.yml sovereign-cloud

# Check deployment status
docker stack services sovereign-cloud

# View logs
docker service logs sovereign-cloud_nextcloud-1
```

---

## ✅ Step 11: Verification & Testing

### Check all services:

```bash
# Check Docker Swarm services
docker service ls

# Check HAProxy stats
curl http://<SERVER_IP>:8404/stats

# Check Prometheus targets
curl http://<SERVER_IP>:9090/targets

# Check Grafana
curl http://<SERVER_IP>:3000/api/health
```

### Test endpoints:

```bash
# Test Nextcloud
curl -I https://nextcloud.yourplatform.eu

# Test Odoo
curl -I https://odoo.yourplatform.eu

# Test Keycloak
curl -I https://auth.yourplatform.eu

# Test Mautic
curl -I https://marketing.yourplatform.eu
```

---

## 🔄 Step 12: Setup Backup Strategy

### Create backup script:

```bash
sudo mkdir -p /opt/backups
sudo nano /opt/backups/backup.sh
```

```bash
#!/bin/bash

BACKUP_DIR="/opt/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# Backup databases
docker exec postgres pg_dumpall -U postgres | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# Backup volumes
docker run --rm -v nextcloud_data:/data -v $BACKUP_DIR:/backup alpine tar czf /backup/nextcloud_$DATE.tar.gz /data
docker run --rm -v odoo_data:/data -v $BACKUP_DIR:/backup alpine tar czf /backup/odoo_$DATE.tar.gz /data
docker run --rm -v mautic_data:/data -v $BACKUP_DIR:/backup alpine tar czf /backup/mautic_$DATE.tar.gz /data

# Delete backups older than 30 days
find $BACKUP_DIR -name "*.gz" -mtime +30 -delete

echo "Backup completed: $DATE"
```

```bash
# Make executable
sudo chmod +x /opt/backups/backup.sh

# Schedule daily backups
sudo crontab -e
```

Add:
```
0 2 * * * /opt/backups/backup.sh >> /var/log/backup.log 2>&1
```

---

## 📊 Step 13: Import Grafana Dashboards

### Access Grafana:
1. Navigate to http://<SERVER_IP>:3000
2. Login: admin / your_secure_password
3. Go to Configuration → Data Sources → Add Prometheus
4. URL: http://prometheus:9090
5. Save & Test

### Import recommended dashboards:
- Node Exporter Full (ID: 1860)
- Docker Container & Host Metrics (ID: 10619)
- HAProxy 2 Full (ID: 12693)

---

## 🎯 Next Steps

1. ✅ Configure DNS records to point to Virtual IP
2. ✅ Setup email delivery (SMTP)
3. ✅ Configure Nextcloud apps (Calendar, Contacts, Talk)
4. ✅ Setup Keycloak realms and clients
5. ✅ Configure Odoo modules
6. ✅ Setup Mautic email campaigns
7. ✅ Integrate WHMCS/Blesta for billing
8. ✅ Perform load testing
9. ✅ Security audit and penetration testing
10. ✅ Create user documentation

---

## 🆘 Troubleshooting

### Docker Swarm issues:
```bash
# Check node status
docker node ls

# View service logs
docker service logs <service_name>

# Restart a service
docker service update --force <service_name>
```

### HAProxy not working:
```bash
# Check HAProxy logs
sudo journalctl -u haproxy -f

# Test configuration
haproxy -c -f /opt/haproxy/haproxy.cfg

# Restart HAProxy
sudo systemctl restart haproxy
```

### Database connection issues:
```bash
# Check PostgreSQL status
sudo systemctl status postgresql

# Check connections
sudo -u postgres psql -c "SELECT * FROM pg_stat_activity;"

# View logs
sudo tail -f /var/log/postgresql/postgresql-14-main.log
```

---

**Setup Guide Version**: 1.0  
**Last Updated**: December 11, 2025  
**Estimated Setup Time**: 1-2 days for experienced sysadmin
