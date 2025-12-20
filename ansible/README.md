# K3s Ansible Deployment Guide

This directory contains Ansible playbooks to install K3s Kubernetes cluster on private nodes through a bastion host.

## Infrastructure

- **Bastion Host**: 51.159.77.46 (public IP)
- **Node 1**: 192.168.0.100 (K3s master)
- **Node 2**: 192.168.0.101 (K3s worker)

---

## Prerequisites

### 1. Install Ansible on your local machine

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install ansible -y

# macOS
brew install ansible

# Verify installation
ansible --version
```

### 2. Setup SSH access

First, configure SSH to access private nodes through bastion:

```bash
# Edit SSH config
nano ~/.ssh/config
```

Add this configuration:

```
# Bastion Host
Host bastion
    HostName 51.159.77.46
    User root
    IdentityFile ~/.ssh/id_rsa
    ServerAliveInterval 60

# Private Node 1
Host node1
    HostName 192.168.0.100
    User root
    ProxyJump bastion
    IdentityFile ~/.ssh/id_rsa

# Private Node 2
Host node2
    HostName 192.168.0.101
    User root
    ProxyJump bastion
    IdentityFile ~/.ssh/id_rsa
```

### 3. Copy SSH keys

```bash
# Copy SSH key to bastion
ssh-copy-id root@51.159.77.46

# Copy SSH key to private nodes (through bastion)
ssh-copy-id -o ProxyJump=root@51.159.77.46 root@192.168.0.100
ssh-copy-id -o ProxyJump=root@51.159.77.46 root@192.168.0.101

# Test connections
ssh bastion echo "Bastion OK"
ssh node1 echo "Node1 OK"
ssh node2 echo "Node2 OK"
```

---

## Quick Start

### Step 1: Test Ansible connectivity

```bash
cd /home/bach/code/plan_df/ansible

# Test connection to all nodes
ansible all -m ping

# Test connection and gather info
ansible-playbook test-connection.yml
```

Expected output:
```
node1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
node2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Step 2: Install K3s cluster

```bash
# Install K3s on both nodes
ansible-playbook install-k3s.yml

# This will:
# 1. Prepare all nodes (disable swap, enable IP forwarding)
# 2. Install K3s server on node1 (master)
# 3. Install K3s agent on node2 (worker)
# 4. Download kubeconfig to ./kubeconfig.yaml
```

Installation takes about 5-10 minutes.

### Step 3: Access the cluster

```bash
# Set kubeconfig
export KUBECONFIG=/home/bach/code/plan_df/ansible/kubeconfig.yaml

# Or copy to default location
mkdir -p ~/.kube
cp /home/bach/code/plan_df/ansible/kubeconfig.yaml ~/.kube/config

# Verify cluster
kubectl get nodes
kubectl get pods -A
```

---

## Detailed Usage

### Test individual hosts

```bash
# Ping specific host
ansible node1 -m ping

# Run command on node1
ansible node1 -a "uptime"

# Run command on all K3s nodes
ansible k3s_cluster -a "free -h"

# Get system info
ansible all -m setup -a "filter=ansible_distribution*"
```

### Install K3s with custom options

Edit `install-k3s.yml` and modify variables:

```yaml
vars:
  k3s_version: v1.28.5+k3s1  # Change K3s version
  cluster_cidr: "10.42.0.0/16"
  service_cidr: "10.43.0.0/16"
```

Then run:

```bash
ansible-playbook install-k3s.yml
```

### Uninstall K3s

```bash
# Uninstall from master
ssh node1 '/usr/local/bin/k3s-uninstall.sh'

# Uninstall from worker
ssh node2 '/usr/local/bin/k3s-agent-uninstall.sh'
```

### View installation logs

```bash
# Check K3s service status
ansible k3s_masters -a "systemctl status k3s"
ansible k3s_workers -a "systemctl status k3s-agent"

# View K3s logs
ansible node1 -a "journalctl -u k3s -n 50"
ansible node2 -a "journalctl -u k3s-agent -n 50"
```

---

## Troubleshooting

### SSH connection issues

```bash
# Test SSH manually
ssh -J root@51.159.77.46 root@192.168.0.100
ssh -J root@51.159.77.46 root@192.168.0.101

# Test with verbose
ssh -vvv -J root@51.159.77.46 root@192.168.0.100

# Check if nodes can reach each other
ssh node1 'ping -c 3 192.168.0.101'
ssh node2 'ping -c 3 192.168.0.100'
```

### Ansible connection issues

```bash
# Test with verbose output
ansible all -m ping -vvv

# Use different user
ansible all -m ping -u root

# Check SSH config
ansible all -m shell -a "echo $SSH_CONNECTION"
```

### K3s installation issues

```bash
# Check system requirements
ansible k3s_cluster -m shell -a "free -h"
ansible k3s_cluster -m shell -a "df -h"

# Verify no swap is active
ansible k3s_cluster -m shell -a "swapon --show"

# Check if port 6443 is accessible
ansible node2 -m shell -a "nc -zv 192.168.0.100 6443"

# Re-run installation
ansible-playbook install-k3s.yml --tags "install"
```

### Node not joining cluster

```bash
# On master, get token
ssh node1 'cat /var/lib/rancher/k3s/server/node-token'

# On worker, check agent logs
ssh node2 'journalctl -u k3s-agent -f'

# Check connectivity from worker to master
ssh node2 'curl -k https://192.168.0.100:6443/ping'
```

---

## Advanced Configuration

### HA Setup (Both nodes as masters)

Edit `inventory.ini`:

```ini
[k3s_masters]
node1 ansible_host=192.168.0.100
node2 ansible_host=192.168.0.101

[k3s_workers]
# Empty - all nodes are masters
```

Then run:

```bash
ansible-playbook install-k3s.yml
```

### Add MetalLB LoadBalancer

```bash
# After K3s is installed
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.0/config/manifests/metallb-native.yaml

# Configure IP pool
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.200-192.168.0.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
EOF
```

### Install Traefik Ingress

```bash
# Traefik is included in K3s but disabled by default in our setup
# To enable:
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v2.10/docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml
```

---

## Useful Commands

```bash
# Check all nodes
kubectl get nodes -o wide

# View all pods
kubectl get pods -A

# Deploy test application
kubectl create deployment nginx --image=nginx --replicas=2
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Get kubeconfig from master
scp -o ProxyJump=root@51.159.77.46 root@192.168.0.100:/etc/rancher/k3s/k3s.yaml ./kubeconfig.yaml

# Access cluster from anywhere
export KUBECONFIG=/path/to/kubeconfig.yaml
# Update server address in kubeconfig
sed -i 's/127.0.0.1/192.168.0.100/g' kubeconfig.yaml
```

---

## Next Steps

1. ✅ Install K3s cluster
2. ✅ Configure kubectl access
3. 📦 Install MetalLB for LoadBalancer
4. 🌐 Install Traefik Ingress Controller
5. 🔐 Install cert-manager for SSL
6. 📊 Install monitoring (Prometheus + Grafana)
7. 🗄️ Setup external PostgreSQL
8. 💾 Setup external Redis
9. 🚀 Deploy applications (Nextcloud, Odoo, etc.)

See the main documentation in `/home/bach/code/plan_df/` for detailed guides.
