---
title: Kubernetes HA Cluster Lab
categories:
- System
- Kubernetes
- Linux
- VMware
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Dựng Kubernetes HA Cluster với Cilium, Longhorn, MetalLB và NGINX Ingress
---

### 1. Giới thiệu

- Trong bài lab này mình triển khai một cụm Kubernetes HA chạy trên VMware ESXi với mô hình 3 control-plane và 3 worker node.
- Stack sử dụng theo hướng production-ready cho môi trường lab hoặc on-premise nhỏ: Cilium CNI, Longhorn storage, MetalLB, NGINX Ingress, Prometheus/Grafana và Loki/Promtail.
- Mục tiêu là có một cụm K8s vừa dễ quan sát, vừa có khả năng chịu lỗi khi mất 1 node control-plane hoặc 1 worker node.

<img src="/assets/img/2026-03-27-kubernetes-ha-cluster-lab/01.png" />

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Kiến trúc tổng thể](#2-kiến-trúc-tổng-thể)
- [3. Tài nguyên và thành phần sử dụng](#3-tài-nguyên-và-thành-phần-sử-dụng)
- [4. Chuẩn bị VM](#4-chuẩn-bị-vm)
  - [4.1. Tạo VM trên ESXi](#41-tạo-vm-trên-esxi)
  - [4.2. Cấu hình hostname và IP tĩnh](#42-cấu-hình-hostname-và-ip-tĩnh)
  - [4.3. Cấu hình chung cho tất cả node](#43-cấu-hình-chung-cho-tất-cả-node)
- [5. HA Control Plane với HAProxy và Keepalived](#5-ha-control-plane-với-haproxy-và-keepalived)
  - [5.1. Cài đặt HAProxy và Keepalived](#51-cài-đặt-haproxy-và-keepalived)
  - [5.2. Cấu hình HAProxy](#52-cấu-hình-haproxy)
  - [5.3. Cấu hình Keepalived](#53-cấu-hình-keepalived)
- [6. Khởi tạo Kubernetes Cluster](#6-khởi-tạo-kubernetes-cluster)
  - [6.1. Init cluster trên master đầu tiên](#61-init-cluster-trên-master-đầu-tiên)
  - [6.2. Join các master còn lại](#62-join-các-master-còn-lại)
  - [6.3. Join các worker node](#63-join-các-worker-node)
  - [6.4. Kiểm tra node](#64-kiểm-tra-node)
- [7. Cài Cilium CNI](#7-cài-cilium-cni)
  - [7.1. Cài Cilium CLI](#71-cài-cilium-cli)
  - [7.2. Cài Cilium](#72-cài-cilium)
  - [7.3. Kiểm tra trạng thái Cilium](#73-kiểm-tra-trạng-thái-cilium)
- [8. Cài Helm](#8-cài-helm)
- [9. Cài MetalLB](#9-cài-metallb)
  - [9.1. Cài đặt MetalLB](#91-cài-đặt-metallb)
  - [9.2. Tạo IP pool và L2 advertisement](#92-tạo-ip-pool-và-l2-advertisement)
  - [9.3. Kiểm tra MetalLB](#93-kiểm-tra-metallb)
- [10. Cài NGINX Ingress Controller](#10-cài-nginx-ingress-controller)
- [11. Cài Longhorn Storage](#11-cài-longhorn-storage)
  - [11.1. Chuẩn bị worker node](#111-chuẩn-bị-worker-node)
  - [11.2. Kiểm tra environment của Longhorn](#112-kiểm-tra-environment-của-longhorn)
  - [11.3. Cài Longhorn](#113-cài-longhorn)
  - [11.4. Đặt Longhorn làm default StorageClass](#114-đặt-longhorn-làm-default-storageclass)
  - [11.5. Bỏ storage reserved](#115-bỏ-storage-reserved)
  - [11.6. Expose nhanh Longhorn UI bằng LoadBalancer](#116-expose-nhanh-longhorn-ui-bằng-loadbalancer)
- [12. Cài Monitoring với kube-prometheus-stack](#12-cài-monitoring-với-kube-prometheus-stack)
- [13. Cài Logging với Loki và Promtail](#13-cài-logging-với-loki-và-promtail)
  - [13.1. Cài Loki](#131-cài-loki)
  - [13.2. Cài Promtail](#132-cài-promtail)
  - [13.3. Kết nối Loki vào Grafana](#133-kết-nối-loki-vào-grafana)
  - [13.4. Ví dụ query log](#134-ví-dụ-query-log)
- [14. Expose các UI qua Ingress](#14-expose-các-ui-qua-ingress)
  - [14.1. Tạo Ingress resources](#141-tạo-ingress-resources)
  - [14.2. Cấu hình hosts trên laptop](#142-cấu-hình-hosts-trên-laptop)
  - [14.3. Truy cập các dịch vụ](#143-truy-cập-các-dịch-vụ)
- [15. Remote vào cluster từ laptop](#15-remote-vào-cluster-từ-laptop)
  - [15.1. Copy kubeconfig](#151-copy-kubeconfig)
  - [15.2. Thêm hosts entry trên laptop](#152-thêm-hosts-entry-trên-laptop)
  - [15.3. Sử dụng trên WSL](#153-sử-dụng-trên-wsl)
  - [15.4. Sử dụng trên PowerShell](#154-sử-dụng-trên-powershell)
- [16. Kiểm tra toàn bộ hệ thống](#16-kiểm-tra-toàn-bộ-hệ-thống)
  - [16.1. Test deploy và ingress](#161-test-deploy-và-ingress)
  - [16.2. Test Longhorn PVC](#162-test-longhorn-pvc)
- [17. Troubleshooting](#17-troubleshooting)
  - [17.1. Lỗi x509 certificate khi remote](#171-lỗi-x509-certificate-khi-remote)
  - [17.2. Worker node trùng hostname](#172-worker-node-trùng-hostname)
  - [17.3. conntrack not found](#173-conntrack-not-found)
- [18. Ghi chú](#18-ghi-chú)

### 2. Kiến trúc tổng thể

```text
                     ┌──────────────────────────┐
                     │    VIP: 10.10.200.10    │
                     │  (HAProxy + Keepalived) │
                     └────────────┬────────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         │                        │                        │
┌────────▼─────────┐  ┌──────────▼────────┐  ┌────────────▼──────┐
│ k8s-master-01    │  │ k8s-master-02     │  │ k8s-master-03     │
│ 10.10.200.11     │  │ 10.10.200.12      │  │ 10.10.200.13      │
│ CP + etcd        │  │ CP + etcd         │  │ CP + etcd         │
└──────────────────┘  └───────────────────┘  └───────────────────┘

┌──────────────────┐  ┌───────────────────┐  ┌───────────────────┐
│ k8s-worker-01    │  │ k8s-worker-02     │  │ k8s-worker-03     │
│ 10.10.200.14     │  │ 10.10.200.15      │  │ 10.10.200.16      │
│ Workload         │  │ Workload          │  │ Workload          │
└──────────────────┘  └───────────────────┘  └───────────────────┘

MetalLB Pool: 10.10.200.100 - 10.10.200.120
Ingress IP:   10.10.200.100 → *.lab.local
```

### 3. Tài nguyên và thành phần sử dụng

| Thành phần | Chi tiết | HA |
|---|---|---|
| OS | Ubuntu Server 24.04 LTS | — |
| Container Runtime | containerd | — |
| Kubernetes | v1.31.x | 3 CP + 3 Worker |
| CNI | Cilium | DaemonSet |
| HA Control Plane | HAProxy + Keepalived | 3 master |
| Load Balancer | MetalLB L2 mode | speaker DaemonSet |
| Ingress | NGINX Ingress Controller | 2 replicas |
| Storage | Longhorn | replica=2 |
| Monitoring | kube-prometheus-stack | Prometheus x2, Grafana x2, Alertmanager x2 |
| Logging | Loki + Promtail | Loki x1, Promtail DaemonSet |

| Hostname | IP | vCPU | RAM | Disk OS | Disk Data | Role |
|---|---|---|---|---|---|---|
| k8s-master-01 | 10.10.200.11 | 4 | 4 GB | 50 GB | — | Control Plane + etcd |
| k8s-master-02 | 10.10.200.12 | 4 | 4 GB | 50 GB | — | Control Plane + etcd |
| k8s-master-03 | 10.10.200.13 | 4 | 4 GB | 50 GB | — | Control Plane + etcd |
| k8s-worker-01 | 10.10.200.14 | 4 | 8 GB | 50 GB | 100 GB `/dev/sdb` | Worker |
| k8s-worker-02 | 10.10.200.15 | 4 | 8 GB | 50 GB | 100 GB `/dev/sdb` | Worker |
| k8s-worker-03 | 10.10.200.16 | 4 | 8 GB | 50 GB | 100 GB `/dev/sdb` | Worker |

- VIP dùng cho kube-apiserver: `10.10.200.10`
- Pool IP của MetalLB: `10.10.200.100 - 10.10.200.120`
- Mỗi worker node được gắn thêm 1 disk riêng cho Longhorn.

### 4. Chuẩn bị VM

#### 4.1. Tạo VM trên ESXi

- Tạo 6 VM theo bảng tài nguyên ở trên.
- Cài Ubuntu Server 24.04 LTS cho toàn bộ các node.

#### 4.2. Cấu hình hostname và IP tĩnh

```bash
# Đặt hostname
sudo hostnamectl set-hostname <hostname>

# Cấu hình IP tĩnh
sudo vi /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens160:
      dhcp4: false
      addresses:
        - 10.10.200.XX/24
      routes:
        - to: default
          via: 10.10.200.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

```bash
sudo netplan apply
```

#### 4.3. Cấu hình chung cho tất cả node

```bash
# Cập nhật hệ thống
sudo apt update && sudo apt upgrade -y

# /etc/hosts
cat <<EOF | sudo tee -a /etc/hosts
10.10.200.10  k8s-api
10.10.200.11  k8s-master-01
10.10.200.12  k8s-master-02
10.10.200.13  k8s-master-03
10.10.200.14  k8s-worker-01
10.10.200.15  k8s-worker-02
10.10.200.16  k8s-worker-03
EOF

# Tắt swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Package cần thiết
sudo apt install -y conntrack socat

# containerd
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# kubeadm, kubelet, kubectl
sudo apt install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

### 5. HA Control Plane với HAProxy và Keepalived

> Chạy trên cả 3 master node.

#### 5.1. Cài đặt HAProxy và Keepalived

```bash
sudo apt install -y haproxy keepalived
```

#### 5.2. Cấu hình HAProxy

```bash
sudo tee /etc/haproxy/haproxy.cfg <<'EOF'
global
    log /dev/log local0
    maxconn 2048
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend k8s-api
    bind *:8443
    default_backend k8s-masters

backend k8s-masters
    balance roundrobin
    option tcp-check
    server k8s-master-01 10.10.200.11:6443 check fall 3 rise 2
    server k8s-master-02 10.10.200.12:6443 check fall 3 rise 2
    server k8s-master-03 10.10.200.13:6443 check fall 3 rise 2
EOF

sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

#### 5.3. Cấu hình Keepalived

`k8s-master-01`:

```bash
sudo tee /etc/keepalived/keepalived.conf <<'EOF'
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight -2
    fall 3
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens160
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass K8sHA@2024
    }
    virtual_ipaddress {
        10.10.200.10/24
    }
    track_script {
        chk_haproxy
    }
}
EOF
```

`k8s-master-02` và `k8s-master-03` dùng cấu hình tương tự, chỉ đổi `state BACKUP` và `priority` tương ứng `99`, `98`.

```bash
sudo systemctl restart keepalived
sudo systemctl enable keepalived

# Verify VIP
ip addr show ens160 | grep 10.10.200.10
```

### 6. Khởi tạo Kubernetes Cluster

#### 6.1. Init cluster trên master đầu tiên

```bash
sudo kubeadm init \
  --control-plane-endpoint "k8s-api:8443" \
  --upload-certs \
  --apiserver-cert-extra-sans=10.10.200.10,10.10.200.11,10.10.200.12,10.10.200.13,k8s-api \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --skip-phases=addon/kube-proxy
```

- Dùng `--skip-phases=addon/kube-proxy` vì Cilium sẽ thay thế kube-proxy.
- Cần lưu lại output của `kubeadm init` để join master và worker.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 6.2. Join các master còn lại

```bash
sudo kubeadm join k8s-api:8443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <cert-key>
```

#### 6.3. Join các worker node

```bash
sudo kubeadm join k8s-api:8443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

#### 6.4. Kiểm tra node

```bash
kubectl get nodes -o wide
```

### 7. Cài Cilium CNI

#### 7.1. Cài Cilium CLI

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}
```

#### 7.2. Cài Cilium

```bash
cilium install \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=k8s-api \
  --set k8sServicePort=8443
```

#### 7.3. Kiểm tra trạng thái Cilium

```bash
cilium status --wait
kubectl get nodes -o wide
```

<img src="/assets/img/2026-03-27-kubernetes-ha-cluster-lab/01.png" />

### 8. Cài Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 9. Cài MetalLB

#### 9.1. Cài đặt MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml

kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=120s
```

#### 9.2. Tạo IP pool và L2 advertisement

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.10.200.100-10.10.200.120
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
EOF
```

#### 9.3. Kiểm tra MetalLB

```bash
kubectl get pods -n metallb-system -o wide
kubectl get ipaddresspool -n metallb-system
```

<img src="/assets/img/2026-03-27-kubernetes-ha-cluster-lab/02.png" />

### 10. Cài NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."metallb\.universe\.tf/address-pool"=default-pool
```

```bash
kubectl get svc -n ingress-nginx
kubectl get pods -n ingress-nginx -o wide
```

<img src="/assets/img/2026-03-27-kubernetes-ha-cluster-lab/03.png" />

### 11. Cài Longhorn Storage

#### 11.1. Chuẩn bị worker node

```bash
sudo apt install -y open-iscsi nfs-common util-linux
sudo systemctl enable --now iscsid

sudo modprobe iscsi_tcp
echo "iscsi_tcp" | sudo tee /etc/modules-load.d/iscsi.conf
sudo systemctl restart iscsid

sudo systemctl disable --now multipathd
cat <<EOF | sudo tee /etc/multipath.conf
blacklist {
    devnode "^sd[a-z0-9]+"
}
EOF

echo "- - -" | sudo tee /sys/class/scsi_host/host*/scan
lsblk

sudo mkfs.ext4 /dev/sdb
sudo mkdir -p /mnt/longhorn
sudo mount /dev/sdb /mnt/longhorn
echo '/dev/sdb /mnt/longhorn ext4 defaults 0 0' | sudo tee -a /etc/fstab
```

#### 11.2. Kiểm tra environment của Longhorn

```bash
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.7.2/scripts/environment_check.sh | bash
```

<img src="/assets/img/2026-03-27-kubernetes-ha-cluster-lab/04.png" />

#### 11.3. Cài Longhorn

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --set defaultSettings.defaultDataPath="/mnt/longhorn" \
  --set defaultSettings.replicaCount=2
```

#### 11.4. Đặt Longhorn làm default StorageClass

```bash
kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}' 2>/dev/null || true
kubectl patch storageclass longhorn -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

#### 11.5. Bỏ storage reserved

```bash
kubectl -n longhorn-system get nodes.longhorn.io -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.disks}{"\n"}{end}'

for NODE in k8s-worker-01 k8s-worker-02 k8s-worker-03; do
  kubectl -n longhorn-system patch nodes.longhorn.io $NODE --type merge \
    -p "{\"spec\":{\"disks\":{\"DISK_KEY\":{\"storageReserved\":0}}}}"
done
```

#### 11.6. Expose nhanh Longhorn UI bằng LoadBalancer

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: longhorn-frontend-lb
  namespace: longhorn-system
  annotations:
    metallb.universe.tf/address-pool: default-pool
spec:
  type: LoadBalancer
  selector:
    app: longhorn-ui
  ports:
    - name: http
      port: 80
      targetPort: 8000
      protocol: TCP
EOF

kubectl get svc -n longhorn-system longhorn-frontend-lb
```

<img src="/assets/img/2026-03-27-kubernetes-ha-cluster-lab/05.png" />

### 12. Cài Monitoring với kube-prometheus-stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword="admin" \
  --set grafana.replicas=2 \
  --set prometheus.prometheusSpec.replicas=2 \
  --set prometheus.prometheusSpec.podAntiAffinity="hard" \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=longhorn \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=10Gi \
  --set alertmanager.alertmanagerSpec.replicas=2 \
  --set alertmanager.alertmanagerSpec.podAntiAffinity="hard" \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.storageClassName=longhorn \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=5Gi
```

- Grafana mặc định đăng nhập với tài khoản `admin` và password `admin`.
- Nên đổi password ngay sau khi đăng nhập lần đầu.

### 13. Cài Logging với Loki và Promtail

#### 13.1. Cài Loki

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki \
  --namespace monitoring \
  --set deploymentMode=SingleBinary \
  --set loki.auth_enabled=false \
  --set singleBinary.replicas=1 \
  --set singleBinary.persistence.storageClass=longhorn \
  --set singleBinary.persistence.size=10Gi \
  --set loki.storage.type=filesystem \
  --set loki.commonConfig.replication_factor=1 \
  --set chunksCache.enabled=false \
  --set resultsCache.enabled=false \
  --set minio.enabled=false \
  --set gateway.enabled=false \
  --set read.replicas=0 \
  --set write.replicas=0 \
  --set backend.replicas=0 \
  --set loki.schemaConfig.configs[0].from="2024-01-01" \
  --set loki.schemaConfig.configs[0].store=tsdb \
  --set loki.schemaConfig.configs[0].object_store=filesystem \
  --set loki.schemaConfig.configs[0].schema=v13 \
  --set loki.schemaConfig.configs[0].index.prefix=index_ \
  --set loki.schemaConfig.configs[0].index.period=24h
```

#### 13.2. Cài Promtail

```bash
helm install promtail grafana/promtail \
  --namespace monitoring \
  --set config.clients[0].url=http://loki:3100/loki/api/v1/push
```

#### 13.3. Kết nối Loki vào Grafana

1. Vào Grafana.
2. Chọn Connections.
3. Chọn Data Sources.
4. Add data source và chọn Loki.
5. URL: `http://loki:3100`
6. Save & Test.

#### 13.4. Ví dụ query log

```logql
{namespace="default"}
{namespace="kube-system"} |= "error"
{namespace="monitoring", pod=~"prometheus.*"}
```

### 14. Expose các UI qua Ingress

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

#### 14.1. Tạo Ingress resources

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
spec:
  ingressClassName: nginx
  rules:
    - host: longhorn.lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: longhorn-frontend
                port:
                  number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
    - host: grafana.lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kube-prometheus-grafana
                port:
                  number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
    - host: prometheus.lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kube-prometheus-kube-prome-prometheus
                port:
                  number: 9090
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: alertmanager-ingress
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
    - host: alertmanager.lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kube-prometheus-kube-prome-alertmanager
                port:
                  number: 9093
EOF
```

#### 14.2. Cấu hình hosts trên laptop

```text
<INGRESS_IP>  longhorn.lab.local grafana.lab.local prometheus.lab.local alertmanager.lab.local
```

#### 14.3. Truy cập các dịch vụ

| Dịch vụ | URL | Credentials |
|---|---|---|
| Longhorn UI | http://longhorn.lab.local | — |
| Grafana | http://grafana.lab.local | admin / admin |
| Prometheus | http://prometheus.lab.local | — |
| Alertmanager | http://alertmanager.lab.local | — |

### 15. Remote vào cluster từ laptop

#### 15.1. Copy kubeconfig

```bash
cat ~/.kube/config
```

Lưu thành file ví dụ `k8s-cluster-lab.config`.

#### 15.2. Thêm hosts entry trên laptop

```text
10.10.200.10  k8s-api
```

#### 15.3. Sử dụng trên WSL

```bash
export KUBECONFIG=/mnt/d/k8s/k8s-cluster-lab.config
echo 'export KUBECONFIG=/mnt/d/k8s/k8s-cluster-lab.config' >> ~/.bashrc
kubectl get nodes -o wide
```

#### 15.4. Sử dụng trên PowerShell

```powershell
$env:KUBECONFIG = "D:\k8s\k8s-cluster-lab.config"
kubectl get nodes -o wide
```

### 16. Kiểm tra toàn bộ hệ thống

```bash
# Nodes
kubectl get nodes -o wide

# Pods hệ thống
kubectl get pods -A

# Services
kubectl get svc -A | grep LoadBalancer

# Ingress
kubectl get ingress -A
```

#### 16.1. Test deploy và ingress

```bash
kubectl create deployment nginx-test --image=nginx --replicas=2
kubectl expose deployment nginx-test --port=80 --type=ClusterIP

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-test-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: test.lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-test
                port:
                  number: 80
EOF

kubectl delete ingress nginx-test-ingress
kubectl delete svc nginx-test
kubectl delete deployment nginx-test
```

#### 16.2. Test Longhorn PVC

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test
      image: busybox
      command: ["sh", "-c", "echo 'Longhorn works!' > /data/test.txt && cat /data/test.txt && sleep 3600"]
      volumeMounts:
        - name: vol
          mountPath: /data
  volumes:
    - name: vol
      persistentVolumeClaim:
        claimName: test-pvc
EOF

kubectl logs test-pod
kubectl delete pod test-pod
kubectl delete pvc test-pvc
```

### 17. Troubleshooting

#### 17.1. Lỗi x509 certificate khi remote

```text
x509: certificate is valid for 10.96.0.1, 10.10.200.11, not 10.10.200.10
```

```bash
sudo mv /etc/kubernetes/pki/apiserver.{crt,key} /tmp/
sudo kubeadm init phase certs apiserver \
  --control-plane-endpoint "k8s-api:8443" \
  --apiserver-cert-extra-sans=10.10.200.10,10.10.200.11,10.10.200.12,10.10.200.13,k8s-api
sudo crictl pods --name kube-apiserver -q | xargs sudo crictl stopp
```

#### 17.2. Worker node trùng hostname

```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force
kubectl delete node <node>

sudo hostnamectl set-hostname <hostname-dung>
sudo kubeadm reset -f && sudo rm -rf /etc/cni/net.d
sudo kubeadm join k8s-api:8443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

#### 17.3. conntrack not found

```bash
sudo apt install -y conntrack
```

### 18. Ghi chú

- Mô hình 3 master giúp control plane vẫn hoạt động khi mất 1 node.
- Cilium thay kube-proxy bằng eBPF nên nhẹ và hiện đại hơn cho lab K8s mới.
- MetalLB L2 phù hợp lab hoặc on-premise, không cần BGP router.
- NGINX Ingress cho phép gom toàn bộ UI về một đầu ingress duy nhất theo host-based routing.
- Longhorn phù hợp lab storage phân tán, đặc biệt khi mỗi worker có disk riêng.
- Với production thực tế, phần logging nên cân nhắc Loki SimpleScalable hoặc object storage backend thay vì SingleBinary.