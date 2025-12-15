# K3s Kubernetes Setup Guide - European Cloud Platform

## 🎯 Why K3s?

- **Lightweight**: ~40MB binary vs 1GB+ for full Kubernetes
- **Production-ready**: Used by CNCF projects and enterprises
- **Easy to install**: Single command setup
- **Batteries included**: Traefik Ingress, Local storage provisioner
- **100% K8s compatible**: All standard K8s tools work
- **Perfect for edge/small clusters**: 2-10 nodes ideal

---

## 📋 Prerequisites

- 3+ Ubuntu 22.04 servers (PlanetHoster Hybrid Cloud)
- 2GB+ RAM per server (4GB+ recommended)
- Root or sudo access
- Open ports: 6443, 443, 80, 10250

## 🏗️ K3s Architecture Options

### Option 1: All-in-One Server Nodes (Recommended for 3-5 nodes) ✅

**What we use in this guide:**
- All 3 nodes are **server nodes** (K3s terminology for masters)
- Each node runs both:
  - **Control plane** (API server, scheduler, controller)
  - **Workloads** (your applications)
- Best for small-medium clusters (3-10 nodes)
- More efficient resource usage

```
┌─────────────────────────────────────────┐
│  Node 1: Server (Master + Worker)      │
│  • Control Plane ✓                     │
│  • Workloads ✓                         │
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│  Node 2: Server (Master + Worker)      │
│  • Control Plane ✓                     │
│  • Workloads ✓                         │
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│  Node 3: Server (Master + Worker)      │
│  • Control Plane ✓                     │
│  • Workloads ✓                         │
└─────────────────────────────────────────┘
```

**Why this is better for your use case:**
- ✅ Uses all 3 nodes for running applications
- ✅ No wasted resources on dedicated masters
- ✅ Simpler setup and management
- ✅ Still highly available (3 control planes)
- ✅ Perfect for 3-10 nodes

### Option 2: Dedicated Masters + Workers (Traditional K8s style)

**When to use:**
- Large clusters (10+ nodes)
- Need dedicated control plane resources
- Have 5+ servers available

```
┌─────────────────────────────────────────┐
│  Node 1-3: Server Only (Masters)       │
│  • Control Plane ✓                     │
│  • Workloads ✗ (tainted)               │
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│  Node 4-N: Agent (Workers)             │
│  • Control Plane ✗                     │
│  • Workloads ✓                         │
└─────────────────────────────────────────┘
```

**To setup dedicated masters (if needed):**
```bash
# On master nodes, taint them to not run workloads
kubectl taint nodes server1 node-role.kubernetes.io/master=true:NoSchedule
kubectl taint nodes server2 node-role.kubernetes.io/master=true:NoSchedule
kubectl taint nodes server3 node-role.kubernetes.io/master=true:NoSchedule

# Then join agent nodes (workers only)
curl -sfL https://get.k3s.io | K3S_URL=https://server1:6443 K3S_TOKEN=xxx sh -
```

### Comparison

| Aspect | All-in-One Servers | Dedicated Masters + Workers |
|--------|-------------------|----------------------------|
| **Min. nodes** | 1 (3 for HA) | 4 (3 masters + 1 worker) |
| **Resource efficiency** | ✅ Excellent | ⚠️ Masters idle |
| **Setup complexity** | ✅ Simple | ⚠️ More complex |
| **Best for** | 3-10 nodes | 10+ nodes |
| **HA** | ✅ Yes | ✅ Yes |
| **Cost** | ✅ Lower | ⚠️ Higher |

**For your project:** Use **Option 1 (All-in-One Servers)** as described in this guide.
- SSH key authentication configured

---

## 🏗️ K3s Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         DNS + CDN                                │
│                      (Cloudflare)                                │
└────────────────────────────┬────────────────────────────────────┘
                             │
                ┌────────────▼────────────┐
                │   MetalLB LoadBalancer  │
                │    (Virtual IP Pool)    │
                └────────────┬────────────┘
                             │
         ┌───────────────────┴───────────────────┐
         │      Traefik Ingress Controller       │
         │   (Built-in with K3s, HA mode)        │
         └───────────────────┬───────────────────┘
                             │
    ┌────────────────────────┼────────────────────────┐
    │                        │                        │
┌───▼────────┐       ┌──────▼──────┐       ┌────────▼───┐
│  K3s       │       │  K3s        │       │  K3s       │
│  Master 1  │◄─────►│  Master 2   │◄─────►│  Master 3  │
│  (Server)  │       │  (Server)   │       │  (Server)  │
└───┬────────┘       └──────┬──────┘       └────────┬───┘
    │                       │                        │
    └───────────────────────┼────────────────────────┘
                            │
         ┌──────────────────┴──────────────────┐
         │                                     │
┌────────▼────────┐               ┌───────────▼─────────┐
│  Workload Pods  │               │   Workload Pods     │
│                 │               │                     │
│ • Nextcloud     │               │ • Nextcloud         │
│ • Odoo          │               │ • Odoo              │
│ • Keycloak      │               │ • Keycloak          │
│ • Mautic        │               │ • Mautic            │
└────────┬────────┘               └───────────┬─────────┘
         │                                    │
         └────────────────┬───────────────────┘
                          │
         ┌────────────────┴────────────────┐
         │                                 │
┌────────▼────────┐            ┌──────────▼──────────┐
│  PostgreSQL     │            │  Redis Cluster      │
│  StatefulSet    │            │  (Session Cache)    │
│  (HA with       │            │                     │
│   Patroni)      │            │                     │
└─────────────────┘            └─────────────────────┘
         │
┌────────▼────────┐
│  MinIO/Longhorn │
│  (Persistent    │
│   Storage)      │
└─────────────────┘
```

---

## 🚀 Part 1: K3s Cluster Setup

### Step 1: Install K3s on Master Node 1 (Server 1)

```bash
# On app-server-01 (First server node)
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --write-kubeconfig-mode=644 \
  --tls-san=$(hostname -I | awk '{print $1}')

# Note: We do NOT disable servicelb and traefik initially
# K3s will install them by default, which is what we want

# Wait for K3s to be ready
sudo systemctl status k3s

# Get the join token (save this!)
sudo cat /var/lib/rancher/k3s/server/node-token

# Get kubeconfig
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Verify - should show 1 node with roles "control-plane,master"
kubectl get nodes
```

**What happens:**
- Node becomes a **server node** (runs control plane + workloads)
- `--cluster-init` enables embedded etcd for HA
- This node can run both control plane and application pods
- Role shows as "control-plane,master" but it's NOT tainted

**Expected output:**
```
NAME           STATUS   ROLES                       AGE   VERSION
app-server-01  Ready    control-plane,master        1m    v1.28.x+k3s1
```

### Step 2: Join Additional Server Nodes (HA Control Plane)

**Important:** These are also **server nodes** (not worker/agent nodes). They will run both control plane and workloads.

```bash
# On app-server-02 (Second server node)
export K3S_TOKEN="<token-from-step-1>"
export MASTER1_IP="<app-server-01-ip>"

curl -sfL https://get.k3s.io | sh -s - server \
  --server https://${MASTER1_IP}:6443 \
  --token ${K3S_TOKEN} \
  --write-kubeconfig-mode=644 \
  --tls-san=$(hostname -I | awk '{print $1}')

# This joins as a SERVER node (master + worker), not just an agent
```

```bash
# On app-server-03 (Third server node - optional but recommended for HA)
export K3S_TOKEN="<token-from-step-1>"
export MASTER1_IP="<app-server-01-ip>"

curl -sfL https://get.k3s.io | sh -s - server \
  --server https://${MASTER1_IP}:6443 \
  --token ${K3S_TOKEN} \
  --write-kubeconfig-mode=644 \
  --tls-san=$(hostname -I | awk '{print $1}')
```

**What happens:**
- Each node joins with `server` role (not `agent`)
- Each node runs its own control plane components
- Each node joins the etcd cluster (distributed consensus)
- Each node can run application workloads
- All nodes can accept kubectl commands

**If you only want worker nodes (not recommended for 3 nodes):**
```bash
# To join as agent/worker only (no control plane):
curl -sfL https://get.k3s.io | K3S_URL=https://${MASTER1_IP}:6443 K3S_TOKEN=${K3S_TOKEN} sh -
# Note: This is NOT what we want for 3 nodes!
```

### Step 3: Verify Cluster

```bash
# On any server node
kubectl get nodes

# Should show all nodes as Ready with roles "control-plane,master"
# NAME           STATUS   ROLES                       AGE   VERSION
# app-server-01  Ready    control-plane,master        5m    v1.28.x+k3s1
# app-server-02  Ready    control-plane,master        3m    v1.28.x+k3s1
# app-server-03  Ready    control-plane,master        1m    v1.28.x+k3s1

# Check if nodes can run workloads (should NOT show NoSchedule taint)
kubectl describe nodes | grep -i taint
# Expected: <none> or node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule

# Verify etcd cluster
sudo k3s kubectl get endpoints -n kube-system k3s
# Should show 3 endpoints (one per server)

# Test scheduling a pod on each node
kubectl run test-nginx --image=nginx --replicas=3
kubectl get pods -o wide
# Should see pods distributed across all 3 nodes
kubectl delete deployment test-nginx
```

**Understanding the output:**

1. **Roles: "control-plane,master"** - This means:
   - Node runs control plane components (API server, scheduler, controller-manager)
   - Node is part of the etcd cluster
   - Node CAN run workload pods (not tainted by default)

2. **Taint: `<none>`** - This means:
   - Pods can be scheduled on this node
   - This is DIFFERENT from traditional K8s where masters are tainted
   - This is what makes K3s efficient for small clusters

3. **All nodes show same role** - This is correct!
   - In K3s with `--cluster-init`, all server nodes are equal
   - Any node can handle API requests
   - Any node can run workloads

**Key Difference from Traditional Kubernetes:**

```
Traditional K8s (e.g., kubeadm):
┌─────────────────────────────────────┐
│  Master nodes (tainted)             │
│  • Control Plane ✓                  │
│  • Workloads ✗ (NoSchedule)         │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│  Worker nodes                       │
│  • Control Plane ✗                  │
│  • Workloads ✓                      │
└─────────────────────────────────────┘

K3s (default behavior):
┌─────────────────────────────────────┐
│  Server nodes (NOT tainted)         │
│  • Control Plane ✓                  │
│  • Workloads ✓                      │
└─────────────────────────────────────┘
```

**If you want traditional K8s behavior (not recommended for 3 nodes):**
```bash
# Taint masters to not run workloads
kubectl taint nodes app-server-01 node-role.kubernetes.io/master=true:NoSchedule
kubectl taint nodes app-server-02 node-role.kubernetes.io/master=true:NoSchedule
kubectl taint nodes app-server-03 node-role.kubernetes.io/master=true:NoSchedule

# Then add agent/worker nodes
curl -sfL https://get.k3s.io | K3S_URL=https://server1:6443 K3S_TOKEN=xxx sh -
```

**For your 3-node setup:** Keep the default (all servers can run workloads) ✅

---

## 🌐 Part 2: Install MetalLB (LoadBalancer)

MetalLB provides LoadBalancer IPs for services in bare-metal/cloud environments.

### Step 1: Install MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

# Wait for MetalLB to be ready
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

### Step 2: Configure IP Address Pool

Create `metallb-config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.240-192.168.1.250  # Change to your available IP range
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250  # Change to your IP range from PlanetHoster
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
```

```bash
kubectl apply -f metallb-config.yaml
```

---

## 🔀 Part 3: Install Traefik Ingress Controller (HA)

### Step 1: Add Traefik Helm Repository

```bash
# Install Helm if not installed
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add Traefik repo
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

### Step 2: Create Traefik Configuration

Create `traefik-values.yaml`:

```yaml
# High Availability configuration
deployment:
  replicas: 3  # Run 3 instances across nodes

# Enable Dashboard
ingressRoute:
  dashboard:
    enabled: true

# Ports configuration
ports:
  web:
    port: 80
    exposedPort: 80
  websecure:
    port: 443
    exposedPort: 443
    tls:
      enabled: true

# Service configuration
service:
  type: LoadBalancer

# Resource limits
resources:
  requests:
    cpu: "100m"
    memory: "50Mi"
  limits:
    cpu: "300m"
    memory: "150Mi"

# Anti-affinity to spread pods across nodes
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app.kubernetes.io/name
            operator: In
            values:
            - traefik
        topologyKey: kubernetes.io/hostname

# Enable access logs
logs:
  access:
    enabled: true

# Enable metrics for Prometheus
metrics:
  prometheus:
    enabled: true
```

### Step 3: Install Traefik

```bash
# Create namespace
kubectl create namespace traefik

# Install Traefik
helm install traefik traefik/traefik \
  --namespace traefik \
  --values traefik-values.yaml

# Verify installation
kubectl get pods -n traefik
kubectl get svc -n traefik

# Get LoadBalancer IP
kubectl get svc -n traefik traefik
```

---

## 🔐 Part 4: Install Cert-Manager (Automatic SSL)

### Step 1: Install Cert-Manager

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml

# Verify installation
kubectl get pods -n cert-manager
```

### Step 2: Create Let's Encrypt ClusterIssuer

Create `letsencrypt-issuer.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@yourplatform.eu  # Change this!
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@yourplatform.eu  # Change this!
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: traefik
```

```bash
kubectl apply -f letsencrypt-issuer.yaml

# Verify
kubectl get clusterissuer
```

---

## 🗄️ Part 5: Deploy PostgreSQL with High Availability

### Option 1: Using Zalando Postgres Operator (Recommended)

```bash
# Install Postgres Operator
kubectl apply -k github.com/zalando/postgres-operator/manifests

# Create namespace
kubectl create namespace databases
```

Create `postgresql-cluster.yaml`:

```yaml
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: sovereign-postgres
  namespace: databases
spec:
  teamId: "sovereign"
  volume:
    size: 100Gi
    storageClass: local-path  # K3s default storage class
  numberOfInstances: 3
  users:
    nextcloud:
    - superuser
    - createdb
    odoo:
    - superuser
    - createdb
    keycloak:
    - superuser
    - createdb
    mautic:
    - superuser
    - createdb
  databases:
    nextcloud: nextcloud
    odoo: odoo
    keycloak: keycloak
    mautic: mautic
  postgresql:
    version: "15"
    parameters:
      max_connections: "200"
      shared_buffers: "1GB"
      effective_cache_size: "3GB"
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 2Gi
  patroni:
    initdb:
      encoding: "UTF8"
      locale: "en_US.UTF-8"
    pg_hba:
    - hostssl all all 0.0.0.0/0 md5
    - host    all all 0.0.0.0/0 md5
```

```bash
kubectl apply -f postgresql-cluster.yaml

# Wait for cluster to be ready
kubectl get postgresql -n databases

# Get connection details
kubectl get secret sovereign-postgres.nextcloud.credentials.postgresql.acid.zalan.do \
  -n databases -o jsonpath='{.data.password}' | base64 --decode
```

---

## 💾 Part 6: Install Longhorn (Distributed Storage)

Longhorn provides distributed block storage for Kubernetes.

```bash
# Install Longhorn
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.5.3/deploy/longhorn.yaml

# Wait for deployment
kubectl get pods -n longhorn-system --watch

# Access Longhorn UI (optional)
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8000:80
```

Create `longhorn-storageclass.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-retain
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "30"
  fromBackup: ""
  fsType: "ext4"
```

```bash
kubectl apply -f longhorn-storageclass.yaml
```

---

## 📦 Part 7: Deploy Core Applications

### 1. Deploy Redis Cluster

Create `redis-cluster.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: redis
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: redis
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
          name: redis
        command:
        - redis-server
        - --appendonly
        - "yes"
        - --cluster-enabled
        - "yes"
        volumeMounts:
        - name: redis-data
          mountPath: /data
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: longhorn-retain
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: redis
spec:
  clusterIP: None
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
```

```bash
kubectl apply -f redis-cluster.yaml
```

### 2. Deploy Keycloak (SSO/IAM)

Create `keycloak-deployment.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: keycloak
---
apiVersion: v1
kind: Secret
metadata:
  name: keycloak-db-secret
  namespace: keycloak
type: Opaque
stringData:
  username: keycloak
  password: "your_secure_password"  # Change this!
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: keycloak
spec:
  replicas: 3
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
      - name: keycloak
        image: quay.io/keycloak/keycloak:23.0
        args:
        - start
        - --hostname=auth.yourplatform.eu
        - --proxy=edge
        - --http-enabled=true
        env:
        - name: KEYCLOAK_ADMIN
          value: "admin"
        - name: KEYCLOAK_ADMIN_PASSWORD
          value: "your_admin_password"  # Change this!
        - name: KC_DB
          value: postgres
        - name: KC_DB_URL
          value: "jdbc:postgresql://sovereign-postgres.databases:5432/keycloak"
        - name: KC_DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: keycloak-db-secret
              key: username
        - name: KC_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: keycloak-db-secret
              key: password
        ports:
        - containerPort: 8080
          name: http
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: keycloak
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: keycloak
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: keycloak
  namespace: keycloak
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - auth.yourplatform.eu
    secretName: keycloak-tls
  rules:
  - host: auth.yourplatform.eu
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: keycloak
            port:
              number: 80
```

```bash
kubectl apply -f keycloak-deployment.yaml

# Check status
kubectl get pods -n keycloak
kubectl get ingress -n keycloak
```

### 3. Deploy Nextcloud

Create `nextcloud-deployment.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nextcloud
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-data
  namespace: nextcloud
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: longhorn-retain
  resources:
    requests:
      storage: 500Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud
  namespace: nextcloud
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nextcloud
  template:
    metadata:
      labels:
        app: nextcloud
    spec:
      containers:
      - name: nextcloud
        image: nextcloud:28-apache
        ports:
        - containerPort: 80
        env:
        - name: POSTGRES_HOST
          value: "sovereign-postgres.databases"
        - name: POSTGRES_DB
          value: "nextcloud"
        - name: POSTGRES_USER
          value: "nextcloud"
        - name: POSTGRES_PASSWORD
          value: "your_db_password"  # Get from K8s secret
        - name: NEXTCLOUD_ADMIN_USER
          value: "admin"
        - name: NEXTCLOUD_ADMIN_PASSWORD
          value: "your_admin_password"
        - name: NEXTCLOUD_TRUSTED_DOMAINS
          value: "nextcloud.yourplatform.eu"
        - name: REDIS_HOST
          value: "redis.redis"
        - name: OVERWRITEPROTOCOL
          value: "https"
        volumeMounts:
        - name: nextcloud-data
          mountPath: /var/www/html
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 2Gi
        readinessProbe:
          httpGet:
            path: /status.php
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /status.php
            port: 80
          initialDelaySeconds: 60
          periodSeconds: 30
      volumes:
      - name: nextcloud-data
        persistentVolumeClaim:
          claimName: nextcloud-data
---
apiVersion: v1
kind: Service
metadata:
  name: nextcloud
  namespace: nextcloud
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nextcloud
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nextcloud
  namespace: nextcloud
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.middlewares: nextcloud-redirect@kubernetescrd
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - nextcloud.yourplatform.eu
    secretName: nextcloud-tls
  rules:
  - host: nextcloud.yourplatform.eu
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nextcloud
            port:
              number: 80
```

```bash
kubectl apply -f nextcloud-deployment.yaml
```

### 4. Deploy Odoo

Create `odoo-deployment.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: odoo
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: odoo-data
  namespace: odoo
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: longhorn-retain
  resources:
    requests:
      storage: 100Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: odoo
  namespace: odoo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: odoo
  template:
    metadata:
      labels:
        app: odoo
    spec:
      containers:
      - name: odoo
        image: odoo:17
        ports:
        - containerPort: 8069
        env:
        - name: HOST
          value: "sovereign-postgres.databases"
        - name: USER
          value: "odoo"
        - name: PASSWORD
          value: "your_db_password"
        - name: DB_PORT
          value: "5432"
        volumeMounts:
        - name: odoo-data
          mountPath: /var/lib/odoo
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
      volumes:
      - name: odoo-data
        persistentVolumeClaim:
          claimName: odoo-data
---
apiVersion: v1
kind: Service
metadata:
  name: odoo
  namespace: odoo
spec:
  ports:
  - port: 80
    targetPort: 8069
  selector:
    app: odoo
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: odoo
  namespace: odoo
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - odoo.yourplatform.eu
    secretName: odoo-tls
  rules:
  - host: odoo.yourplatform.eu
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: odoo
            port:
              number: 80
```

```bash
kubectl apply -f odoo-deployment.yaml
```

---

## 📊 Part 8: Deploy Monitoring Stack

### Install Prometheus & Grafana using kube-prometheus-stack

```bash
# Add Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=your_secure_password \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=100Gi

# Verify installation
kubectl get pods -n monitoring
```

### Access Grafana

```bash
# Port-forward to access Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Or create Ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - grafana.yourplatform.eu
    secretName: grafana-tls
  rules:
  - host: grafana.yourplatform.eu
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-grafana
            port:
              number: 80
EOF
```

---

## 🔄 Part 9: Blue-Green Deployment with K3s

### Strategy 1: Using Kubernetes Deployments (Recommended)

K3s/Kubernetes natively supports rolling updates and blue-green deployments.

#### Rolling Update (Default)

```bash
# Update image version
kubectl set image deployment/nextcloud nextcloud=nextcloud:29-apache -n nextcloud

# Kubernetes automatically:
# 1. Creates new pods with new version
# 2. Waits for them to be ready
# 3. Gradually removes old pods
# 4. Ensures zero downtime

# Check rollout status
kubectl rollout status deployment/nextcloud -n nextcloud

# Rollback if issues
kubectl rollout undo deployment/nextcloud -n nextcloud
```

#### Blue-Green Deployment (Manual Control)

Create `nextcloud-blue-green.yaml`:

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
        image: nextcloud:28-apache  # Current version
        # ... rest of config
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
        image: nextcloud:29-apache  # New version
        # ... rest of config
---
# Service (switch between blue/green)
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
# 1. Deploy green (new version)
kubectl apply -f nextcloud-blue-green.yaml

# 2. Test green deployment internally
kubectl port-forward -n nextcloud deployment/nextcloud-green 8080:80

# 3. If tests pass, switch service to green
kubectl patch service nextcloud -n nextcloud -p '{"spec":{"selector":{"version":"green"}}}'

# 4. Monitor for issues
kubectl logs -n nextcloud -l version=green --tail=100 -f

# 5. If issues, instant rollback to blue
kubectl patch service nextcloud -n nextcloud -p '{"spec":{"selector":{"version":"blue"}}}'

# 6. If successful, delete blue deployment
kubectl delete deployment nextcloud-blue -n nextcloud
```

### Strategy 2: Using Argo Rollouts (Advanced)

For more sophisticated deployments with canary, blue-green, and progressive delivery.

```bash
# Install Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install Argo Rollouts CLI
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

Example Rollout with Blue-Green:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: nextcloud
  namespace: nextcloud
spec:
  replicas: 3
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: nextcloud
  template:
    metadata:
      labels:
        app: nextcloud
    spec:
      containers:
      - name: nextcloud
        image: nextcloud:28-apache
        # ... rest of config
  strategy:
    blueGreen:
      activeService: nextcloud-active
      previewService: nextcloud-preview
      autoPromotionEnabled: false  # Manual approval required
      scaleDownDelaySeconds: 300   # Wait 5 min before scaling down old version
```

```bash
# Trigger deployment
kubectl argo rollouts set image nextcloud nextcloud=nextcloud:29-apache -n nextcloud

# Check status
kubectl argo rollouts status nextcloud -n nextcloud

# Promote to production (after testing preview)
kubectl argo rollouts promote nextcloud -n nextcloud

# Abort if issues
kubectl argo rollouts abort nextcloud -n nextcloud
```

---

## 🔐 Part 10: Backup & Disaster Recovery

### Install Velero (Kubernetes Backup)

```bash
# Install Velero CLI
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.2/velero-v1.12.2-linux-amd64.tar.gz
tar -xvf velero-v1.12.2-linux-amd64.tar.gz
sudo mv velero-v1.12.2-linux-amd64/velero /usr/local/bin/

# Install Velero with MinIO/S3 backend
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.minio:9000

# Create backup schedule (daily)
velero schedule create daily-backup --schedule="0 2 * * *"

# Manual backup
velero backup create manual-backup-$(date +%Y%m%d)

# Restore from backup
velero restore create --from-backup manual-backup-20251211
```

---

## ✅ Verification Checklist

```bash
# Check all nodes
kubectl get nodes

# Check all pods
kubectl get pods --all-namespaces

# Check services
kubectl get svc --all-namespaces

# Check ingress
kubectl get ingress --all-namespaces

# Check certificates
kubectl get certificate --all-namespaces

# Check storage
kubectl get pvc --all-namespaces

# Test external access
curl -I https://nextcloud.yourplatform.eu
curl -I https://odoo.yourplatform.eu
curl -I https://auth.yourplatform.eu
```

---

## 🚀 Advantages of K3s over Docker Swarm

| Feature | K3s | Docker Swarm |
|---------|-----|--------------|
| **Ecosystem** | Full Kubernetes ecosystem | Limited to Docker |
| **Tools** | Helm, Operators, ArgoCD, etc. | Limited tooling |
| **Scaling** | Excellent (1-100+ nodes) | Good (1-50 nodes) |
| **HA** | Native multi-master | Requires manual setup |
| **Storage** | Multiple options (Longhorn, Rook) | Limited options |
| **Monitoring** | kube-prometheus-stack | Manual setup |
| **Auto-scaling** | HPA, VPA, Cluster Autoscaler | Manual only |
| **GitOps** | ArgoCD, Flux | Not available |
| **Security** | Pod Security, Network Policies | Basic |
| **Community** | Massive | Declining |

---

## 📚 Useful Commands Reference

```bash
# K3s management
sudo systemctl status k3s
sudo systemctl restart k3s
sudo journalctl -u k3s -f

# Kubectl shortcuts
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'
alias kd='kubectl describe'
alias kl='kubectl logs'

# Quick pod shell access
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash

# Watch resources
watch kubectl get pods --all-namespaces

# Resource usage
kubectl top nodes
kubectl top pods --all-namespaces

# Debugging
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --tail=100 -f
kubectl events --all-namespaces

# Clean up
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0
```

---

## 🎯 Next Steps

1. ✅ Setup DNS records for all services
2. ✅ Configure SSO integration with Keycloak
3. ✅ Setup automated backups with Velero
4. ✅ Implement GitOps with ArgoCD
5. ✅ Add horizontal pod autoscaling
6. ✅ Setup log aggregation (Loki)
7. ✅ Perform load testing
8. ✅ Security hardening and audit

---

**Document Version**: 1.0 - K3s Edition  
**Last Updated**: December 11, 2025  
**Estimated Setup Time**: 1-2 days
