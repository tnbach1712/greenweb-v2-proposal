# Quick Start Guide - K3s European Cloud Platform

## 🚀 Get Started in 1 Hour

This guide will help you deploy a working K3s cluster with Nextcloud in under 1 hour.

---

## ✅ Prerequisites

- **3 Ubuntu 22.04 servers** (PlanetHoster)
  - Server 1: 8 vCPU, 16GB RAM
  - Server 2: 8 vCPU, 16GB RAM  
  - Server 3: 4 vCPU, 8GB RAM (optional for 3-master HA)
- **Root or sudo access**
- **SSH key authentication configured**
- **Domain name** with access to DNS settings

---

## 📋 30-Minute Minimal Setup

### Step 1: Install K3s on First Server (2 minutes)

```bash
# SSH to Server 1
ssh root@<server1-ip>

# Install K3s
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --write-kubeconfig-mode=644

# Wait for it to be ready
kubectl get nodes

# Get join token for other nodes
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Step 2: Join Second Server (2 minutes)

```bash
# SSH to Server 2
ssh root@<server2-ip>

# Join cluster (replace <TOKEN> and <SERVER1_IP>)
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://<SERVER1_IP>:6443 \
  --token <TOKEN> \
  --write-kubeconfig-mode=644

# Verify from Server 1
kubectl get nodes
# Should show both nodes
```

### Step 3: Install cert-manager (2 minutes)

```bash
# On Server 1
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml

# Wait for pods to be ready
kubectl get pods -n cert-manager --watch
```

### Step 4: Create Let's Encrypt Issuer (1 minute)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@yourplatform.eu
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
EOF
```

### Step 5: Deploy PostgreSQL (3 minutes)

```bash
# Create namespace
kubectl create namespace databases

# Deploy PostgreSQL
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: databases
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "ChangeMeInProduction123!"
        - name: POSTGRES_DB
          value: "nextcloud"
        - name: POSTGRES_USER
          value: "nextcloud"
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 50Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: databases
spec:
  clusterIP: None
  ports:
  - port: 5432
  selector:
    app: postgres
EOF

# Wait for PostgreSQL to be ready
kubectl get pods -n databases --watch
```

### Step 6: Deploy Nextcloud (5 minutes)

```bash
# Create namespace
kubectl create namespace nextcloud

# Deploy Nextcloud
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-data
  namespace: nextcloud
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud
  namespace: nextcloud
spec:
  replicas: 2
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
          value: "postgres.databases"
        - name: POSTGRES_DB
          value: "nextcloud"
        - name: POSTGRES_USER
          value: "nextcloud"
        - name: POSTGRES_PASSWORD
          value: "ChangeMeInProduction123!"
        - name: NEXTCLOUD_ADMIN_USER
          value: "admin"
        - name: NEXTCLOUD_ADMIN_PASSWORD
          value: "ChangeMeAdmin123!"
        - name: NEXTCLOUD_TRUSTED_DOMAINS
          value: "nextcloud.yourplatform.eu"
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
EOF

# Wait for Nextcloud to be ready (this may take 5 minutes)
kubectl get pods -n nextcloud --watch
```

### Step 7: Configure DNS (2 minutes)

```bash
# Get Traefik LoadBalancer IP
kubectl get svc -n kube-system traefik

# Note the EXTERNAL-IP (or Cluster IP if no LoadBalancer)
```

**In your DNS provider (e.g., Cloudflare):**
- Create A record: `nextcloud.yourplatform.eu` → `<EXTERNAL-IP>`

### Step 8: Access Nextcloud (1 minute)

Wait 2-3 minutes for DNS propagation and SSL certificate generation, then:

```bash
# Check certificate status
kubectl get certificate -n nextcloud

# When ready, visit:
https://nextcloud.yourplatform.eu

# Login:
# Username: admin
# Password: ChangeMeAdmin123!
```

**🎉 Congratulations! You have a working Nextcloud on K3s!**

---

## 🔧 Common Issues & Fixes

### Issue 1: Pods stuck in "Pending"

```bash
# Check events
kubectl describe pod <pod-name> -n nextcloud

# Common cause: Not enough resources
# Solution: Add more nodes or reduce resource requests
```

### Issue 2: "no matches for kind Ingress"

```bash
# Traefik not installed or old K3s version
# Solution: Reinstall K3s or install Traefik manually
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v2.10/docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml
```

### Issue 3: SSL certificate not issued

```bash
# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# Check certificate status
kubectl describe certificate nextcloud-tls -n nextcloud

# Common cause: DNS not propagated or firewall blocking port 80
# Solution: Wait for DNS, check firewall
```

### Issue 4: Can't access from browser

```bash
# Check Ingress
kubectl get ingress -n nextcloud

# Check if Traefik is running
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik

# Test from server
curl -k https://nextcloud.yourplatform.eu

# If works from server but not browser:
# - Check DNS propagation: nslookup nextcloud.yourplatform.eu
# - Check firewall: ports 80, 443 must be open
```

---

## 📊 Verify Your Setup

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

# Resource usage
kubectl top nodes
kubectl top pods -n nextcloud
```

---

## 🔄 Next Steps (30 minutes each)

### Deploy Keycloak (SSO)

```bash
kubectl create namespace keycloak

cat <<EOF | kubectl apply -f -
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
        args: ["start-dev"]
        env:
        - name: KEYCLOAK_ADMIN
          value: "admin"
        - name: KEYCLOAK_ADMIN_PASSWORD
          value: "ChangeMeAdmin123!"
        ports:
        - containerPort: 8080
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
EOF
```

Don't forget to add DNS: `auth.yourplatform.eu` → `<EXTERNAL-IP>`

### Deploy Odoo (ERP/CRM)

```bash
kubectl create namespace odoo

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: odoo-data
  namespace: odoo
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
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
          value: "postgres.databases"
        - name: USER
          value: "odoo"
        - name: PASSWORD
          value: "ChangeMeOdoo123!"
        volumeMounts:
        - name: odoo-data
          mountPath: /var/lib/odoo
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
EOF
```

**Note:** Create Odoo database in PostgreSQL first:
```bash
kubectl exec -it postgres-0 -n databases -- psql -U postgres
CREATE DATABASE odoo;
CREATE USER odoo WITH ENCRYPTED PASSWORD 'ChangeMeOdoo123!';
GRANT ALL PRIVILEGES ON DATABASE odoo TO odoo;
\q
```

Don't forget to add DNS: `odoo.yourplatform.eu` → `<EXTERNAL-IP>`

---

## 📊 Install Monitoring (15 minutes)

```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=ChangeMeGrafana123!

# Wait for pods
kubectl get pods -n monitoring --watch

# Access Grafana (port-forward)
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Or create Ingress for Grafana
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
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

Access Grafana at `https://grafana.yourplatform.eu`
- Username: `admin`
- Password: `ChangeMeGrafana123!`

---

## 🎯 Production Checklist

Before going live with customers:

- [ ] Change all default passwords
- [ ] Setup proper SMTP for email delivery
- [ ] Configure backups (Velero)
- [ ] Setup monitoring alerts
- [ ] Configure firewall rules
- [ ] Enable network policies
- [ ] Setup log aggregation
- [ ] Perform security audit
- [ ] Load testing
- [ ] Document runbooks
- [ ] Setup on-call rotation
- [ ] Create disaster recovery plan

---

## 🔐 Security Hardening

```bash
# Enable firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 6443/tcp
sudo ufw allow 443/tcp
sudo ufw allow 80/tcp
sudo ufw enable

# Install Fail2ban
sudo apt install fail2ban -y
sudo systemctl enable fail2ban

# Disable password authentication
sudo nano /etc/ssh/sshd_config
# Set: PasswordAuthentication no
sudo systemctl restart sshd
```

---

## 📚 Useful Commands

```bash
# Kubectl shortcuts
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'

# Watch resources
watch kubectl get pods --all-namespaces

# Shell into pod
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash

# View logs
kubectl logs -f <pod-name> -n <namespace>

# Describe resource
kubectl describe pod <pod-name> -n <namespace>

# Delete and recreate pod
kubectl delete pod <pod-name> -n <namespace>

# Scale deployment
kubectl scale deployment nextcloud --replicas=3 -n nextcloud

# Update image
kubectl set image deployment/nextcloud nextcloud=nextcloud:29 -n nextcloud

# Rollback
kubectl rollout undo deployment/nextcloud -n nextcloud
```

---

## 🆘 Support

If you need help:
1. Check the logs: `kubectl logs <pod-name> -n <namespace>`
2. Check events: `kubectl get events -n <namespace>`
3. Describe the resource: `kubectl describe pod <pod-name> -n <namespace>`
4. Check K3s documentation: https://docs.k3s.io/
5. Check Kubernetes documentation: https://kubernetes.io/docs/

---

**Quick Start Version**: 1.0  
**Last Updated**: December 11, 2025  
**Time to Complete**: 30-60 minutes for basic setup
