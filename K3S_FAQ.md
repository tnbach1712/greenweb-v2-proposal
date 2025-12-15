# K3s FAQ - Frequently Asked Questions

## 🤔 Architecture & Concepts

### Q1: K3s có dùng kiểu master-worker như K8s truyền thống không?

**Trả lời:** K3s **linh hoạt hơn** K8s truyền thống. Bạn có thể chọn:

**Option A: All-in-One Server Nodes (Recommended cho 3-5 nodes) ✅**
- Tất cả nodes đều là **server nodes**
- Mỗi node vừa chạy control plane, vừa chạy workloads
- **Đây là default behavior của K3s**
- Hiệu quả hơn cho small-medium clusters

```
┌────────────────────────────┐
│  Server Node 1             │
│  • Control Plane ✓         │
│  • Your Apps ✓             │
│  • etcd ✓                  │
└────────────────────────────┘
┌────────────────────────────┐
│  Server Node 2             │
│  • Control Plane ✓         │
│  • Your Apps ✓             │
│  • etcd ✓                  │
└────────────────────────────┘
┌────────────────────────────┐
│  Server Node 3             │
│  • Control Plane ✓         │
│  • Your Apps ✓             │
│  • etcd ✓                  │
└────────────────────────────┘
```

**Option B: Dedicated Masters + Workers (như K8s truyền thống)**
- Một số nodes là **server nodes** (chỉ control plane, tainted)
- Các nodes còn lại là **agent nodes** (chỉ workloads)
- Dùng khi có nhiều nodes (10+)

```
┌────────────────────────────┐
│  Server Nodes (1-3)        │
│  • Control Plane ✓         │
│  • Your Apps ✗ (tainted)   │
│  • etcd ✓                  │
└────────────────────────────┘
┌────────────────────────────┐
│  Agent Nodes (4-N)         │
│  • Control Plane ✗         │
│  • Your Apps ✓             │
└────────────────────────────┘
```

**Recommendation cho European Cloud Platform:** 
- Dùng **Option A** (All-in-One) với 3 server nodes
- Hiệu quả hơn, đơn giản hơn, tiết kiệm tài nguyên

---

### Q2: Tại sao K3s server nodes có thể chạy workloads mà K8s masters thì không?

**Trả lời:** Đây là **design philosophy** khác nhau:

**Traditional Kubernetes (kubeadm):**
```bash
# Masters automatically tainted
kubectl describe node master1 | grep Taint
# Taints: node-role.kubernetes.io/master:NoSchedule
#         node-role.kubernetes.io/control-plane:NoSchedule

# Pods will NOT schedule on masters by default
```

**Why?** 
- Designed for large clusters (100+ nodes)
- Separate control plane for stability
- Masters focus on cluster management

**K3s (default):**
```bash
# Server nodes NOT tainted
kubectl describe node server1 | grep Taint
# Taints: <none>

# Pods WILL schedule on servers by default
```

**Why?**
- Designed for edge/small clusters (1-10 nodes)
- Resource efficiency (don't waste 3 servers just for control plane)
- Still highly available (3 control planes)

**💡 Taint là gì?** Xem giải thích chi tiết: **[KUBERNETES_TAINTS_EXPLAINED.md](KUBERNETES_TAINTS_EXPLAINED.md)**

**You can change K3s to behave like K8s:**
```bash
# Taint server nodes manually
kubectl taint nodes server1 node-role.kubernetes.io/master=true:NoSchedule
kubectl taint nodes server2 node-role.kubernetes.io/master=true:NoSchedule
kubectl taint nodes server3 node-role.kubernetes.io/master=true:NoSchedule

# Now servers won't run workloads (like traditional K8s)
```

---

### Q3: Làm sao biết node là server hay agent?

**Trả lời:** Xem role và process:

```bash
# Check node roles
kubectl get nodes

# Output examples:
# Server node:
# NAME      STATUS   ROLES                       AGE   VERSION
# server1   Ready    control-plane,master        5m    v1.28.x+k3s1

# Agent node (if you have one):
# NAME      STATUS   ROLES    AGE   VERSION
# agent1    Ready    <none>   3m    v1.28.x+k3s1

# Check what's running
ps aux | grep k3s

# On server nodes:
# /usr/local/bin/k3s server

# On agent nodes:
# /usr/local/bin/k3s agent
```

---

### Q4: Với 3 nodes, nên setup như thế nào?

**Trả lời:** **3 server nodes (all-in-one)** ✅

**Setup:**
```bash
# Node 1: Initialize cluster
curl -sfL https://get.k3s.io | sh -s - server --cluster-init

# Node 2: Join as server
curl -sfL https://get.k3s.io | sh -s - server --server https://node1:6443 --token xxx

# Node 3: Join as server
curl -sfL https://get.k3s.io | sh -s - server --server https://node1:6443 --token xxx
```

**Result:**
- 3 control planes (HA)
- 3 nodes for workloads
- Efficient resource usage

**❌ DON'T do this with 3 nodes:**
```bash
# Node 1-3: Servers only (tainted)
# Node 4+: Agents (workers)
# This wastes 3 servers!
```

---

### Q5: etcd cluster hoạt động như thế nào trong K3s?

**Trả lời:** K3s có 2 options:

**Option 1: Embedded etcd (Recommended) ✅**
```bash
# Use --cluster-init
curl -sfL https://get.k3s.io | sh -s - server --cluster-init

# Creates embedded etcd cluster
# Each server node runs its own etcd member
# Automatically handles quorum and leader election
```

**Result:**
```
Server 1: API server + Scheduler + Controller + etcd member
Server 2: API server + Scheduler + Controller + etcd member  
Server 3: API server + Scheduler + Controller + etcd member

etcd cluster: 3 members, quorum = 2
```

**Option 2: External etcd (Advanced)**
```bash
# Setup separate etcd cluster first
# Then point K3s to it
curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint="https://etcd1:2379,https://etcd2:2379,https://etcd3:2379"
```

**For your project:** Use **embedded etcd** (Option 1) - simpler and sufficient.

---

### Q6: Control plane components chạy ở đâu trong K3s?

**Trả lời:** Control plane trong K3s **khác biệt** với K8s truyền thống:

**Traditional Kubernetes:**
```bash
# Control plane runs as pods in kube-system namespace
kubectl get pods -n kube-system
# kube-apiserver-master1
# kube-scheduler-master1
# kube-controller-manager-master1
# etcd-master1
```

**K3s:**
```bash
# Control plane runs as a SINGLE PROCESS
ps aux | grep k3s
# /usr/local/bin/k3s server

# This single binary includes:
# - API server
# - Scheduler
# - Controller manager
# - Embedded etcd (if --cluster-init)
# - Kubelet
# - Kube-proxy (if not disabled)

# You won't see separate pods for control plane
kubectl get pods -n kube-system
# coredns-xxx
# local-path-provisioner-xxx
# metrics-server-xxx
# traefik-xxx
# (No kube-apiserver, kube-scheduler pods!)
```

**Benefits:**
- Lighter weight (~40MB binary)
- Faster startup
- Less memory usage
- Easier to manage
- Still 100% K8s compatible API

---

## 🔧 Setup & Configuration

### Q7: Làm thế nào để add thêm nodes sau này?

**Add server node (control plane + workloads):**
```bash
# Get token from existing server
sudo cat /var/lib/rancher/k3s/server/node-token

# On new node
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://existing-server:6443 \
  --token <TOKEN>
```

**Add agent node (workloads only):**
```bash
# Get token from existing server
sudo cat /var/lib/rancher/k3s/server/node-token

# On new node
curl -sfL https://get.k3s.io | K3S_URL=https://existing-server:6443 K3S_TOKEN=<TOKEN> sh -
```

---

### Q8: Làm thế nào để remove node khỏi cluster?

```bash
# On the node to remove
sudo /usr/local/bin/k3s-uninstall.sh        # For server nodes
# or
sudo /usr/local/bin/k3s-agent-uninstall.sh  # For agent nodes

# From any other node
kubectl delete node <node-name>

# If node is in NotReady state and won't delete
kubectl delete node <node-name> --force --grace-period=0
```

---

### Q9: K3s config file ở đâu?

```bash
# K3s config (if used)
/etc/rancher/k3s/config.yaml

# Kubeconfig
/etc/rancher/k3s/k3s.yaml

# Copy to standard location
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Data directory
/var/lib/rancher/k3s/

# Embedded etcd data
/var/lib/rancher/k3s/server/db/etcd/
```

---

### Q10: Làm sao restart K3s service?

```bash
# On server nodes
sudo systemctl restart k3s

# On agent nodes
sudo systemctl restart k3s-agent

# Check status
sudo systemctl status k3s
sudo journalctl -u k3s -f

# Check logs
sudo journalctl -u k3s --since "10 minutes ago"
```

---

## 🚀 Operations

### Q11: Làm sao upgrade K3s?

```bash
# Check current version
k3s --version

# Upgrade to latest stable
curl -sfL https://get.k3s.io | sh -s - server

# Or specific version
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.28.5+k3s1 sh -s - server

# Upgrade all nodes one by one
# Server nodes first, then agent nodes
```

**Best practice:**
```bash
# 1. Backup first
sudo cp -r /var/lib/rancher/k3s /backup/k3s-backup-$(date +%Y%m%d)

# 2. Upgrade one server at a time
# Server 1 → wait for healthy → Server 2 → wait → Server 3

# 3. Check cluster health between each upgrade
kubectl get nodes
kubectl get pods --all-namespaces
```

---

### Q12: Làm sao check cluster health?

```bash
# Node status
kubectl get nodes

# All pods
kubectl get pods --all-namespaces

# Component status (K3s doesn't use this, but cluster should work)
kubectl get --raw /healthz

# etcd health (if embedded)
sudo k3s etcd-snapshot save --name health-check
sudo k3s etcd-snapshot ls

# API server health
curl -k https://localhost:6443/healthz

# Check for problems
kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

---

### Q13: Làm sao backup K3s cluster?

**Backup etcd:**
```bash
# Create snapshot
sudo k3s etcd-snapshot save --name backup-$(date +%Y%m%d-%H%M%S)

# List snapshots
sudo k3s etcd-snapshot ls

# Snapshots saved to:
ls -lah /var/lib/rancher/k3s/server/db/snapshots/
```

**Restore from snapshot:**
```bash
# Stop K3s on ALL server nodes
sudo systemctl stop k3s

# On node 1, restore snapshot
sudo k3s server --cluster-reset --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/backup-xxx

# Restart K3s
sudo systemctl start k3s

# On other nodes, delete data and rejoin
sudo rm -rf /var/lib/rancher/k3s/server/db
sudo systemctl start k3s
```

**Backup entire cluster (better):**
```bash
# Use Velero (recommended)
# See K3S_SETUP_GUIDE.md Part 10
```

---

### Q14: Có thể chạy K3s với external database (PostgreSQL, MySQL)?

**Yes!** Thay vì embedded etcd:

```bash
# Install K3s with PostgreSQL backend
curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint="postgres://username:password@hostname:5432/k3s"

# Or MySQL
curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint="mysql://username:password@tcp(hostname:3306)/k3s"
```

**When to use:**
- Managed database available (RDS, Cloud SQL)
- Don't want to manage etcd
- Need database-level backups

**Limitations:**
- Only 1 or 2 server nodes supported (not 3+)
- External database becomes single point of failure
- Need to secure database separately

**Recommendation:** Use **embedded etcd** for HA (3+ servers) ✅

---

## 💡 Best Practices

### Q15: Resource requirements cho K3s?

**Minimum:**
- CPU: 1 core
- RAM: 512MB
- Disk: 10GB

**Recommended for production:**

**Server nodes:**
- CPU: 2-4 cores
- RAM: 4-8GB
- Disk: 50-100GB SSD

**Agent nodes:**
- CPU: 2-4 cores (depending on workloads)
- RAM: 4-16GB (depending on workloads)
- Disk: 50GB+

**Your setup (3 server nodes):**
- 8 vCPU, 16GB RAM each ✅ Perfect!
- Can handle 50+ customers easily

---

### Q16: Ports cần mở cho K3s?

**Server nodes:**
```bash
# Kubernetes API
6443/tcp

# etcd (if external, between servers)
2379-2380/tcp

# Kubelet metrics
10250/tcp

# Ingress (Traefik)
80/tcp
443/tcp

# SSH
22/tcp
```

**UFW Example:**
```bash
sudo ufw allow 22/tcp
sudo ufw allow 6443/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 10250/tcp
sudo ufw enable
```

**Between cluster nodes (private network):**
- Allow all traffic between cluster nodes
- Or specifically: 6443, 10250, 2379-2380, 8472 (flannel)

---

### Q17: K3s có production-ready không?

**Yes! 100% production-ready** ✅

**Used by:**
- SUSE (owner of Rancher)
- Multiple Fortune 500 companies
- Edge deployments (IoT, retail)
- Small-medium production clusters

**Proven at scale:**
- Hundreds of thousands of deployments
- 1-100+ node clusters
- Edge to datacenter
- 100% K8s API compatible

**For your European Cloud Platform:**
- Perfect fit for 3-10 node clusters
- Proven reliability
- Active development and support
- Large community

---

### Q18: K3s vs K8s - compatibility?

**100% Kubernetes API compatible** ✅

**What works:**
- All kubectl commands
- All Kubernetes resources (pods, deployments, services, etc.)
- Helm charts
- Kubernetes Operators
- CRDs (Custom Resource Definitions)
- All standard tools (ArgoCD, Prometheus, etc.)

**What's different:**
- Single binary instead of multiple components
- Different default components (Traefik instead of no ingress)
- Embedded etcd option
- Lighter weight (~40MB vs 1GB+)

**Bottom line:** If it works on K8s, it works on K3s ✅

---

### Q19: Có thể migrate từ Docker Swarm sang K3s không?

**Yes, but not automatic.** Need to:

1. **Deploy K3s cluster** (new infrastructure)
2. **Convert Docker Compose to K8s manifests:**
   ```bash
   # Use kompose tool
   kompose convert -f docker-compose.yml
   # Generates K8s YAML files
   ```
3. **Migrate data:**
   - Backup databases
   - Copy volumes/persistent data
   - Import into K3s PVCs
4. **Test thoroughly**
5. **Switch DNS/load balancer**
6. **Decommission Swarm cluster**

**Better approach:** Start new projects with K3s ✅

---

### Q20: K3s có cần Docker không?

**No!** K3s has **containerd built-in** ✅

**Container runtime:**
- K3s uses containerd (built-in)
- Docker not required
- Docker images work fine (OCI compatible)

**If you want to use Docker:**
```bash
# Install Docker first
curl -fsSL https://get.docker.com | sh

# Install K3s with Docker
curl -sfL https://get.k3s.io | sh -s - server --docker

# Check runtime
kubectl get nodes -o wide
# CONTAINER-RUNTIME column shows "docker://..." or "containerd://..."
```

**Recommendation:** Use default containerd (no Docker needed) ✅

---

## 🆘 Troubleshooting

### Q21: K3s service không start được?

```bash
# Check status
sudo systemctl status k3s

# Check logs
sudo journalctl -u k3s -n 100 --no-pager

# Common issues:
# 1. Port 6443 already in use
sudo lsof -i :6443

# 2. Previous K3s installation not cleaned
sudo /usr/local/bin/k3s-uninstall.sh
# Then reinstall

# 3. Not enough memory
free -h
# Need at least 512MB free

# 4. Firewall blocking
sudo ufw status
sudo ufw allow 6443/tcp
```

---

### Q22: Nodes hiển thị NotReady?

```bash
# Check node
kubectl describe node <node-name>

# Common causes:
# 1. Kubelet not running
sudo systemctl status k3s

# 2. Network issues
ping <other-node-ip>

# 3. Resource exhaustion
df -h  # Check disk space
free -h  # Check memory

# 4. etcd issues (servers only)
sudo k3s kubectl get endpoints -n kube-system k3s

# Fix: Restart K3s
sudo systemctl restart k3s
```

---

### Q23: Pods stuck in Pending?

```bash
# Check pod details
kubectl describe pod <pod-name> -n <namespace>

# Common causes:
# 1. Not enough resources
kubectl top nodes
kubectl describe node

# 2. No nodes match selector
kubectl get nodes --show-labels

# 3. PVC not bound
kubectl get pvc -A

# 4. ImagePullBackOff
kubectl describe pod <pod-name> | grep -A 10 Events
```

---

## 📚 Resources

- [K3s Official Documentation](https://docs.k3s.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [K3s GitHub](https://github.com/k3s-io/k3s)
- [Rancher Forums](https://forums.rancher.com/c/k3s/)
- [CNCF Slack #k3s](https://slack.cncf.io/)

---

**FAQ Version**: 1.0  
**Last Updated**: December 11, 2025  
**For**: European Cloud Platform Project
