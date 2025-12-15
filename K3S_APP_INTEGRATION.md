# K3s Application Integration with External Services

## 🎯 Purpose

This guide shows how to **connect K3s applications** (Nextcloud, Odoo, Keycloak, Mautic) to **external PostgreSQL and Redis clusters**.

---

## 📋 Prerequisites

Before following this guide, ensure you have:
- ✅ K3s cluster running ([K3S_SETUP_GUIDE.md](K3S_SETUP_GUIDE.md))
- ✅ External PostgreSQL cluster ([EXTERNAL_SERVICES_SETUP.md](EXTERNAL_SERVICES_SETUP.md))
- ✅ External Redis cluster ([EXTERNAL_SERVICES_SETUP.md](EXTERNAL_SERVICES_SETUP.md))
- ✅ S3 object storage configured (PlanetHoster or MinIO)

---

## 🔐 Step 1: Create Kubernetes Secrets

### 1.1 PostgreSQL Connection Secret

```bash
# Create namespace for secrets (if not exists)
kubectl create namespace shared-secrets

# PostgreSQL connection details
kubectl create secret generic postgresql-credentials \
  --from-literal=host=<POSTGRES_VM1_IP> \
  --from-literal=port=6432 \
  --from-literal=username=app_user \
  --from-literal=password=<CHANGE_ME_APP_USER_PASSWORD> \
  -n shared-secrets

# Or use PgBouncer for connection pooling
kubectl create secret generic pgbouncer-endpoint \
  --from-literal=host=<POSTGRES_VM1_IP> \
  --from-literal=port=6432 \
  -n shared-secrets
```

### 1.2 Redis Connection Secret

```bash
# Redis Sentinel configuration
kubectl create secret generic redis-credentials \
  --from-literal=sentinels=<REDIS_VM1_IP>:26379,<REDIS_VM2_IP>:26379,<REDIS_VM3_IP>:26379 \
  --from-literal=master-name=mymaster \
  --from-literal=password=<CHANGE_ME_REDIS_PASSWORD> \
  -n shared-secrets
```

### 1.3 S3 Storage Secret

```bash
# S3 credentials for object storage
kubectl create secret generic s3-credentials \
  --from-literal=endpoint=<S3_ENDPOINT> \
  --from-literal=bucket=<BUCKET_NAME> \
  --from-literal=region=<REGION> \
  --from-literal=access-key=<ACCESS_KEY> \
  --from-literal=secret-key=<SECRET_KEY> \
  -n shared-secrets
```

---

## 📦 Step 2: Deploy Nextcloud with External Services

### 2.1 Nextcloud ConfigMap

```yaml
# nextcloud-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nextcloud-config
  namespace: nextcloud
data:
  # PostgreSQL configuration
  POSTGRES_DB: "nextcloud_db"
  POSTGRES_HOST: "<POSTGRES_VM1_IP>"  # PgBouncer endpoint
  POSTGRES_PORT: "6432"
  
  # Redis configuration (session and file locking)
  REDIS_HOST_SENTINEL: "<REDIS_VM1_IP>:26379,<REDIS_VM2_IP>:26379,<REDIS_VM3_IP>:26379"
  REDIS_MASTER_NAME: "mymaster"
  
  # S3 primary storage
  OBJECTSTORE_S3_BUCKET: "<BUCKET_NAME>"
  OBJECTSTORE_S3_REGION: "<REGION>"
  OBJECTSTORE_S3_HOST: "<S3_ENDPOINT>"
  OBJECTSTORE_S3_PORT: "443"
  OBJECTSTORE_S3_SSL: "true"
  OBJECTSTORE_S3_USEPATH_STYLE: "false"
  
  # Nextcloud settings
  NEXTCLOUD_TRUSTED_DOMAINS: "nextcloud.digitalforest.eu"
  TRUSTED_PROXIES: "10.42.0.0/16"  # K3s pod network
  OVERWRITEPROTOCOL: "https"
```

### 2.2 Nextcloud Deployment

```yaml
# nextcloud-deployment.yaml
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
        image: nextcloud:29-apache
        ports:
        - containerPort: 80
          name: http
        
        env:
        # PostgreSQL connection from secret
        - name: POSTGRES_HOST
          valueFrom:
            configMapKeyRef:
              name: nextcloud-config
              key: POSTGRES_HOST
        - name: POSTGRES_PORT
          valueFrom:
            configMapKeyRef:
              name: nextcloud-config
              key: POSTGRES_PORT
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: nextcloud-config
              key: POSTGRES_DB
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgresql-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-credentials
              key: password
        
        # Redis configuration
        - name: REDIS_HOST_SENTINEL
          valueFrom:
            configMapKeyRef:
              name: nextcloud-config
              key: REDIS_HOST_SENTINEL
        - name: REDIS_MASTER_NAME
          valueFrom:
            configMapKeyRef:
              name: nextcloud-config
              key: REDIS_MASTER_NAME
        - name: REDIS_HOST_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: password
        
        # S3 object storage
        - name: OBJECTSTORE_S3_BUCKET
          valueFrom:
            configMapKeyRef:
              name: nextcloud-config
              key: OBJECTSTORE_S3_BUCKET
        - name: OBJECTSTORE_S3_REGION
          valueFrom:
            configMapKeyRef:
              name: nextcloud-config
              key: OBJECTSTORE_S3_REGION
        - name: OBJECTSTORE_S3_HOST
          valueFrom:
            configMapKeyRef:
              name: nextcloud-config
              key: OBJECTSTORE_S3_HOST
        - name: OBJECTSTORE_S3_KEY
          valueFrom:
            secretKeyRef:
              name: s3-credentials
              key: access-key
        - name: OBJECTSTORE_S3_SECRET
          valueFrom:
            secretKeyRef:
              name: s3-credentials
              key: secret-key
        
        # Nextcloud settings
        - name: NEXTCLOUD_TRUSTED_DOMAINS
          valueFrom:
            configMapKeyRef:
              name: nextcloud-config
              key: NEXTCLOUD_TRUSTED_DOMAINS
        - name: TRUSTED_PROXIES
          valueFrom:
            configMapKeyRef:
              name: nextcloud-config
              key: TRUSTED_PROXIES
        - name: OVERWRITEPROTOCOL
          valueFrom:
            configMapKeyRef:
              name: nextcloud-config
              key: OVERWRITEPROTOCOL
        
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        
        livenessProbe:
          httpGet:
            path: /status.php
            port: 80
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /status.php
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
---
apiVersion: v1
kind: Service
metadata:
  name: nextcloud
  namespace: nextcloud
spec:
  selector:
    app: nextcloud
  ports:
  - port: 80
    targetPort: 80
    name: http
  type: ClusterIP
```

### 2.3 Nextcloud Ingress

```yaml
# nextcloud-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nextcloud
  namespace: nextcloud
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.middlewares: nextcloud-redirect@kubernetescrd
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - nextcloud.digitalforest.eu
    secretName: nextcloud-tls
  rules:
  - host: nextcloud.digitalforest.eu
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

### 2.4 Deploy Nextcloud

```bash
# Create namespace
kubectl create namespace nextcloud

# Apply configurations
kubectl apply -f nextcloud-config.yaml
kubectl apply -f nextcloud-deployment.yaml
kubectl apply -f nextcloud-ingress.yaml

# Check deployment
kubectl get pods -n nextcloud
kubectl logs -f deployment/nextcloud -n nextcloud

# Verify database connection
kubectl exec -it deployment/nextcloud -n nextcloud -- \
  php occ db:add-missing-indices
```

---

## 🏢 Step 3: Deploy Odoo with External Services

### 3.1 Odoo ConfigMap

```yaml
# odoo-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: odoo-config
  namespace: odoo
data:
  # PostgreSQL configuration
  HOST: "<POSTGRES_VM1_IP>"
  PORT: "6432"
  USER: "app_user"
  DB_NAME: "odoo_db"
  
  # Odoo configuration
  LIMIT_TIME_CPU: "600"
  LIMIT_TIME_REAL: "1200"
  MAX_CRON_THREADS: "2"
  WORKERS: "4"
```

### 3.2 Odoo Deployment

```yaml
# odoo-deployment.yaml
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
          name: http
        
        env:
        - name: HOST
          valueFrom:
            configMapKeyRef:
              name: odoo-config
              key: HOST
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: odoo-config
              key: PORT
        - name: USER
          valueFrom:
            secretKeyRef:
              name: postgresql-credentials
              key: username
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-credentials
              key: password
        
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        
        livenessProbe:
          httpGet:
            path: /web/health
            port: 8069
          initialDelaySeconds: 60
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /web/health
            port: 8069
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: odoo
  namespace: odoo
spec:
  selector:
    app: odoo
  ports:
  - port: 8069
    targetPort: 8069
  type: ClusterIP
```

---

## 🔑 Step 4: Deploy Keycloak with External Services

### 4.1 Keycloak Deployment

```yaml
# keycloak-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: keycloak
spec:
  replicas: 2
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
        - --optimized
        - --hostname=keycloak.digitalforest.eu
        - --proxy=edge
        
        env:
        # Database configuration
        - name: KC_DB
          value: "postgres"
        - name: KC_DB_URL_HOST
          value: "<POSTGRES_VM1_IP>"
        - name: KC_DB_URL_PORT
          value: "6432"
        - name: KC_DB_URL_DATABASE
          value: "keycloak_db"
        - name: KC_DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: postgresql-credentials
              key: username
        - name: KC_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-credentials
              key: password
        
        # Keycloak admin credentials
        - name: KEYCLOAK_ADMIN
          value: "admin"
        - name: KEYCLOAK_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: keycloak-admin
              key: password
        
        # Caching with Infinispan (embedded)
        - name: KC_CACHE
          value: "ispn"
        - name: KC_CACHE_STACK
          value: "kubernetes"
        
        ports:
        - containerPort: 8080
          name: http
        
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: keycloak
spec:
  selector:
    app: keycloak
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
```

---

## 📧 Step 5: Deploy Mautic with External Services

### 5.1 Mautic Deployment

```yaml
# mautic-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mautic
  namespace: mautic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mautic
  template:
    metadata:
      labels:
        app: mautic
    spec:
      containers:
      - name: mautic
        image: mautic/mautic:v5-apache
        ports:
        - containerPort: 80
          name: http
        
        env:
        # Database configuration
        - name: MAUTIC_DB_HOST
          value: "<POSTGRES_VM1_IP>:6432"
        - name: MAUTIC_DB_NAME
          value: "mautic_db"
        - name: MAUTIC_DB_USER
          valueFrom:
            secretKeyRef:
              name: postgresql-credentials
              key: username
        - name: MAUTIC_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-credentials
              key: password
        
        # Redis cache
        - name: MAUTIC_REDIS_SENTINELS
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: sentinels
        - name: MAUTIC_REDIS_MASTER_NAME
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: master-name
        - name: MAUTIC_REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: password
        
        # Mautic configuration
        - name: MAUTIC_TRUSTED_PROXIES
          value: "0.0.0.0/0"
        - name: MAUTIC_SITE_URL
          value: "https://mautic.digitalforest.eu"
        
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
---
apiVersion: v1
kind: Service
metadata:
  name: mautic
  namespace: mautic
spec:
  selector:
    app: mautic
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

---

## 🔍 Step 6: Verify Connections

### 6.1 Test PostgreSQL Connection

```bash
# From Nextcloud pod
kubectl exec -it deployment/nextcloud -n nextcloud -- \
  psql "postgresql://app_user:<PASSWORD>@<POSTGRES_VM1_IP>:6432/nextcloud_db" -c "SELECT version();"

# From Odoo pod
kubectl exec -it deployment/odoo -n odoo -- \
  psql "postgresql://app_user:<PASSWORD>@<POSTGRES_VM1_IP>:6432/odoo_db" -c "SELECT version();"
```

### 6.2 Test Redis Connection

```bash
# Install redis-cli in a test pod
kubectl run redis-test --image=redis:alpine -it --rm -- sh

# Inside the pod:
redis-cli -h <REDIS_VM1_IP> -p 6379 -a <REDIS_PASSWORD> PING
# Should return: PONG

# Test Sentinel
redis-cli -h <REDIS_VM1_IP> -p 26379 SENTINEL get-master-addr-by-name mymaster
```

### 6.3 Test S3 Connection

```bash
# Install s3cmd in a test pod
kubectl run s3-test --image=alpine -it --rm -- sh

# Inside the pod:
apk add py3-pip
pip3 install s3cmd

s3cmd --access_key=<ACCESS_KEY> \
      --secret_key=<SECRET_KEY> \
      --host=<S3_ENDPOINT> \
      --host-bucket='%(bucket)s.<S3_ENDPOINT>' \
      ls s3://<BUCKET_NAME>/
```

---

## 📊 Step 7: Monitor Connections

### 7.1 Check Application Logs

```bash
# Nextcloud logs
kubectl logs -f deployment/nextcloud -n nextcloud | grep -i "database\|redis\|s3"

# Odoo logs
kubectl logs -f deployment/odoo -n odoo | grep -i "database"

# Keycloak logs
kubectl logs -f deployment/keycloak -n keycloak | grep -i "database"
```

### 7.2 Monitor PostgreSQL Connections

On PostgreSQL VM:

```bash
# Active connections per database
psql -h localhost -U postgres -c "
SELECT datname, count(*) as connections 
FROM pg_stat_activity 
WHERE datname IN ('nextcloud_db', 'odoo_db', 'keycloak_db', 'mautic_db')
GROUP BY datname;
"

# PgBouncer pool status
psql -h localhost -U postgres -p 6432 -c "SHOW POOLS;"
```

### 7.3 Monitor Redis Connections

On Redis VM:

```bash
# Connected clients
redis-cli -a <REDIS_PASSWORD> INFO clients

# Memory usage
redis-cli -a <REDIS_PASSWORD> INFO memory

# Check keys from applications
redis-cli -a <REDIS_PASSWORD> --scan --pattern "nextcloud*" | head -10
```

---

## 🔧 Troubleshooting

### Issue: Application can't connect to PostgreSQL

**Solution:**
```bash
# 1. Verify firewall rules on PostgreSQL VM
sudo ufw status | grep 6432

# 2. Test connection from K3s node
telnet <POSTGRES_VM1_IP> 6432

# 3. Check PgBouncer logs
sudo tail -f /var/log/postgresql/pgbouncer.log

# 4. Check pod DNS resolution
kubectl exec -it deployment/nextcloud -n nextcloud -- nslookup <POSTGRES_VM1_IP>
```

### Issue: Redis connection timeout

**Solution:**
```bash
# 1. Check Sentinel status
redis-cli -h <REDIS_VM1_IP> -p 26379 SENTINEL masters

# 2. Verify firewall rules
sudo ufw status | grep 6379
sudo ufw status | grep 26379

# 3. Test from K3s node
telnet <REDIS_VM1_IP> 6379
telnet <REDIS_VM1_IP> 26379
```

### Issue: S3 storage not working

**Solution:**
```bash
# 1. Verify S3 credentials
kubectl get secret s3-credentials -n shared-secrets -o yaml

# 2. Test S3 endpoint from K3s node
curl -I https://<S3_ENDPOINT>

# 3. Check Nextcloud S3 config
kubectl exec -it deployment/nextcloud -n nextcloud -- \
  php occ config:list | grep objectstore
```

---

## ✅ Checklist

- [ ] Kubernetes secrets created for PostgreSQL, Redis, S3
- [ ] Nextcloud deployed and connected to external services
- [ ] Odoo deployed and connected to PostgreSQL
- [ ] Keycloak deployed and connected to PostgreSQL
- [ ] Mautic deployed and connected to PostgreSQL and Redis
- [ ] Database connections verified from all applications
- [ ] Redis connections working (session storage)
- [ ] S3 object storage configured for Nextcloud
- [ ] Monitoring alerts configured in Prometheus/Grafana
- [ ] Backup verification completed

---

## 📚 Related Documentation

- [EXTERNAL_SERVICES_SETUP.md](EXTERNAL_SERVICES_SETUP.md) - Setup PostgreSQL, Redis, Monitoring
- [K3S_SETUP_GUIDE.md](K3S_SETUP_GUIDE.md) - K3s cluster setup
- [ARCHITECTURE_PLAN.md](ARCHITECTURE_PLAN.md) - Overall architecture

---

**Next Steps:** Configure monitoring dashboards, setup automated backups, test disaster recovery procedures.
