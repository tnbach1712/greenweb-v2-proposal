# Kubernetes Taints and Tolerations - Giải thích chi tiết

## 🎯 Taint là gì?

**Taint** (dấu vết, nhãn cấm) là một cơ chế trong Kubernetes để **ngăn chặn pods được schedule (triển khai) lên một node cụ thể**, trừ khi pod đó có **toleration** (khả năng chịu đựng) phù hợp.

### Ví dụ đơn giản:

```
Giống như biển cảnh báo:
"⚠️ Cấm vào - Chỉ nhân viên được phép"

Node bị taint = Có biển cảnh báo
Pod không có toleration = Không được vào
Pod có toleration = Được phép vào
```

---

## 🔍 Taint trong thực tế

### Kubernetes truyền thống (kubeadm)

```bash
# Khi bạn khởi tạo master node
kubeadm init

# Master node TỰ ĐỘNG được taint:
kubectl describe node master1 | grep Taint

# Output:
Taints: node-role.kubernetes.io/master:NoSchedule
        node-role.kubernetes.io/control-plane:NoSchedule
```

**Ý nghĩa:**
- ❌ Pods thông thường KHÔNG thể chạy trên master nodes
- ✅ Chỉ system pods (có toleration) mới chạy được
- 🎯 Master nodes chỉ dành cho control plane

### K3s (default behavior)

```bash
# Khi bạn khởi tạo server node
curl -sfL https://get.k3s.io | sh -s - server --cluster-init

# Server node KHÔNG có taint:
kubectl describe node server1 | grep Taint

# Output:
Taints: <none>
```

**Ý nghĩa:**
- ✅ Pods thông thường CÓ THỂ chạy trên server nodes
- ✅ Server nodes vừa chạy control plane, vừa chạy workloads
- 🎯 Sử dụng hiệu quả tài nguyên

---

## 📊 So sánh trực quan

### Kubernetes truyền thống (Master bị taint)

```
┌─────────────────────────────────────────┐
│  Master Node (TAINTED)                  │
│  ⚠️ Taint: NoSchedule                   │
│                                         │
│  ✅ kube-apiserver                      │
│  ✅ kube-scheduler                      │
│  ✅ kube-controller                     │
│  ✅ etcd                                │
│                                         │
│  ❌ Your app pods (BLOCKED)            │
│  ❌ nginx (BLOCKED)                     │
│  ❌ database (BLOCKED)                  │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  Worker Node (NO TAINT)                 │
│  ✅ No restrictions                     │
│                                         │
│  ✅ Your app pods                       │
│  ✅ nginx                               │
│  ✅ database                            │
└─────────────────────────────────────────┘
```

### K3s (Server KHÔNG bị taint)

```
┌─────────────────────────────────────────┐
│  Server Node (NO TAINT)                 │
│  ✅ No restrictions                     │
│                                         │
│  ✅ API server (control plane)         │
│  ✅ Scheduler (control plane)          │
│  ✅ Controller (control plane)         │
│  ✅ etcd (control plane)               │
│                                         │
│  ✅ Your app pods (ALLOWED)            │
│  ✅ nginx (ALLOWED)                     │
│  ✅ database (ALLOWED)                  │
└─────────────────────────────────────────┘
```

---

## 🛠️ Cách sử dụng Taints

### 1. Xem taints của node

```bash
# Xem tất cả nodes và taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Hoặc describe một node cụ thể
kubectl describe node server1 | grep -A 5 Taints
```

### 2. Thêm taint vào node

```bash
# Syntax:
kubectl taint nodes <node-name> <key>=<value>:<effect>

# Ví dụ: Taint node để không chạy workloads
kubectl taint nodes server1 node-role.kubernetes.io/master=true:NoSchedule

# Giờ server1 sẽ TỪ CHỐI pods thông thường
```

**Effects (hiệu ứng):**

| Effect | Ý nghĩa |
|--------|---------|
| **NoSchedule** | Pods mới KHÔNG được schedule lên node này |
| **PreferNoSchedule** | Pods NÊN tránh schedule lên node này (không bắt buộc) |
| **NoExecute** | Pods đang chạy sẽ bị evict (đuổi ra), pods mới không được schedule |

### 3. Xóa taint khỏi node

```bash
# Thêm dấu trừ (-) ở cuối key
kubectl taint nodes server1 node-role.kubernetes.io/master-

# Bây giờ server1 lại cho phép pods chạy
```

### 4. Taint với nhiều effects

```bash
# NoSchedule: Không cho pods mới
kubectl taint nodes node1 dedicated=gpu:NoSchedule

# PreferNoSchedule: Ưu tiên không cho pods
kubectl taint nodes node2 maintenance=true:PreferNoSchedule

# NoExecute: Đuổi pods đang chạy
kubectl taint nodes node3 evict=true:NoExecute
```

---

## 🎭 Tolerations - Chìa khóa để vào node bị taint

Nếu node bị taint, pod cần có **toleration** để được phép chạy trên node đó.

### Ví dụ: Pod với toleration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-toleration
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
```

**Giải thích:**
- Pod này có "chìa khóa" (toleration)
- Có thể chạy trên node bị taint với key `node-role.kubernetes.io/master`
- Như nhân viên có thẻ từ được vào khu vực cấm

### Toleration operators

```yaml
# Operator: Equal (phải khớp chính xác)
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "nvidia"
  effect: "NoSchedule"

# Operator: Exists (chỉ cần key tồn tại)
tolerations:
- key: "node-role.kubernetes.io/master"
  operator: "Exists"
  effect: "NoSchedule"
```

---

## 🎯 Use Cases thực tế

### 1. Dedicated Nodes cho GPU workloads

```bash
# Taint GPU nodes
kubectl taint nodes gpu-node-1 gpu=nvidia:NoSchedule

# Chỉ ML/AI pods (có toleration) mới chạy trên GPU nodes
```

```yaml
# ML workload with GPU toleration
apiVersion: v1
kind: Pod
metadata:
  name: pytorch-training
spec:
  containers:
  - name: pytorch
    image: pytorch/pytorch:latest
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "nvidia"
    effect: "NoSchedule"
  nodeSelector:
    gpu: "nvidia"
```

### 2. Maintenance Mode

```bash
# Đưa node vào maintenance (đuổi pods ra)
kubectl taint nodes node1 maintenance=true:NoExecute

# Pods sẽ tự động di chuyển sang nodes khác
# Node1 không nhận pods mới

# Sau khi maintenance xong, xóa taint
kubectl taint nodes node1 maintenance-
```

### 3. Reserved Nodes cho Critical Apps

```bash
# Taint nodes cho production
kubectl taint nodes prod-node-1 environment=production:NoSchedule

# Chỉ production pods được chạy
```

```yaml
# Production app
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  template:
    spec:
      tolerations:
      - key: "environment"
        operator: "Equal"
        value: "production"
        effect: "NoSchedule"
```

---

## 🔬 Demo thực tế

### Scenario: Tạo dedicated node cho databases

```bash
# Step 1: Taint node cho databases only
kubectl taint nodes server3 workload=database:NoSchedule

# Step 2: Deploy nginx (NO toleration)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-no-toleration
spec:
  containers:
  - name: nginx
    image: nginx
EOF

# Step 3: Check where it runs
kubectl get pods -o wide
# nginx-no-toleration sẽ chạy trên server1 hoặc server2
# KHÔNG BAO GIỜ chạy trên server3 (bị taint)

# Step 4: Deploy database (WITH toleration)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
  - name: postgres
    image: postgres:15
  tolerations:
  - key: "workload"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
  nodeSelector:
    workload: database
EOF

# Step 5: Check where postgres runs
kubectl get pods -o wide
# postgres CÓ THỂ chạy trên server3
# Vì nó có toleration phù hợp
```

---

## 📋 Cheat Sheet

```bash
# Xem taints
kubectl get nodes -o json | jq '.items[].spec.taints'
kubectl describe node <node> | grep Taint

# Thêm taint
kubectl taint nodes <node> key=value:NoSchedule
kubectl taint nodes <node> key=value:PreferNoSchedule
kubectl taint nodes <node> key=value:NoExecute

# Xóa taint
kubectl taint nodes <node> key-

# Xóa taint cụ thể
kubectl taint nodes <node> key:NoSchedule-

# Xóa tất cả taints
kubectl patch node <node> -p '{"spec":{"taints":[]}}'
```

---

## 🎓 Best Practices

### 1. K3s với 3 nodes: KHÔNG nên taint

```bash
# ❌ Sai: Lãng phí tài nguyên
kubectl taint nodes server1 node-role.kubernetes.io/master:NoSchedule
kubectl taint nodes server2 node-role.kubernetes.io/master:NoSchedule
kubectl taint nodes server3 node-role.kubernetes.io/master:NoSchedule
# Giờ KHÔNG có node nào chạy workloads!

# ✅ Đúng: Để mặc định (no taint)
# Tất cả 3 server nodes đều chạy workloads
```

### 2. Khi nào nên dùng taints?

**✅ Nên dùng:**
- Dedicated GPU nodes
- Maintenance mode
- Special hardware (SSD, high-memory)
- Production vs staging isolation
- Có 10+ nodes

**❌ Không nên dùng:**
- Cluster nhỏ (3-5 nodes)
- Muốn sử dụng hiệu quả tài nguyên
- Không có yêu cầu đặc biệt

### 3. Test trước khi production

```bash
# Test 1: Taint một node
kubectl taint nodes test-node test=true:NoSchedule

# Test 2: Deploy pod without toleration
kubectl run test-nginx --image=nginx

# Test 3: Verify không chạy trên tainted node
kubectl get pods -o wide

# Test 4: Cleanup
kubectl delete pod test-nginx
kubectl taint nodes test-node test-
```

---

## 🆚 K3s vs K8s: Taint Comparison

| Aspect | K8s (kubeadm) | K3s |
|--------|---------------|-----|
| **Master taints** | ✅ Auto-tainted | ❌ No taints |
| **Master run workloads** | ❌ No | ✅ Yes |
| **Efficient for 3 nodes** | ❌ No | ✅ Yes |
| **Need manual taint** | ❌ No | ✅ Yes (if wanted) |

---

## 💡 Tóm tắt

**Taint = Biển cảnh báo trên node**
- Node có taint = Node có hạn chế
- Pod không có toleration = Không được vào
- Pod có toleration = Được phép vào

**K3s vs K8s:**
- K8s: Masters tự động bị taint → Không chạy workloads
- K3s: Servers KHÔNG bị taint → Chạy được workloads

**Cho European Cloud Platform:**
- 3 server nodes → KHÔNG nên taint
- Để mặc định (no taint) → Hiệu quả nhất
- Khi có 10+ nodes → Có thể cân nhắc taint

---

**Document Version**: 1.0  
**Last Updated**: December 11, 2025  
**Related**: K3S_FAQ.md, K3S_SETUP_GUIDE.md
