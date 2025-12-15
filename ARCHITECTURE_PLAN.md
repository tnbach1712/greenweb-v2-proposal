# European Cloud Platform - Architecture & Implementation Plan

## 📋 Executive Summary

**Project Goal**: Build a European sovereign cloud platform offering open-source alternatives to Microsoft 365 and US tech solutions, focusing on data sovereignty, eco-friendly practices, and ethical data management.

**Target Market**: European companies and citizens seeking:
- Data sovereignty and GDPR compliance
- Eco-friendly and ethical technology
- Independence from US tech giants

**Business Model**: SaaS platform with self-service subscription (#1 priority)

---

## 🎯 Phase 1: MVP Launch (3-4 months)

### Core Services to Deploy
1. **Nextcloud Hub** - Collaborative workspace (M365 alternative)
   - File storage & sharing
   - Office documents (Collabora Online)
   - Calendar, Contacts, Mail
   - Video conferencing (Nextcloud Talk)
   
2. **Keycloak** - Identity & Access Management (IAM)
   - Single Sign-On (SSO)
   - Multi-tenant user management
   - LDAP/Active Directory integration
   
3. **Odoo** - ERP/CRM Suite
   - CRM, Sales, Project Management
   - Invoicing & Accounting
   - HR Management
   
4. **Mautic** - Marketing Automation
   - Email campaigns
   - Lead tracking
   - Marketing analytics

### Billing & Provisioning
- **WHMCS** (current)
- Auto-provisioning via APIs
- Multi-tier subscription plans

---

## 🏗️ High-Availability Architecture (99.9% Uptime)

### Architecture Overview - Hybrid Infrastructure (K3s + External Services)

```
                                    ┌─────────────────┐
                                    │   DNS + CDN     │
                                    │  (Cloudflare)   │
                                    └────────┬────────┘
                                             │
                    ┌────────────────────────┴────────────────────────┐
                    │        MetalLB LoadBalancer (VIP Pool)          │
                    └────────────────────────┬────────────────────────┘
                                             │
                    ┌────────────────────────┴────────────────────────┐
                    │     Traefik Ingress Controller (HA - 3 pods)    │
                    │     • Automatic TLS with cert-manager           │
                    │     • Health checks & auto-routing              │
                    └────────────────────────┬────────────────────────┘
                                             │
         ┌───────────────────────────────────┼───────────────────────────────────┐
         │                                   │                                   │
┌────────▼────────┐              ┌──────────▼──────────┐              ┌────────▼────────┐
│  K3s Server 1   │◄────etcd────►│   K3s Server 2      │◄────etcd────►│  K3s Server 3   │
│  (Control       │              │   (Control Plane    │              │  (Control       │
│   Plane +       │              │    + Workloads)     │              │   Plane +       │
│   Workloads)    │              │                     │              │   Workloads)    │
└────────┬────────┘              └──────────┬──────────┘              └────────┬────────┘
         │                                  │                                  │
         └──────────────────────────────────┼──────────────────────────────────┘
                                            │
         ┌──────────────────────────────────┴──────────────────────────────────┐
         │               K3s Workloads (Application Layer Only)                │
         │                                                                     │
         │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐   │
         │  │ Nextcloud   │  │   Odoo      │  │  Keycloak   │  │  Mautic   │   │
         │  │ (3 replicas)│  │ (2 replicas)│  │ (3 replicas)│  │(2 replicas)│  │
         │  └─────────────┘  └─────────────┘  └─────────────┘  └───────────┘   │
         │                                                                     │
         │  • Auto-scaling with HPA                                            │
         │  • Rolling updates / Blue-Green deployments                         │
         │  • Pod anti-affinity (spread across nodes)                          │
         │  • StatefulSets removed (using external DB/Redis)                   │
         └──────────────────────────────────┬──────────────────────────────────┘
                                            │
                                            │ Connects to External Services ↓
         ┌──────────────────────────────────┴──────────────────────────────────┐
         │                                  │                                   │
         │                                  │                                   │
┌────────▼────────────┐         ┌──────────▼──────────┐         ┌─────────▼──────────┐
│  PostgreSQL Cluster │         │  Redis Sentinel     │         │  Monitoring Stack  │
│  (External VMs)     │         │  (External VMs)     │         │  (External VM)     │
│                     │         │                     │         │                    │
│  • Primary (VM1)    │         │  • Master (VM1)     │         │  • Prometheus      │
│  • Standby (VM2)    │         │  • Replica (VM2)    │         │  • Grafana         │
│  • Streaming Repl.  │         │  • Sentinel (VM3)   │         │  • AlertManager    │
│  • Automatic        │         │  • Auto-failover    │         │  • Node Exporters  │
│    Failover         │         │  • Persistent       │         │  • Loki (logs)     │
│  • pgBackRest       │         │                     │         │                    │
└─────────────────────┘         └─────────────────────┘         └────────────────────┘
         │                               │                               │
         │                               │                               │
┌────────▼────────────────────────────────▼───────────────────────────────▼──────────┐
│                         S3 Object Storage                                           │
│                    (PlanetHoster Object Storage)                                    │
│                                                                                     │
│  • Nextcloud files                                                                 │
│  • Database backups (pgBackRest)                                                   │
│  • Application backups                                                             │
│  • Logs archive                                                                    │
│  • Monitoring data archive                                                         │
│  • Versioning enabled                                                              │
│  • Lifecycle policies (auto-delete old backups)                                    │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**Key Components:**

**K3s Cluster (Application Layer Only):**
- **K3s 3-Server Cluster**: Control plane + application workloads only
- **Traefik Ingress**: Automatic routing, TLS, health checks
- **MetalLB**: LoadBalancer IPs
- **cert-manager**: Automatic SSL certificates
- **No StatefulSets**: All stateful services external

**External Database Layer (Dedicated VMs):**
- **PostgreSQL Cluster**: 
  - VM1: Primary (active)
  - VM2: Standby (hot standby with streaming replication)
  - Automatic failover (Patroni or repmgr)
  - Connection pooling (PgBouncer)
  - Backup to S3 with pgBackRest

**External Cache Layer (Dedicated VMs):**
- **Redis Sentinel**:
  - VM1: Redis master + Sentinel
  - VM2: Redis replica + Sentinel  
  - VM3: Sentinel only (quorum)
  - Automatic failover
  - Persistent AOF + RDB

**External Monitoring (Dedicated VM):**
- **Prometheus**: Metrics collection from K3s and VMs
- **Grafana**: Dashboards and visualization
- **AlertManager**: Alert routing
- **Loki**: Log aggregation
- **Node Exporter**: System metrics from all nodes

**External Object Storage:**
- **S3-Compatible Storage**: PlanetHoster Object Storage or MinIO
- **Uses**: Nextcloud files, backups, logs, archives
- **Features**: Versioning, lifecycle policies, encryption

---

## 🛠️ Technology Stack Recommendations

### Load Balancing & Failover
- **Traefik** (K3s built-in Ingress Controller)
- **MetalLB** for LoadBalancer service type
- **Keepalived** (optional) for Virtual IP failover
- Automatic TLS with cert-manager
- Health checks and automatic pod restarts

### Application Layer
| Service | Instances | Deployment | Purpose |
|---------|-----------|------------|---------|
| **Nextcloud** | 2-3 (K3s pods) | K3s Deployment | Collaborative workspace |
| **Odoo** | 2-3 (K3s pods) | K3s Deployment | ERP/CRM |
| **Keycloak** | 2-3 (K3s pods) | K3s Deployment | SSO/IAM |
| **Mautic** | 2 (K3s pods) | K3s Deployment | Marketing automation |
| **WHMCS/Blesta** | 2 (K3s pods) | K3s Deployment | Billing & provisioning |

**Note:** All applications deployed as K3s Deployments (not StatefulSets) since state is externalized.

### Database Layer (External - Dedicated VMs)
- **PostgreSQL 15+** on dedicated VMs
  - **VM 1 (Primary)**: 4 vCPU, 8GB RAM, 100GB SSD
  - **VM 2 (Standby)**: 4 vCPU, 8GB RAM, 100GB SSD
  - **Setup**: Streaming replication (synchronous or async)
  - **Failover**: Automatic with Patroni + etcd
  - **Connection Pooling**: PgBouncer (deployed on each VM or separate)
  - **Backup**: pgBackRest to S3 (daily full, hourly incremental)
  - **Monitoring**: postgres_exporter → Prometheus

**Database Configuration:**
```yaml
# Shared databases on single PostgreSQL cluster
Databases:
  - nextcloud_db     # Nextcloud database
  - odoo_db          # Odoo database  
  - keycloak_db      # Keycloak database
  - mautic_db        # Mautic database
  - whmcs_db         # WHMCS database

# Or separate PostgreSQL instances per application if needed
```

### Cache & Session Management (External - Dedicated VMs)
- **Redis Sentinel** on dedicated VMs
  - **VM 1**: Redis master + Sentinel (2 vCPU, 4GB RAM)
  - **VM 2**: Redis replica + Sentinel (2 vCPU, 4GB RAM)
  - **VM 3**: Sentinel only (1 vCPU, 2GB RAM) - for quorum
  - **Persistence**: AOF (appendonly) + RDB snapshots
  - **Failover**: Automatic with Redis Sentinel
  - **Backup**: Daily RDB snapshots to S3
  - **Monitoring**: redis_exporter → Prometheus

**Redis Uses:**
- Session storage for all applications
- Nextcloud file locking and caching
- Odoo cache
- Rate limiting data

### Storage Layer (External S3)
- **PlanetHoster Object Storage** (S3-compatible)
  - Nextcloud files storage
  - Database backups (pgBackRest)
  - Application backups
  - Log archives
  - Monitoring data archives
- **Alternative**: Self-hosted MinIO on separate VMs
- **Features**:
  - Versioning enabled
  - Lifecycle policies (auto-delete backups > 90 days)
  - Server-side encryption
  - Cross-region replication (optional)

### Container Orchestration
- **K3s** (Lightweight Kubernetes)
  - Production-ready, lightweight K8s distribution
  - Built-in Traefik Ingress Controller
  - Automatic TLS with cert-manager
  - Easy multi-node clustering
  - 100% K8s API compatible

### Monitoring & Observability (External - Dedicated VM)
- **Monitoring Server**: 4 vCPU, 8GB RAM, 100GB SSD
  - **Prometheus**: Metrics collection and storage
    - Scrape K3s cluster metrics (kubelet, kube-state-metrics, node-exporter)
    - Scrape external PostgreSQL (postgres_exporter on DB VMs)
    - Scrape external Redis (redis_exporter on Redis VMs)
    - Scrape application metrics from K3s pods
    - Retention: 30 days local, long-term storage to S3
  - **Grafana**: Visualization and dashboards
    - Pre-built dashboards for K3s, PostgreSQL, Redis
    - Custom application dashboards
    - Alerting rules and notifications
  - **AlertManager**: Alert routing and notifications
    - Email, Slack, PagerDuty integrations
    - Alert grouping and deduplication
  - **Loki** (optional): Centralized log aggregation
    - Application logs from K3s pods
    - System logs from K3s nodes and external VMs
    - Database logs from PostgreSQL VMs
    - Log retention and querying
  - **Uptime Kuma**: Simple uptime monitoring for external services

**Why External Monitoring?**
- Monitor K3s cluster health independently
- Survives K3s cluster failure or upgrades
- Lower resource contention on K3s nodes
- Easier to access during K3s downtime

### Security
- **Fail2ban** - Intrusion prevention
- **CrowdSec** - Collaborative security
- **Let's Encrypt** - SSL/TLS certificates
- **Keycloak** - Centralized authentication
- **Nginx ModSecurity** - Web Application Firewall (WAF)

### Backup & Disaster Recovery
- **Restic** or **Borg** - Encrypted backups
- **pgBackRest** - PostgreSQL backups
- 3-2-1 backup strategy:
  - 3 copies of data
  - 2 different storage types
  - 1 off-site backup
- Daily incremental, weekly full backups
- RPO: 4 hours, RTO: 1 hour

---

## 🚀 Deployment Strategy: Blue-Green with K3s

### What is Blue-Green Deployment?

Blue-Green deployment is a release strategy that maintains two identical production environments:
- **Blue** = Current production
- **Green** = New version

### Benefits with K3s/Kubernetes
- **Zero downtime updates** - Seamless traffic switching
- **Instant rollback capability** - Switch back in seconds
- **Safe testing in production** - Test green before switching traffic
- **Native K8s support** - Built into Kubernetes deployments
- **Automated health checks** - Only switch when green is healthy

### Implementation Methods

#### Method 1: Native Kubernetes Rolling Updates (Easiest)

Kubernetes automatically performs rolling updates with zero downtime:

```bash
# Update to new version
kubectl set image deployment/nextcloud nextcloud=nextcloud:29 -n nextcloud

# Kubernetes automatically:
# 1. Creates new pods with new version
# 2. Waits for readiness probes to pass
# 3. Routes traffic to new pods
# 4. Terminates old pods gradually
# 5. Ensures minimum replica count always running

# Monitor rollout
kubectl rollout status deployment/nextcloud -n nextcloud

# Instant rollback if issues detected
kubectl rollout undo deployment/nextcloud -n nextcloud
```

**Visual Process:**
```
Step 1: Current state (v1.0)
[Pod v1.0] [Pod v1.0] [Pod v1.0] ← All traffic
     ↓
Step 2: Deploy new version (v1.1)
[Pod v1.0] [Pod v1.0] [Pod v1.0] [Pod v1.1] ← Health checking
     ↓
Step 3: New pod ready
[Pod v1.0] [Pod v1.0] [Pod v1.0] [Pod v1.1] ← Traffic starts routing
     ↓
Step 4: Rolling update continues
[Pod v1.0] [Pod v1.0] [Pod v1.1] [Pod v1.1] ← Traffic gradually shifts
     ↓
Step 5: Complete
[Pod v1.1] [Pod v1.1] [Pod v1.1] ← All traffic to new version
```

#### Method 2: Manual Blue-Green (Full Control)

For more control over the switching process:

```yaml
# Blue deployment (current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud-blue
  namespace: nextcloud
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nextcloud
      version: blue
  template:
    metadata:
      labels:
        app: nextcloud
        version: blue
    spec:
      containers:
      - name: nextcloud
        image: nextcloud:28-apache
---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud-green
  namespace: nextcloud
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nextcloud
      version: green
  template:
    metadata:
      labels:
        app: nextcloud
        version: green
    spec:
      containers:
      - name: nextcloud
        image: nextcloud:29-apache
---
# Service selector (switch between blue/green)
apiVersion: v1
kind: Service
metadata:
  name: nextcloud
  namespace: nextcloud
spec:
  selector:
    app: nextcloud
    version: blue  # Change to 'green' to switch
  ports:
  - port: 80
    targetPort: 80
```

**Deployment Process:**
```bash
# 1. Deploy green environment
kubectl apply -f nextcloud-green.yaml

# 2. Test green internally
kubectl port-forward -n nextcloud deployment/nextcloud-green 8080:80
# Access at http://localhost:8080 and test

# 3. If tests pass, switch traffic to green
kubectl patch service nextcloud -n nextcloud \
  -p '{"spec":{"selector":{"version":"green"}}}'

# Traffic instantly switches to green!

# 4. Monitor for issues
kubectl logs -n nextcloud -l version=green --tail=100 -f

# 5. If issues, instant rollback to blue
kubectl patch service nextcloud -n nextcloud \
  -p '{"spec":{"selector":{"version":"blue"}}}'

# 6. If successful, cleanup blue
kubectl delete deployment nextcloud-blue -n nextcloud
```

#### Method 3: Argo Rollouts (Advanced - Canary + Blue-Green)

For progressive delivery with canary releases:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: nextcloud
  namespace: nextcloud
spec:
  replicas: 3
  strategy:
    blueGreen:
      activeService: nextcloud-active
      previewService: nextcloud-preview
      autoPromotionEnabled: false  # Manual approval
      scaleDownDelaySeconds: 300   # Keep old version for 5 min
  template:
    spec:
      containers:
      - name: nextcloud
        image: nextcloud:28-apache
```

```bash
# Install Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Deploy new version
kubectl argo rollouts set image nextcloud nextcloud=nextcloud:29 -n nextcloud

# Preview at preview URL before promoting
curl https://nextcloud-preview.yourplatform.eu

# Promote to production
kubectl argo rollouts promote nextcloud -n nextcloud

# Or abort if issues
kubectl argo rollouts abort nextcloud -n nextcloud
```

### Comparison of Methods

| Feature | Rolling Update | Manual Blue-Green | Argo Rollouts |
|---------|---------------|-------------------|---------------|
| **Complexity** | Simple | Medium | Advanced |
| **Control** | Automatic | Full manual | Configurable |
| **Rollback Speed** | Fast (~30s) | Instant (<5s) | Instant (<5s) |
| **Testing** | Limited | Full preview | Full preview + canary |
| **Setup Time** | 0 (built-in) | 5 minutes | 15 minutes |
| **Recommended For** | Most deployments | Critical services | Large teams |

### CI/CD Pipeline Integration

```yaml
# GitLab CI/CD example
stages:
  - build
  - test
  - deploy-staging
  - deploy-production

build:
  stage: build
  script:
    - docker build -t registry.yourplatform.eu/nextcloud:$CI_COMMIT_SHA .
    - docker push registry.yourplatform.eu/nextcloud:$CI_COMMIT_SHA

deploy-production:
  stage: deploy-production
  script:
    # Update deployment with new image
    - kubectl set image deployment/nextcloud \
        nextcloud=registry.yourplatform.eu/nextcloud:$CI_COMMIT_SHA \
        -n nextcloud
    
    # Wait for rollout to complete
    - kubectl rollout status deployment/nextcloud -n nextcloud
    
    # Run smoke tests
    - ./smoke-tests.sh
  only:
    - main
  when: manual  # Require manual approval
```

### Health Checks Configuration

Critical for automatic rollback:

```yaml
spec:
  containers:
  - name: nextcloud
    image: nextcloud:29-apache
    # Liveness probe - restart if unhealthy
    livenessProbe:
      httpGet:
        path: /status.php
        port: 80
      initialDelaySeconds: 60
      periodSeconds: 30
      timeoutSeconds: 5
      failureThreshold: 3
    
    # Readiness probe - remove from load balancer if not ready
    readinessProbe:
      httpGet:
        path: /status.php
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 3
      failureThreshold: 3
```

### Rollback Strategy

**Automatic Rollback Triggers:**
- Readiness probe fails for 3+ attempts
- Liveness probe fails for 3+ attempts
- Pod crash loop detected

**Manual Rollback:**
```bash
# View rollout history
kubectl rollout history deployment/nextcloud -n nextcloud

# Rollback to previous version
kubectl rollout undo deployment/nextcloud -n nextcloud

# Rollback to specific revision
kubectl rollout undo deployment/nextcloud -n nextcloud --to-revision=3
```

---

## 📦 PlanetHoster Infrastructure Setup

### Recommended Configuration with External Services

#### Option 1: MVP Setup (Single PostgreSQL, Basic Redis)
**Total: €200-250/month**

**K3s Cluster (3 servers):**
- **3x K3s Server Nodes** (4 vCPU, 8 GB RAM each) - €35-40/server/month
  - All nodes run control plane + application workloads
  - No taints, all-in-one mode
  - Total: €105-120/month

**External Database:**
- **1x PostgreSQL VM** (4 vCPU, 8 GB RAM, 100GB SSD) - €35-40/month
  - Single instance for MVP
  - Manual backups to S3
  - Upgrade to HA later

**External Cache:**
- **1x Redis VM** (2 vCPU, 4 GB RAM, 50GB SSD) - €20-25/month
  - Single instance with persistence
  - AOF + RDB snapshots

**External Monitoring:**
- **1x Monitoring VM** (2 vCPU, 4 GB RAM, 50GB SSD) - €20-25/month
  - Prometheus + Grafana + AlertManager
  - Optional Loki for logs

**Object Storage:**
- **PlanetHoster Object Storage** (S3-compatible) - €15-25/month
  - 500GB-1TB storage included
  - Pay-per-GB for overage (~€0.01/GB)
  - For Nextcloud files, backups, archives

**Breakdown:**
- K3s cluster (3 nodes): €105-120/month
- PostgreSQL VM: €35-40/month
- Redis VM: €20-25/month  
- Monitoring VM: €20-25/month
- Object Storage: €15-25/month
- **Total: €195-235/month**

---

#### Option 2: High-Availability Setup (Production)
**Total: €400-500/month**

**K3s Cluster (5 servers for higher availability):**
- **5x K3s Server Nodes** (4 vCPU, 8 GB RAM each) - €35-40/server/month
  - Quorum of 3 for etcd, 5 for better availability
  - Total: €175-200/month

**External Database (HA):**
- **2x PostgreSQL VMs** (4 vCPU, 8 GB RAM, 100GB SSD) - €35-40/server/month
  - VM1: Primary database server
  - VM2: Hot standby with streaming replication
  - Automatic failover with Patroni + etcd
  - Total: €70-80/month

**External Cache (HA):**
- **3x Redis VMs**:
  - **2x Redis + Sentinel** (2 vCPU, 4 GB RAM each) - €20-25/server/month
  - **1x Sentinel only** (1 vCPU, 2 GB RAM) - €10-15/month
  - Automatic failover with Redis Sentinel
  - Total: €50-65/month

**External Monitoring:**
- **1x Monitoring VM** (4 vCPU, 8 GB RAM, 100GB SSD) - €35-40/month
  - Prometheus + Grafana + AlertManager + Loki
  - Larger retention period (30 days)

**Object Storage:**
- **PlanetHoster Object Storage** (S3-compatible) - €30-50/month
  - 2TB storage with versioning
  - Cross-region replication enabled
  - Pay-per-GB for overage (~€0.01/GB)

**Optional - Dedicated Backup VM:**
- **1x Backup VM** (2 vCPU, 4 GB RAM, 200GB SSD) - €25-30/month
  - Dedicated backup orchestration (pgBackRest, restic)
  - Backup verification and restoration testing

**Breakdown:**
- K3s cluster (5 nodes): €175-200/month
- PostgreSQL cluster (2 VMs): €70-80/month
- Redis cluster (3 VMs): €50-65/month
- Monitoring VM: €35-40/month
- Object Storage: €30-50/month
- Optional Backup VM: €25-30/month
- **Total: €360-435/month (€385-465 with backup VM)**

---

#### Option 3: Enterprise Setup (Multi-Region)
**Total: €800-1000/month**

**Primary Region (K3s + External Services):**
- 5x K3s nodes: €175-200/month
- 2x PostgreSQL VMs: €70-80/month
- 3x Redis VMs: €50-65/month
- 1x Monitoring VM: €35-40/month
- Object Storage: €50-75/month
- **Subtotal: €380-460/month**

**Secondary Region (DR Standby):**
- 3x K3s nodes (standby): €105-120/month
- 1x PostgreSQL VM (replica): €35-40/month
- Object Storage replication: €50-75/month
- **Subtotal: €190-235/month**

**Additional Costs:**
- Cross-region bandwidth: €50-100/month
- Advanced monitoring/logging: €30-50/month
- SSL certificates (if not Let's Encrypt): €0-20/month

**Total: €650-845/month**

**Estimated Monthly Cost**: €400-600

---

## 🔄 CI/CD Pipeline

### Tools
- **GitLab CI/CD** or **GitHub Actions**
- **Watchtower** - Automatic container updates (optional)
- **Ansible** - Configuration management

### Pipeline Stages

```yaml
stages:
  - build
  - test
  - deploy-staging
  - integration-tests
  - deploy-production-green
  - health-check
  - switch-traffic
  - cleanup

# Example: .gitlab-ci.yml
build:
  stage: build
  script:
    - docker build -t nextcloud:$CI_COMMIT_SHA .
    - docker push registry.yourplatform.eu/nextcloud:$CI_COMMIT_SHA

test:
  stage: test
  script:
    - docker run nextcloud:$CI_COMMIT_SHA npm test

deploy-staging:
  stage: deploy-staging
  script:
    - docker stack deploy -c docker-compose-staging.yml staging

deploy-green:
  stage: deploy-production-green
  script:
    - docker stack deploy -c docker-compose-green.yml production-green
  when: manual  # Require manual approval

switch-traffic:
  stage: switch-traffic
  script:
    - ansible-playbook switch-to-green.yml
  when: manual  # Require manual approval
```

---

## 🔐 Security Best Practices

### Network Security
- **Firewall**: Only expose ports 80, 443, 22 (SSH)
- **VPN**: WireGuard for admin access
- **Fail2ban**: Ban IPs after failed login attempts
- **DDoS Protection**: Cloudflare or PlanetHoster's built-in

### Application Security
- **WAF**: ModSecurity with OWASP Core Rule Set
- **SSL/TLS**: A+ rating on SSL Labs
- **Security Headers**: HSTS, CSP, X-Frame-Options
- **Rate Limiting**: Nginx/HAProxy rate limits

### Data Security
- **Encryption at Rest**: LUKS for disk encryption
- **Encryption in Transit**: TLS 1.3 only
- **Backup Encryption**: Restic with strong passphrase
- **Secret Management**: Vault or Docker Secrets

### Compliance
- **GDPR**: Data residency in Europe
- **ISO 27001**: Follow security controls
- **Regular Audits**: Quarterly security assessments
- **Penetration Testing**: Annual pen tests

---

## 📊 Monitoring & Alerting

### Key Metrics to Monitor

#### Infrastructure
- CPU, RAM, Disk usage
- Network bandwidth
- Disk I/O

#### Application
- Response time (p50, p95, p99)
- Error rate
- Request rate
- Active users

#### Database
- Connection pool usage
- Query performance
- Replication lag
- Disk space

### Alerting Rules

```yaml
# Prometheus alerting rules
groups:
- name: critical
  rules:
  - alert: ServiceDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Service {{ $labels.instance }} is down"

  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High error rate on {{ $labels.service }}"

  - alert: DatabaseReplicationLag
    expr: pg_replication_lag > 30
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Database replication lag > 30s"
```

---

## 📅 Implementation Timeline

### Month 1: Infrastructure Setup
- Week 1-2: Provision PlanetHoster servers
- Week 2-3: Setup load balancers, networking, security
- Week 3-4: Deploy monitoring stack (Prometheus, Grafana)

### Month 2: Core Services Deployment
- Week 1: Deploy Keycloak (SSO/IAM)
- Week 2: Deploy Nextcloud + Collabora
- Week 3: Deploy Odoo
- Week 4: Deploy Mautic

### Month 3: Integration & Testing
- Week 1-2: Integrate all services with Keycloak SSO
- Week 2-3: Setup WHMCS/Blesta auto-provisioning
- Week 3-4: Load testing, security testing

### Month 4: Beta Launch
- Week 1-2: Beta testing with 10-20 pilot customers
- Week 2-3: Bug fixes and optimization
- Week 3-4: Public launch preparation

---

## 💰 Cost Estimation

### Infrastructure (Monthly)
| Item | Cost (EUR) |
|------|------------|
| PlanetHoster Servers (MVP) | €160-200 |
| Object Storage (100 GB) | €1 |
| Backup Storage (200 GB) | €2 |
| Domain & SSL | €5 |
| **Total Infrastructure** | **€168-208** |

### Software Licenses (All Open-Source)
- Nextcloud: Free (Enterprise support optional: €1,900/year)
- Odoo: Free (Enterprise: €20/user/month)
- Keycloak: Free
- Mautic: Free
- **Total Software**: **€0 (MVP)**

### Development & Operations
- Developer time: 3-4 months
- Ongoing maintenance: 20-40 hours/month

---

## 🎯 Success Metrics

### Technical KPIs
- **Uptime**: ≥ 99.9% (target)
- **Response Time**: < 200ms (p95)
- **Recovery Time**: < 1 hour (RTO)
- **Data Loss**: < 4 hours (RPO)

### Business KPIs
- **Customer Acquisition**: 50 customers in 6 months
- **Monthly Recurring Revenue**: €5,000 by month 6
- **Churn Rate**: < 5%
- **Customer Satisfaction**: > 4.5/5

---

## 🚨 Risk Mitigation

### Technical Risks
| Risk | Mitigation |
|------|------------|
| Service downtime | HA architecture, automatic failover |
| Data loss | Daily backups, replication, PITR |
| Security breach | WAF, monitoring, regular audits |
| Vendor lock-in | Open-source stack, portable |

### Business Risks
| Risk | Mitigation |
|------|------------|
| Low adoption | Marketing, free trial, pilot customers |
| Competition | Focus on European sovereignty USP |
| Scaling issues | Start with proven infrastructure |
| Partnership failure | Diversify providers, own infrastructure |

---

## 🔧 Next Steps

### Immediate Actions (Week 1-2)
1. ✅ Finalize partnership agreement with PlanetHoster
2. ✅ Purchase domain name (e.g., sovereigncloud.eu)
3. ✅ Provision initial servers (2x cloud servers + 1x DB)
4. ✅ Setup Git repository and project management
5. ✅ Create detailed technical specifications

### Short-term (Month 1)
1. Setup CI/CD pipeline
2. Deploy monitoring stack
3. Configure load balancers and networking
4. Setup backup and disaster recovery

### Medium-term (Month 2-3)
1. Deploy and configure all core services
2. Integrate SSO across all services
3. Setup billing and auto-provisioning
4. Security hardening and compliance

### Long-term (Month 4+)
1. Beta launch with pilot customers
2. Gather feedback and iterate
3. Public launch marketing campaign
4. Scale infrastructure based on demand

---

## 📚 Recommended Documentation

### Essential Reading
- [Nextcloud Administration Manual](https://docs.nextcloud.com/server/stable/admin_manual/)
- [Keycloak Documentation](https://www.keycloak.org/documentation)
- [Odoo Documentation](https://www.odoo.com/documentation/)
- [HAProxy Configuration Guide](http://docs.haproxy.org/)
- [PostgreSQL High Availability](https://www.postgresql.org/docs/current/high-availability.html)

### Best Practices
- [The Twelve-Factor App](https://12factor.net/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [GDPR Compliance Checklist](https://gdpr.eu/checklist/)

---

## 🤝 Partnership with PlanetHoster

### Proposed Collaboration Model
1. **Infrastructure Partnership**
   - Discounted rates for startup phase
   - Technical support and consultation
   - Co-marketing opportunities

2. **Revenue Sharing (Optional)**
   - PlanetHoster provides infrastructure credit
   - Your platform pays % of revenue or fixed monthly fee
   - Scales with customer growth

3. **Joint Development**
   - You: Platform development, customer acquisition
   - PlanetHoster: Infrastructure, reliability, support
   - Shared success metrics

---

## 📞 Contact & Support

For questions or collaboration inquiries:
- Project Repository: [To be created]
- Documentation: [To be created]
- Support Email: [To be defined]

---

**Document Version**: 1.0  
**Last Updated**: December 11, 2025  
**Author**: Architecture Team  
**Status**: Proposed Architecture for Review
