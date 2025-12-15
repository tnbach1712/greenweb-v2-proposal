# MetalLB Setup Guide

## Tổng quan

MetalLB là LoadBalancer cho Kubernetes trên bare-metal servers. Nó cấp IP addresses cho Services type LoadBalancer.

---

## Bước 1: Cài đặt MetalLB

### 1.1. Cài đặt qua manifest

```bash
# Cài đặt MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.0/config/manifests/metallb-native.yaml

# Kiểm tra pods đã chạy
kubectl get pods -n metallb-system

# Đợi pods ready (khoảng 30 giây)
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

### 1.2. Hoặc cài qua Helm (khuyên dùng)

```bash
# Thêm Helm repo
helm repo add metallb https://metallb.github.io/metallb
helm repo update

# Cài đặt MetalLB
helm install metallb metallb/metallb -n metallb-system --create-namespace

# Kiểm tra
helm list -n metallb-system
kubectl get pods -n metallb-system
```

---

## Bước 2: Cấu hình IP Address Pool

### 2.1. Layer 2 Mode (Đơn giản - Khuyên dùng cho bắt đầu)

Tạo file `metallb-config.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  # Chọn dải IP KHÔNG SỬ DỤNG trong mạng của bạn
  # Ví dụ: Mạng bạn là 192.168.1.0/24, router là 192.168.1.1
  # Bạn có thể dùng 192.168.1.200-192.168.1.250 (nếu không bị DHCP chiếm)
  - 192.168.1.200-192.168.1.210  # 10 IP addresses

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
```

**Cách chọn dải IP:**

1. **Kiểm tra mạng hiện tại:**
   ```bash
   # Xem IP của máy chủ
   ip addr show
   
   # Ví dụ output:
   # inet 192.168.1.50/24  ← Mạng là 192.168.1.0/24
   ```

2. **Chọn dải IP không dùng:**
   - Router thường là: `.1` (192.168.1.1)
   - DHCP thường cấp: `.100-.199` (192.168.1.100-192.168.1.199)
   - **Bạn chọn:** `.200-.210` (192.168.1.200-192.168.1.210) ✅

3. **Apply config:**
   ```bash
   kubectl apply -f metallb-config.yaml
   
   # Kiểm tra
   kubectl get ipaddresspools -n metallb-system
   kubectl get l2advertisements -n metallb-system
   ```

---

### 2.2. BGP Mode (Advanced - Cho production)

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: bgp-pool
  namespace: metallb-system
spec:
  addresses:
  - 203.0.113.10-203.0.113.20  # Public IPs từ ISP

---
apiVersion: metallb.io/v1beta1
kind: BGPPeer
metadata:
  name: router-peer
  namespace: metallb-system
spec:
  myASN: 64500         # ASN của bạn
  peerASN: 64501       # ASN của router
  peerAddress: 192.168.1.1  # IP của router

---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: bgp-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - bgp-pool
```

**Lưu ý:** BGP mode cần router hỗ trợ BGP (VD: Cisco, Juniper, MikroTik).

---

## Bước 3: Test MetalLB

### 3.1. Tạo test service

```yaml
# test-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer  # ← MetalLB sẽ cấp IP cho service này
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

### 3.2. Deploy và test

```bash
# Deploy
kubectl apply -f test-nginx.yaml

# Kiểm tra Service có IP chưa
kubectl get svc nginx-loadbalancer

# Output mong đợi:
# NAME                  TYPE           EXTERNAL-IP      PORT(S)
# nginx-loadbalancer    LoadBalancer   192.168.1.200    80:30123/TCP

# Test truy cập
curl http://192.168.1.200

# Nếu thấy "Welcome to nginx!" là thành công ✅
```

### 3.3. Xóa test

```bash
kubectl delete -f test-nginx.yaml
```

---

## Bước 4: Setup cho Traefik (Ingress Controller)

Trong dự án của bạn, Traefik sẽ dùng MetalLB để có IP public.

```yaml
# traefik-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik
  namespace: kube-system
spec:
  type: LoadBalancer  # ← MetalLB cấp IP
  selector:
    app: traefik
  ports:
  - name: web
    port: 80
    targetPort: 8000
  - name: websecure
    port: 443
    targetPort: 8443
```

**Sau khi deploy:**

```bash
# Xem IP của Traefik
kubectl get svc traefik -n kube-system

# Output:
# NAME      TYPE           EXTERNAL-IP      PORT(S)
# traefik   LoadBalancer   192.168.1.200    80:30080/TCP,443:30443/TCP

# Bây giờ bạn có thể trỏ domain đến 192.168.1.200
```

---

## Troubleshooting

### 1. Service stuck ở "Pending"

```bash
# Kiểm tra MetalLB pods
kubectl get pods -n metallb-system

# Xem logs
kubectl logs -n metallb-system deployment/controller
kubectl logs -n metallb-system daemonset/speaker

# Kiểm tra config
kubectl get ipaddresspools -n metallb-system
kubectl get l2advertisements -n metallb-system
```

**Nguyên nhân thường gặp:**
- ❌ Chưa tạo IPAddressPool
- ❌ Dải IP đã được dùng bởi DHCP/device khác
- ❌ MetalLB pods chưa ready

### 2. Không truy cập được IP

```bash
# Ping IP
ping 192.168.1.200

# Kiểm tra ARP
arp -a | grep 192.168.1.200

# Kiểm tra firewall
sudo iptables -L -n | grep 192.168.1.200
```

**Giải pháp:**
- Tắt firewall tạm để test: `sudo ufw disable`
- Kiểm tra router có block traffic không
- Thử IP khác trong pool

### 3. Conflict IP

Nếu IP bị conflict với device khác:

```bash
# Thay đổi dải IP trong metallb-config.yaml
# Ví dụ: từ 192.168.1.200-210 → 192.168.1.220-230

# Apply lại
kubectl apply -f metallb-config.yaml

# Xóa service và tạo lại để lấy IP mới
kubectl delete svc nginx-loadbalancer
kubectl apply -f test-nginx.yaml
```

---

## Best Practices

### 1. Production Setup

```yaml
# Dùng nhiều IP ranges
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.205  # Web services
  - 192.168.1.210-192.168.1.215  # Database services
  autoAssign: true  # Tự động assign IP
```

### 2. IP cố định cho service quan trọng

```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik
  annotations:
    metallb.universe.tf/loadBalancerIPs: 192.168.1.200  # IP cố định
spec:
  type: LoadBalancer
  # ...
```

### 3. Monitoring

```bash
# Xem IP đã được assign
kubectl get svc -A | grep LoadBalancer

# Xem events
kubectl get events -n metallb-system --sort-by='.lastTimestamp'
```

---

## So sánh Layer 2 vs BGP

| Feature | Layer 2 | BGP |
|---------|---------|-----|
| **Setup** | ✅ Đơn giản (5 phút) | ❌ Phức tạp (cần config router) |
| **Router requirement** | ✅ Không cần config | ❌ Cần router hỗ trợ BGP |
| **Load balancing** | ❌ Chỉ 1 node xử lý | ✅ Nhiều nodes |
| **Failover** | ⚠️ Chậm (10s) | ✅ Nhanh (<1s) |
| **Scalability** | ⚠️ Hạn chế | ✅ Tốt |
| **Use case** | Home lab, SMB | Enterprise, ISP |

---

## Tóm tắt

1. **Cài MetalLB:** `kubectl apply -f metallb-manifest.yaml`
2. **Tạo IP pool:** Chọn dải IP không dùng (VD: 192.168.1.200-210)
3. **Apply config:** `kubectl apply -f metallb-config.yaml`
4. **Test:** Tạo Service type LoadBalancer, kiểm tra có IP không
5. **Deploy Traefik:** Traefik sẽ tự động lấy IP từ MetalLB

**Khuyên dùng cho bạn:**
- ✅ **Layer 2 mode** (đơn giản, đủ cho SMB)
- ✅ Dải IP: 192.168.1.200-210 (10 IPs)
- ✅ Traefik sẽ lấy 1 IP cố định (VD: 192.168.1.200)

**Alternatives:**
- **Không dùng MetalLB:** Dùng NodePort + external load balancer (HAProxy, nginx)
- **Cloud:** Dùng cloud load balancer (AWS ALB, Azure LB)
- **Hybrid:** MetalLB cho internal, Cloudflare tunnel cho external
