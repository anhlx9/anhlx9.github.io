---
title: "Next-Gen GPU Cloud: Hội Tụ VM & Container Trên Kubernetes"
categories:
- Kubernetes
- Ceph
- Cloud
tags:
- rke2
- kubevirt
- ceph
- capsule
- cilium
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Kubernetes-native Next-Gen GPU Cloud: RKE2 HA, KubeVirt, Rook-Ceph, multi-tenant GPU provisioning với 3 khách hàng — CPU-only, whole-GPU VM Windows, shared-GPU container
---

GPU Cloud thế hệ mới bán **compute linh hoạt theo nhu cầu**: nguyên GPU cho AI training nặng, chia nhỏ GPU cho inference, không GPU cho workload thường — tất cả trên một control plane, đồng thời bán cả VM lẫn container với quota riêng từng tenant.

Mình build lab này để PoC toàn bộ pattern đó trên 3 VM Ubuntu 24.04, kiến trúc converged all-in-one: control-plane, compute, storage và GPU provisioning gom lên cùng 3 node. Mỗi node phục vụ một loại khách hàng — **ctrl01** (CPU-only), **ctrl02** (whole-GPU), **ctrl03** (fractional GPU time-slicing) — đúng cách GPU cloud thương mại tận dụng phần cứng đắt tiền. Production sẽ tách tier riêng, dùng MIG thay time-slicing và Cilium BGP thay L2 Announcements.

Stack:

1. **RKE2** — Kubernetes HA, CIS-hardened, etcd quorum 3 node
2. **Cilium** — CNI eBPF, L2 Announcements thay MetalLB
3. **Rook-Ceph** — distributed storage, RBD block-mode RWX cho VM live migration
4. **KubeVirt + CDI** — chạy VM (KVM) như K8s workload, import Ubuntu/Windows image vào PVC
5. **fake-gpu-operator** — giả lập GPU whole + time-sliced (lab; production: NVIDIA GPU Operator)
6. **Capsule** — multi-tenancy hard isolation, custom Role per tenant, không cluster-admin
7. **kube-vip** — VIP Kubernetes API HA

> **Phạm vi bài**
> - Do nội dung quá dài, mình lược bỏ một số thành phần — Kueue, Cilium Gateway API, Prometheus/Grafana, VolumeSnapshot,... — để tập trung vào **demo business flow**: cấp phát GPU linh hoạt bán cho 3 loại khách hàng, đúng mô hình GPU cloud thương mại trong thời đại AI
> - Stack công nghệ là những thứ mình đang quan tâm; kiến trúc và phân bổ resource theo hạ tầng mình có. **Nếu áp dụng, bạn cần điều chỉnh spec VM, IP range, số node và các thành phần cho phù hợp với môi trường của mình**

<img src="../assets/img/2026-05-27-next-gen-gpu-cloud-rke2-kubevirt/gpu-cloud-architecture.svg"/>

---

## Mục lục

- [Mục lục](#mục-lục)
- [1. Tại sao Kubernetes cho Next-Gen GPU Cloud](#1-tại-sao-kubernetes-cho-next-gen-gpu-cloud)
  - [1.1 OpenStack vs Kubernetes-native](#11-openstack-vs-kubernetes-native)
  - [1.2 Topology 3 nodes trong Lab — phân bổ theo loại khách hàng](#12-topology-3-nodes-trong-lab--phân-bổ-theo-loại-khách-hàng)
- [2. Chuẩn bị Lab](#2-chuẩn-bị-lab)
- [3. Cài RKE2 HA Cluster](#3-cài-rke2-ha-cluster)
  - [3.1 Bootstrap node đầu tiên (ctrl01)](#31-bootstrap-node-đầu-tiên-ctrl01)
  - [3.2 Join ctrl02, ctrl03](#32-join-ctrl02-ctrl03)
  - [3.3 Verify cluster HA](#33-verify-cluster-ha)
- [4. kubectl, Helm, Cilium CLI](#4-kubectl-helm-cilium-cli)
- [5. Cilium L2 Announcements — LoadBalancer cho bare metal (thay MetalLB)](#5-cilium-l2-announcements--loadbalancer-cho-bare-metal-thay-metallb)
- [6. Rook-Ceph Distributed Storage](#6-rook-ceph-distributed-storage)
  - [6.1 Cài Rook Operator](#61-cài-rook-operator)
  - [6.2 Tạo CephCluster](#62-tạo-cephcluster)
  - [6.3 Tạo Pools + StorageClass (replication=3, RBD-RWX, CephFS)](#63-tạo-pools--storageclass-replication3-rbd-rwx-cephfs)
  - [6.4 Ceph Dashboard](#64-ceph-dashboard)
- [7. KubeVirt — chạy VM trên Kubernetes](#7-kubevirt--chạy-vm-trên-kubernetes)
- [8. KubeVirt Manager — Admin Portal](#8-kubevirt-manager--admin-portal)
- [9. fake-gpu-operator — Topology theo node](#9-fake-gpu-operator--topology-theo-node)
- [10. Capsule — Multi-tenancy với custom Role](#10-capsule--multi-tenancy-với-custom-role)
  - [10.1 Cài Capsule](#101-cài-capsule)
  - [10.2 Tạo custom ClusterRole `tenant-owner`](#102-tạo-custom-clusterrole-tenant-owner)
  - [10.3 Helper script — sinh kubeconfig tenant](#103-helper-script--sinh-kubeconfig-tenant)
- [11. Golden Image Templates](#11-golden-image-templates)
  - [11.1 Ubuntu 22.04 — VM golden image](#111-ubuntu-2204--vm-golden-image)
  - [11.2 Windows Server 2022 — VM golden image](#112-windows-server-2022--vm-golden-image)
  - [11.3 Container template — CUDA workload (KH3)](#113-container-template--cuda-workload-kh3)
- [12. Demo: Bán cho 3 khách hàng khác nhau](#12-demo-bán-cho-3-khách-hàng-khác-nhau)
  - [12.1 Tổng quan 3 khách hàng](#121-tổng-quan-3-khách-hàng)
  - [12.2 Admin chuẩn bị Tenants](#122-admin-chuẩn-bị-tenants)
  - [12.3 KH1 — cust-cpu mua VM Ubuntu 22.04 (không GPU)](#123-kh1--cust-cpu-mua-vm-ubuntu-2204-không-gpu)
  - [12.4 KH2 — cust-gpu-whole mua VM Windows Server 2022 + 1 GPU nguyên](#124-kh2--cust-gpu-whole-mua-vm-windows-server-2022--1-gpu-nguyên)
  - [12.5 KH3 — cust-gpu-shared mua 1 container + GPU chia nhỏ](#125-kh3--cust-gpu-shared-mua-1-container--gpu-chia-nhỏ)
  - [12.6 Verify cross-tenant isolation](#126-verify-cross-tenant-isolation)
- [13. Service URLs \& Credentials](#13-service-urls--credentials)
- [14. Lời kết](#14-lời-kết)

---

## 1. Tại sao Kubernetes cho Next-Gen GPU Cloud

### 1.1 OpenStack vs Kubernetes-native

OpenStack sinh ra trong kỷ nguyên VM-centric (2010–2018): GPU passthrough qua Nova hạn chế khi scale, còn muốn cung cấp song song VM lẫn container thì operator phải duy trì hai control plane riêng biệt — Nova cho VM, Zun (deprecated 2024) cho container.

Kubernetes giải quyết bằng một nền tảng duy nhất: KubeVirt chạy VM như K8s workload, chia sẻ cùng scheduler, storage và network với container. Khi AI workload cần fractional GPU, per-tenant GPU quota, live-migration — **OpenStack lùi về phân khúc private cloud thuần VM**, còn với **bài toán phân phối GPU**, **Kubernetes là lựa chọn duy nhất hợp lý**.

| Yêu cầu AI workload | OpenStack | Kubernetes-native |
|---|---|---|
| **Cấp phát GPU cho container** | Cần Magnum + Zun (deprecated), hoặc Nova GPU PCI | Built-in qua Device Plugin framework, GPU Operator |
| **Fractional GPU (time-slice, MIG)** | Không hỗ trợ native | GPU Operator + time-slicing ConfigMap, MIG, DRA |
| **Multi-tenant GPU quota** | Nova quota cho cores/RAM, không có cho GPU | ResourceQuota cho `nvidia.com/gpu` (Capsule enforce per-tenant) |
| **VM + Container cùng platform** | Chỉ VM (Nova). Container qua Zun (deprecated 2024) | KubeVirt cho VM + Pod cho container, cùng K8s API |
| **GitOps / IaC** | Heat templates, complex | Helm + ArgoCD/Flux, mọi config là YAML |
| **AI ecosystem** | Không có native integration | Kubeflow, KServe, Ray, vLLM, NIM — all native K8s |
| **GPU scheduling/queueing** | Không có | Kueue, Volcano, KAI Scheduler |

### 1.2 Topology 3 nodes trong Lab — phân bổ theo loại khách hàng

GPU node (A100/H100) đắt hơn CPU node 10–20×. Cloud provider không đặt CPU-only workload lên GPU node — lãng phí tài nguyên. Lab phân bổ đúng pattern đó: mỗi node phục vụ một loại khách hàng.

| Node | IP | Roles K8s | Roles Ceph | GPU pool | Khách hàng phục vụ |
|---|---|---|---|---|---|
| **ctrl01** | 10.10.200.11 | etcd, control-plane, worker | mon, osd | **không** — `gpu-pool=cpu-only` | Khách không cần GPU (web, dev VM, microservice...) |
| **ctrl02** | 10.10.200.12 | etcd, control-plane, worker | mon, osd | **whole GPU** — `gpu-pool=whole`, 2× simulated A100 | Khách mua nguyên GPU (training, render, Windows AI workstation) |
| **ctrl03** | 10.10.200.13 | etcd, control-plane, worker | mon, osd | **shared GPU** — `gpu-pool=shared`, 2× A100 chia 4 slice = 8 fractional | Khách mua GPU chia nhỏ (inference, dev/test, learning) |
| **VIP** | 10.10.200.51 | API endpoint (kube-vip ARP) | — | — | — |

Capsule inject `nodeSelector` per tenant để workload land đúng pool:

- `cust-cpu` → `gpu-pool=cpu-only` → ctrl01
- `cust-gpu-whole` → `gpu-pool=whole` → ctrl02
- `cust-gpu-shared` → `gpu-pool=shared` → ctrl03

---

## 2. Chuẩn bị Lab

3 VM Ubuntu 24.04.4 trên ESXi, cấu hình đồng nhất: **8 vCPU / 16 GB RAM / 200 GB OS / 100 GB Ceph OSD / ens160 (mgmt) + ens192 (cluster/storage)**.

> **ESXi:** Expose hardware assisted virtualization to the guest OS = true (VM Options → CPU) — bắt buộc để KVM nested chạy được.

```
ctrl01: 10.10.200.11  |  ctrl02: 10.10.200.12  |  ctrl03: 10.10.200.13
VIP K8s API:  10.10.200.51
LB pool 10.10.200.60-99:
  .61 Ceph dashboard  |  .62 KubeVirt Manager
  .66 cust-cpu  |  .67 cust-gpu-whole  |  .68 cust-gpu-shared
Cluster CIDR: 10.42.0.0/16  |  Service CIDR: 10.43.0.0/16  |  DNS: 10.43.0.10
```

Mình chạy block chuẩn bị OS sau trên cả 3 node:

```bash
# Swap off (bắt buộc cho Kubernetes)
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Disable ufw và apparmor (conflict với KVM nested)
systemctl disable --now ufw apparmor

# Chrony — Ceph yêu cầu clock skew < 50ms giữa các mon
apt update && apt install -y chrony
chronyc tracking | grep "Leap status"

# Kernel modules
tee /etc/modules-load.d/k8s.conf <<EOF
br_netfilter
overlay
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF
modprobe br_netfilter overlay ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack

# KVM nested (bắt buộc cho KubeVirt)
modprobe kvm_intel nested=1
echo "options kvm_intel nested=1" > /etc/modprobe.d/kvm-intel.conf
cat /sys/module/kvm_intel/parameters/nested   # phải ra: Y

# sysctl
tee /etc/sysctl.d/99-k8s-ceph-kubevirt.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
net.ipv4.conf.all.rp_filter         = 0
net.ipv4.conf.default.rp_filter     = 0
vm.swappiness                       = 0
vm.overcommit_memory                = 1
vm.max_map_count                    = 524288
fs.inotify.max_user_instances       = 8192
fs.inotify.max_user_watches         = 524288
fs.file-max                         = 2097152
kernel.pid_max                      = 4194304
net.core.somaxconn                  = 32768
net.netfilter.nf_conntrack_max      = 1000000
EOF
sysctl --system

# Packages
apt install -y \
  curl wget vim git jq htop iotop \
  nfs-common open-iscsi cryptsetup ceph-common \
  qemu-guest-agent gnupg lsb-release apt-transport-https \
  ca-certificates software-properties-common

# Ubuntu 24.04: phải dùng iptables-legacy — nftables mặc định làm KubeVirt rules fail im lặng
update-alternatives --set iptables /usr/sbin/iptables-legacy
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy

# /etc/hosts
tee -a /etc/hosts <<EOF
10.10.200.11  ctrl01
10.10.200.12  ctrl02
10.10.200.13  ctrl03
10.10.200.51  k8s-api.anhlx.lab
EOF
```

---

## 3. Cài RKE2 HA Cluster

Mình bootstrap ctrl01 trước, sau đó join ctrl02 và ctrl03.

### 3.1 Bootstrap node đầu tiên (ctrl01)

Tạo config trước khi install — quan trọng vì RKE2 sẽ apply manifests trong `/var/lib/rancher/rke2/server/manifests/` ngay khi start:

```bash
mkdir -p /etc/rancher/rke2 /var/lib/rancher/rke2/server/manifests

tee /etc/rancher/rke2/config.yaml <<EOF
# Tên cluster
write-kubeconfig-mode: "0644"
token: "Sup3rSecr3tT0ken-Lab-2026"

# Node-specific
node-name: "ctrl01"
node-ip: "10.10.200.11"
advertise-address: "10.10.200.11"

# TLS SAN — bao gồm VIP để client kubectl tin
tls-san:
  - "10.10.200.51"
  - "k8s-api.anhlx.lab"
  - "ctrl01"
  - "ctrl02"
  - "ctrl03"
  - "10.10.200.11"
  - "10.10.200.12"
  - "10.10.200.13"

# CNI = Cilium (sẽ override config phía dưới)
cni: cilium

# Disable components mặc định không dùng
disable:
  - rke2-ingress-nginx          # Không dùng ingress nginx
  - rke2-canal                  # Bỏ canal, dùng Cilium
  - rke2-snapshot-validation-webhook   # Lab; production nên giữ

# Kube-proxy disabled — Cilium replace bằng eBPF
disable-kube-proxy: true

# CIDR
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
cluster-dns: "10.43.0.10"

# Etcd snapshot
etcd-snapshot-schedule-cron: "0 */6 * * *"
etcd-snapshot-retention: 28
EOF
```

Tạo HelmChartConfig cho Cilium **trước** khi start RKE2. RKE2 sẽ apply manifests theo alphabetical order trong thư mục `manifests/`:

```bash
tee /var/lib/rancher/rke2/server/manifests/rke2-cilium-config.yaml <<EOF
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-cilium
  namespace: kube-system
spec:
  valuesContent: |-
    # ===== Kube-proxy replacement =====
    kubeProxyReplacement: true
    k8sServiceHost: 10.10.200.11    # Bootstrap với IP ctrl01, sẽ update sang VIP sau khi cluster up
    k8sServicePort: 6443

    # ===== eBPF datapath =====
    bpf:
      masquerade: true
    routingMode: tunnel
    tunnelProtocol: vxlan

    # ===== Ingress / LB =====
    # L2 Announcements — thay MetalLB
    l2announcements:
      enabled: true
      leaseDuration: "15s"
      leaseRenewDeadline: "10s"
      leaseRetryPeriod: "2s"

    externalIPs:
      enabled: true

    # ===== Hubble (observability) =====
    hubble:
      enabled: true
      relay:
        enabled: true
      ui:
        enabled: true

    # ===== IPAM =====
    ipam:
      mode: "cluster-pool"
      operator:
        clusterPoolIPv4PodCIDRList:
          - "10.42.0.0/16"
        clusterPoolIPv4MaskSize: 24

    # ===== Operator =====
    operator:
      replicas: 2

    # ===== Resource =====
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        memory: 1Gi
EOF
```

Lưu ý quan trọng về L2 Announcements: cần `kubeProxyReplacement: true` (đã có), `externalIPs.enabled: true` (đã có), và `l2announcements.enabled: true`. Devices interface sẽ auto-detect; nếu cần force, thêm `devices: "ens160"`.

Tạo kube-vip manifest cho VIP API (timing default 15/10/2):

```bash
tee /var/lib/rancher/rke2/server/manifests/kube-vip.yaml <<EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-vip
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:kube-vip-role
rules:
  - apiGroups: [""]
    resources: ["services", "services/status", "nodes", "endpoints"]
    verbs: ["list","get","watch","update","patch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["list","get","watch","update","create"]
  - apiGroups: ["discovery.k8s.io"]
    resources: ["endpointslices"]
    verbs: ["list","get","watch","update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-vip-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-vip-role
subjects:
  - kind: ServiceAccount
    name: kube-vip
    namespace: kube-system
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-vip-ds
  namespace: kube-system
  labels:
    app.kubernetes.io/name: kube-vip-ds
    app.kubernetes.io/version: v0.9.0
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-vip-ds
  template:
    metadata:
      labels:
        app.kubernetes.io/name: kube-vip-ds
        app.kubernetes.io/version: v0.9.0
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-role.kubernetes.io/control-plane
                    operator: Exists
      containers:
        - name: kube-vip
          image: ghcr.io/kube-vip/kube-vip:v0.9.0
          imagePullPolicy: IfNotPresent
          args:
            - manager
          env:
            - name: vip_arp
              value: "true"
            - name: port
              value: "6443"
            - name: vip_cidr
              value: "32"
            - name: cp_enable
              value: "true"
            - name: cp_namespace
              value: kube-system
            - name: vip_ddns
              value: "false"
            - name: svc_enable
              value: "false"   # Service LB do Cilium L2 lo, không phải kube-vip
            - name: vip_leaderelection
              value: "true"
            # === Timing default Kubernetes 15/10/2 
            - name: vip_leaseduration
              value: "15"
            - name: vip_renewdeadline
              value: "10"
            - name: vip_retryperiod
              value: "2"
            - name: address
              value: "10.10.200.51"
          securityContext:
            capabilities:
              add: ["NET_ADMIN","NET_RAW","SYS_TIME"]
      hostNetwork: true
      serviceAccountName: kube-vip
      tolerations:
        - effect: NoSchedule
          operator: Exists
        - effect: NoExecute
          operator: Exists
EOF
```

Cài và start RKE2:

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.35.4+rke2r1 sh -

systemctl enable rke2-server.service
systemctl start rke2-server.service

# Theo dõi log đến khi node Ready (mất ~3-5 phút)
journalctl -u rke2-server -f
```

Trong khi chờ, mở terminal khác setup kubectl:

```bash
cp /etc/rancher/rke2/rke2.yaml ~/rke2.yaml
chown $(id -u):$(id -g) ~/rke2.yaml
export KUBECONFIG=~/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin

echo 'export KUBECONFIG=~/rke2.yaml' >> ~/.bashrc
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc

kubectl get nodes -o wide
```

Verify Cilium + kube-vip up:

```bash
kubectl get pods -n kube-system | grep -E "cilium|kube-vip"
# cilium-xxxxx                        1/1 Running
# cilium-operator-xxxxx               1/1 Running
# kube-vip-ds-xxxxx                   1/1 Running

# VIP đã claimed?
ip addr show ens160 | grep 10.10.200.51
# inet 10.10.200.51/32 scope global ens160
```

Test access qua VIP:

```bash
kubectl get nodes -o wide --server=https://10.10.200.51:6443
# root@ctrl01:/root/# kubectl get nodes -o wide --server=https://10.10.200.51:6443
# NAME     STATUS   ROLES                AGE   VERSION          INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
# ctrl01   Ready    control-plane,etcd   19m   v1.35.4+rke2r1   10.10.200.11   <none>        Ubuntu 24.04.4 LTS   6.8.0-106-generic   containerd://2.2.3-k3s1
# root@ctrl01:/root/#

```

### 3.2 Join ctrl02, ctrl03

Lấy token từ ctrl01:

```bash
# Token đã set trong config: Sup3rSecr3tT0ken-Lab-2026
# Hoặc đọc từ file:
cat /var/lib/rancher/rke2/server/node-token
```

Trên **ctrl02**:

```bash
mkdir -p /etc/rancher/rke2

tee /etc/rancher/rke2/config.yaml <<EOF
server: https://10.10.200.51:9345
token: "Sup3rSecr3tT0ken-Lab-2026"

node-name: "ctrl02"
node-ip: "10.10.200.12"
advertise-address: "10.10.200.12"

tls-san:
  - "10.10.200.51"
  - "k8s-api.anhlx.lab"
  - "ctrl02"

cni: cilium
disable-kube-proxy: true

disable:
  - rke2-ingress-nginx
  - rke2-canal
  - rke2-snapshot-validation-webhook
EOF

curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.35.4+rke2r1 sh -
systemctl enable --now rke2-server.service

journalctl -u rke2-server -f
```

Tương tự cho **ctrl03**, đổi `node-name`, `node-ip`, `advertise-address`:

```bash
mkdir -p /etc/rancher/rke2

tee /etc/rancher/rke2/config.yaml <<EOF
server: https://10.10.200.51:9345
token: "Sup3rSecr3tT0ken-Lab-2026"

node-name: "ctrl03"
node-ip: "10.10.200.13"
advertise-address: "10.10.200.13"

tls-san:
  - "10.10.200.51"
  - "k8s-api.anhlx.lab"
  - "ctrl03"

cni: cilium
disable-kube-proxy: true

disable:
  - rke2-ingress-nginx
  - rke2-canal
  - rke2-snapshot-validation-webhook
EOF

curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.35.4+rke2r1 sh -
systemctl enable --now rke2-server.service
```

### 3.3 Verify cluster HA

Từ ctrl01:

```bash
kubectl get nodes -o wide
# NAME     STATUS   ROLES                AGE   VERSION          INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
# ctrl01   Ready    control-plane,etcd   27m   v1.35.4+rke2r1   10.10.200.11   <none>        Ubuntu 24.04.4 LTS   6.8.0-106-generic   containerd://2.2.3-k3s1
# ctrl02   Ready    control-plane,etcd   73s   v1.35.4+rke2r1   10.10.200.12   <none>        Ubuntu 24.04.4 LTS   6.8.0-106-generic   containerd://2.2.3-k3s1
# ctrl03   Ready    control-plane,etcd   42s   v1.35.4+rke2r1   10.10.200.13   <none>        Ubuntu 24.04.4 LTS   6.8.0-117-generic   containerd://2.2.3-k3s1

kubectl get pods -n kube-system
# Tất cả pod Cilium, kube-vip, coredns, etcd, kube-apiserver, kube-controller-manager, kube-scheduler đều Running

# Verify etcd quorum
kubectl get pods -n kube-system -l component=etcd -o wide
# NAME          READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
# etcd-ctrl01   1/1     Running   0          28m     10.10.200.11   ctrl01   <none>           <none>
# etcd-ctrl02   1/1     Running   0          2m34s   10.10.200.12   ctrl02   <none>           <none>
# etcd-ctrl03   1/1     Running   0          116s    10.10.200.13   ctrl03   <none>           <none>
```

**Update Cilium k8sServiceHost từ ctrl01 IP sang VIP** — đây là pattern bootstrap mà cần làm cẩn thận:

```bash
kubectl -n kube-system patch helmchartconfig rke2-cilium --type=merge -p='
{
  "spec": {
    "valuesContent": "kubeProxyReplacement: true\nk8sServiceHost: 10.10.200.51\nk8sServicePort: 6443\nbpf:\n  masquerade: true\nroutingMode: tunnel\ntunnelProtocol: vxlan\nl2announcements:\n  enabled: true\n  leaseDuration: \"15s\"\n  leaseRenewDeadline: \"10s\"\n  leaseRetryPeriod: \"2s\"\nexternalIPs:\n  enabled: true\nhubble:\n  enabled: true\n  relay:\n    enabled: true\n  ui:\n    enabled: true\nipam:\n  mode: \"cluster-pool\"\n  operator:\n    clusterPoolIPv4PodCIDRList:\n      - \"10.42.0.0/16\"\n    clusterPoolIPv4MaskSize: 24\noperator:\n  replicas: 2\nresources:\n  requests:\n    cpu: 100m\n    memory: 256Mi\n  limits:\n    memory: 1Gi"
  }
}'

# Helm chart sẽ reconcile, restart Cilium pod từng node (rolling)
kubectl -n kube-system rollout status ds/cilium

# Sau khi xong, mỗi Cilium pod đều talk tới VIP
kubectl -n kube-system exec ds/cilium -- env | grep -E "KUBERNETES_SERVICE|CILIUM_K8S"
# KUBERNETES_SERVICE_HOST=10.10.200.51
# KUBERNETES_SERVICE_PORT=6443
# CILIUM_K8S_NAMESPACE=kube-system
# KUBERNETES_SERVICE_PORT_HTTPS=443

kubectl -n kube-system logs ds/cilium | grep -i "Establishing connection to"
# Found 3 pods, using pod/cilium-q4qnn
# time=2026-05-27T02:55:46.494097255Z level=info msg="Establishing connection to apiserver" module=agent.infra.k8s-client ipAddr=https://10.10.200.51:6443
```

---

## 4. kubectl, Helm, Cilium CLI

```bash
# Helm 3 (3.21+)
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
# version.BuildInfo{Version:"v3.21.x"...}

# Cilium CLI (cho debug)
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
tar xzvf cilium-linux-amd64.tar.gz -C /usr/local/bin
rm cilium-linux-amd64.tar.gz

cilium status --wait
# Cilium: OK | Operator: OK | Hubble Relay: OK | DaemonSet 3/3 Ready
```

---

## 5. Cilium L2 Announcements — LoadBalancer cho bare metal (thay MetalLB)

Cilium đã có L2 Announcements production-ready — mình không cần cài thêm MetalLB nữa. Một CNI stack gọn hơn, Hubble thấy được cả LB traffic, và không phải duy trì upgrade matrix cho 2 component khác vendor.

Cấu hình gồm 2 CRD: `CiliumLoadBalancerIPPool` (range IP) + `CiliumL2AnnouncementPolicy` (chọn service nào announce ra interface nào).

```bash
# IP pool 10.10.200.60-99 cho service LoadBalancer
kubectl apply -f - <<EOF
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: lab-pool
spec:
  blocks:
    - start: "10.10.200.60"
      stop:  "10.10.200.99"
EOF
```

```bash
# Policy announce trên interface ens160 (mgmt network)
kubectl apply -f - <<EOF
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: default-l2-policy
  namespace: kube-system
spec:
  loadBalancerIPs: true
  externalIPs: true
  interfaces:
    - ^ens160$
  nodeSelector:
    matchLabels:
      kubernetes.io/os: linux
EOF
```

L2 Announcements hoạt động active-passive: chỉ 1 node tại 1 thời điểm ARP reply cho 1 service IP. Khi node đó down, Cilium leader election chọn node khác (10-15 giây). Đây là behavior giống MetalLB L2 mode.

---

## 6. Rook-Ceph Distributed Storage

Mình cài Rook-Ceph theo thứ tự: operator → CephCluster CR → pool và StorageClass. 

### 6.1 Cài Rook Operator

```bash
ROOK_VERSION=v1.19.5

kubectl apply --server-side -f https://raw.githubusercontent.com/rook/rook/${ROOK_VERSION}/deploy/examples/crds.yaml

# Xác nhận CRD core đã có
kubectl get crd cephclusters.ceph.rook.io

kubectl apply -f https://raw.githubusercontent.com/rook/rook/${ROOK_VERSION}/deploy/examples/common.yaml

kubectl apply -f https://raw.githubusercontent.com/rook/rook/${ROOK_VERSION}/deploy/examples/operator.yaml

kubectl -n rook-ceph rollout status deployment/rook-ceph-operator

# Rook v1.19+ mặc định bật ROOK_USE_CSI_OPERATOR=true nhưng OperatorConfig CRD (csi.ceph.io/v1)
# chưa được install → operator xóa toàn bộ CSI DaemonSet rồi không tạo lại được → PVC stuck Pending.
# Fix: tắt CSI Operator mode để Rook deploy CSI theo cách truyền thống (DaemonSet + Deployment).
kubectl -n rook-ceph patch configmap rook-ceph-operator-config --type=merge \
  -p '{"data":{"ROOK_USE_CSI_OPERATOR":"false"}}'

# CSI CRDs từ ceph-csi project — Rook v1.19+ cần nhưng không có trong crds.yaml
# Phải apply trước khi tạo CephCluster
kubectl apply -f - <<EOF
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: cephconnections.csi.ceph.io
spec:
  group: csi.ceph.io
  names:
    kind: CephConnection
    listKind: CephConnectionList
    plural: cephconnections
    singular: cephconnection
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          x-kubernetes-preserve-unknown-fields: true
      subresources:
        status: {}
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: clientprofiles.csi.ceph.io
spec:
  group: csi.ceph.io
  names:
    kind: ClientProfile
    listKind: ClientProfileList
    plural: clientprofiles
    singular: clientprofile
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          x-kubernetes-preserve-unknown-fields: true
      subresources:
        status: {}
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: networkfences.csi.ceph.io
spec:
  group: csi.ceph.io
  names:
    kind: NetworkFence
    listKind: NetworkFenceList
    plural: networkfences
    singular: networkfence
  scope: Cluster
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          x-kubernetes-preserve-unknown-fields: true
      subresources:
        status: {}
EOF
```

### 6.2 Tạo CephCluster

```bash
kubectl apply -f - <<EOF
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v19.2.3
    allowUnsupported: false
  dataDirHostPath: /var/lib/rook
  skipUpgradeChecks: false
  mon:
    count: 3
    allowMultiplePerNode: false
  mgr:
    count: 2
    allowMultiplePerNode: false
    modules:
      - name: dashboard
        enabled: true
      - name: prometheus
        enabled: true
  dashboard:
    enabled: true
    ssl: false
  monitoring:
    enabled: true
  network:
    provider: host
  crashCollector:
    disable: false
  cleanupPolicy:
    confirmation: ""
  resources:
    mon:
      requests:
        cpu: 200m
        memory: 512Mi
      limits:
        memory: 1Gi
    osd:
      requests:
        cpu: 300m
        memory: 1Gi
      limits:
        memory: 2Gi
    mgr:
      requests:
        cpu: 200m
        memory: 512Mi
      limits:
        memory: 1Gi
  storage:
    useAllNodes: true
    useAllDevices: false
    deviceFilter: "^sdb"   # Disk thứ 2 đã prep ở section 3.7
    config:
      osdsPerDevice: "1"
      storeType: bluestore
  placement:
    all:
      tolerations:
        - operator: Exists
EOF
```

Theo dõi cluster lên:

```bash
kubectl -n rook-ceph get cephcluster
# NAME        DATADIRHOSTPATH   MONCOUNT   AGE   PHASE   MESSAGE                         HEALTH
# rook-ceph   /var/lib/rook     3          5m    Ready   Cluster created successfully    HEALTH_OK

kubectl -n rook-ceph get pods
# rook-ceph-mon-a-xxx           Running
# rook-ceph-mon-b-xxx           Running
# rook-ceph-mon-c-xxx           Running
# rook-ceph-mgr-a-xxx           Running
# rook-ceph-mgr-b-xxx           Running
# rook-ceph-osd-0-xxx           Running
# rook-ceph-osd-1-xxx           Running
# rook-ceph-osd-2-xxx           Running
# rook-ceph-crashcollector-...  Running
```


### 6.3 Tạo Pools + StorageClass (replication=3, RBD-RWX, CephFS)

Replication=3 đảm bảo an toàn dữ liệu khi mất 1 OSD. RBD block-mode RWX cho phép VM live migration.

```bash
# RBD pool cho VM disk + PVC chung
kubectl apply -f - <<EOF
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  parameters:
    pg_num: "32"
    pgp_num: "32"
---
# CephFS cho RWX volume (logs, shared config, dataset)
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: cephfs
  namespace: rook-ceph
spec:
  metadataPool:
    failureDomain: host
    replicated:
      size: 3
  dataPools:
    - name: data0
      failureDomain: host
      replicated:
        size: 3
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true
    resources:
      requests:
        cpu: 200m
        memory: 512Mi
      limits:
        memory: 1Gi
EOF
```

Tạo **3 StorageClass**:

1. `rook-ceph-block` — RBD RWO (mặc định) cho container PVC
2. **`rook-ceph-block-rwx`** — RBD block-mode `ReadWriteMany` cho VM live migration
3. `rook-cephfs` — CephFS RWX cho shared filesystem

```bash
kubectl apply -f - <<EOF
---
# 1. RBD RWO — cho container thông thường (mặc định)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering,exclusive-lock,object-map,fast-diff,deep-flatten
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
  csi.storage.k8s.io/fstype: ext4
allowVolumeExpansion: true
reclaimPolicy: Delete
---
# 2. RBD block-mode RWX — cho VM disk hỗ trợ live migration ✨
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-rwx
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  # imageFeatures KHÔNG có exclusive-lock vì RWX multi-attach
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
  # KHÔNG có fstype — đây là block raw device, KubeVirt sẽ tự lo
allowVolumeExpansion: true
reclaimPolicy: Delete
---
# 3. CephFS RWX — cho shared filesystem
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: cephfs
  pool: cephfs-data0
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
allowVolumeExpansion: true
reclaimPolicy: Delete
EOF
```

Verify:

```bash
kubectl get sc
# NAME                  PROVISIONER                     RECLAIMPOLICY   ...
# rook-ceph-block (def) rook-ceph.rbd.csi.ceph.com      Delete
# rook-ceph-block-rwx   rook-ceph.rbd.csi.ceph.com      Delete
# rook-cephfs           rook-ceph.cephfs.csi.ceph.com   Delete
```

### 6.4 Ceph Dashboard

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: rook-ceph-mgr-dashboard-lb
  namespace: rook-ceph
spec:
  type: LoadBalancer
  selector:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
  ports:
    - name: dashboard
      port: 7000
      targetPort: 7000
EOF

kubectl get svc -n rook-ceph rook-ceph-mgr-dashboard-lb
# rook-ceph-mgr-dashboard-lb   LoadBalancer   10.43.x.x   10.10.200.61   7000:NodePort/TCP

# Lấy password admin
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 -d && echo
```

Truy cập http://10.10.200.61:7000 với user `admin` và password vừa lấy.

>Lưu ý: Ceph tự redirect sang IP của active MGR node — đây là behavior bình thường, dashboard vẫn hoạt động đầy đủ.
---

## 7. KubeVirt — chạy VM trên Kubernetes

KubeVirt + CDI. KubeVirt thế hệ này mang lại **PCIe NUMA topology awareness** cho AI VM (khi có GPU thật) và **Hypervisor Abstraction Layer** (chuẩn bị cho future multi-hypervisor).

```bash
# KubeVirt operator
KUBEVIRT_VERSION=v1.8.0
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-operator.yaml

# KubeVirt CR — cấu hình cluster
kubectl apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  certificateRotateStrategy: {}
  configuration:
    developerConfiguration:
      featureGates:
        - GPU
        - HostDevices
        - LiveMigration
        - VMLiveUpdateFeatures
        - ExpandDisks
        - HotplugVolumes
        - HotplugNICs
        - VMExport
        - Sysprep
    smbios:
      family: Lab-CloudVM
      manufacturer: GPU-Cloud-Lab
      product: KubeVirt
      sku: lab-2026
      version: "1.0"
  imagePullPolicy: IfNotPresent
  workloadUpdateStrategy:
    workloadUpdateMethods:
      - LiveMigrate
EOF

# Chờ KubeVirt ready
kubectl -n kubevirt wait --for=condition=Available kubevirt/kubevirt --timeout=10m

kubectl get pods -n kubevirt
# virt-api-xxxxx               1/1 Running
# virt-controller-xxxxx        1/1 Running (x2)
# virt-handler-xxxxx           1/1 Running (x3, một trên mỗi node)
# virt-operator-xxxxx          1/1 Running (x2)
```

Cài CDI:

```bash
CDI_VERSION=v1.65.0
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-operator.yaml
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-cr.yaml

# Config CDI dùng RBD-RWX làm storage cho scratch space
kubectl -n cdi patch cdi cdi --type merge --patch '
{
  "spec": {
    "config": {
      "scratchSpaceStorageClass": "rook-ceph-block"
    }
  }
}'

kubectl -n cdi wait --for=condition=Available cdi/cdi --timeout=5m

kubectl get pods -n cdi
# cdi-apiserver-xxxxx        1/1 Running
# cdi-deployment-xxxxx       1/1 Running
# cdi-operator-xxxxx         1/1 Running
# cdi-uploadproxy-xxxxx      1/1 Running
```

Cài virtctl CLI (cho admin):

```bash
KUBEVIRT_VERSION=v1.8.0
curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/virtctl-${KUBEVIRT_VERSION}-linux-amd64
chmod +x virtctl
mv virtctl /usr/local/bin/

virtctl version
```

Smoke test VM (Cirros nhỏ, ~50MB). Mình download image về node trước rồi serve local — CDI import từ internet chậm và dễ timeout:

```bash
# Download một lần trên ctrl01
wget -q https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img -O /tmp/cirros.img
python3 -m http.server 8888 --directory /tmp &
```

```bash
kubectl apply -f - <<EOF
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: cirros-dv
  namespace: default
spec:
  source:
    http:
      url: "http://10.10.200.11:8888/cirros.img"
  storage:
    volumeMode: Filesystem   # CDI v1.65 default Block mode với RBD → Permission denied; cần explicit Filesystem
    accessModes: [ReadWriteOnce]
    resources:
      requests:
        storage: 1Gi
    storageClassName: rook-ceph-block
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: cirros-smoke
  namespace: default
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        cpu:
          cores: 1
        resources:
          requests:
            memory: 256Mi
        devices:
          disks:
            - name: rootdisk
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
      networks:
        - name: default
          pod: {}
      volumes:
        - name: rootdisk
          dataVolume:
            name: cirros-dv
EOF

# Chờ import xong
kubectl get dv cirros-dv -w
# NAME        PHASE       PROGRESS   ...
# cirros-dv   Succeeded   100.0%

kubectl get vm,vmi
# NAME       AGE   STATUS    READY
# cirros-smoke   1m    Running   True

# Console
virtctl console cirros-smoke
# Login: cirros / gocubsgo
# Type 'exit' và Ctrl+] để thoát console

# Cleanup smoke test
kubectl delete vm cirros-smoke
kubectl delete dv cirros-dv
pkill -f "http.server 8888"

```

VM smoke test thành công xác nhận: CDI import work, KubeVirt scheduling work, virtio-net work, Cilium pod network với VM work.


---

## 8. KubeVirt Manager — Admin Portal

Admin UI cho VM lifecycle (start/stop/console/migrate).

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: kubevirt-manager
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubevirt-manager
  namespace: kubevirt-manager
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubevirt-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: kubevirt-manager
    namespace: kubevirt-manager
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubevirt-manager
  namespace: kubevirt-manager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubevirt-manager
  template:
    metadata:
      labels:
        app: kubevirt-manager
    spec:
      serviceAccountName: kubevirt-manager
      containers:
        - name: app
          image: kubevirtmanager/kubevirt-manager:latest
          ports: [{containerPort: 8080}]
          resources:
            requests:
              cpu: 20m
              memory: 64Mi
            limits:
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: kubevirt-manager
  namespace: kubevirt-manager
  annotations:
    io.cilium/lb-ipam-ips: "10.10.200.62"
spec:
  type: LoadBalancer
  selector:
    app: kubevirt-manager
  ports:
    - port: 80
      targetPort: 8080
EOF

kubectl -n kubevirt-manager rollout status deployment/kubevirt-manager
```

Truy cập http://10.10.200.62 — admin có thể list VM, start/stop/restart, mở VNC console, trigger live migration.

VirtualMachineClusterInstancetype + Preferences cho 3 khách hàng:

```bash
kubectl apply -f - <<EOF
---
# Instance type — small CPU-only (KH1)
apiVersion: instancetype.kubevirt.io/v1beta1
kind: VirtualMachineClusterInstancetype
metadata:
  name: u1.small
spec:
  cpu:
    guest: 1
  memory:
    guest: 2Gi
---
# Instance type — medium GPU whole (KH2 Windows)
apiVersion: instancetype.kubevirt.io/v1beta1
kind: VirtualMachineClusterInstancetype
metadata:
  name: gpu1.whole
spec:
  cpu:
    guest: 4
  memory:
    guest: 6Gi
  gpus:
    - name: gpu1
      deviceName: nvidia.com/gpu
---
# Preference — Ubuntu Linux
apiVersion: instancetype.kubevirt.io/v1beta1
kind: VirtualMachineClusterPreference
metadata:
  name: ubuntu
spec:
  cpu:
    preferredCPUTopology: preferSockets
  devices:
    preferredDiskBus: virtio
    preferredInterfaceModel: virtio
  features:
    preferredAcpi: {}
  firmware:
    preferredUseEfi: true
---
# Preference — Windows 2022
apiVersion: instancetype.kubevirt.io/v1beta1
kind: VirtualMachineClusterPreference
metadata:
  name: windows.2k22
spec:
  clock:
    preferredClockOffset:
      timezone: "Asia/Ho_Chi_Minh"
    preferredTimer:
      hpet:
        present: false
      hyperv: {}
      pit:
        tickPolicy: delay
      rtc:
        tickPolicy: catchup
  cpu:
    preferredCPUTopology: preferSockets
  devices:
    preferredDiskBus: sata
    preferredInterfaceModel: e1000
    preferredTPM: {}
  features:
    preferredAcpi: {}
    preferredApic: {}
    preferredHyperv:
      relaxed: {}
      vapic: {}
      spinlocks:
        spinlocks: 8191
    preferredSmm: {}
  firmware:
    preferredUseEfi: true
    preferredEfi:
      secureBoot: false
EOF
```

---

## 9. fake-gpu-operator — Topology theo node

Mình phân bổ 3 node theo loại GPU pool:

- **ctrl01** — KHÔNG có GPU pool, không label, không deploy fake-gpu-operator
- **ctrl02** — pool `whole`, 2 GPU integer (không time-slicing)
- **ctrl03** — pool `shared`, 2 GPU × 4 slice = 8 fractional

Bước 1: cài KWOK (dependency của fake-gpu-operator):

```bash
KWOK_VERSION=v0.7.0
kubectl apply -f https://github.com/kubernetes-sigs/kwok/releases/download/${KWOK_VERSION}/kwok.yaml

kubectl -n kube-system get pods -l app=kwok-controller
# kwok-controller-xxxxx   1/1   Running
```

Bước 2: Label nodes — đây là cách phân bổ chính:

```bash
# ctrl01 — CPU only, KHÔNG label gpu-pool simulated
kubectl label node ctrl01 gpu-pool=cpu-only --overwrite

# ctrl02 — whole GPU
kubectl label node ctrl02 \
  gpu-pool=whole \
  run.ai/simulated-gpu-node-pool=whole \
  --overwrite

# ctrl03 — shared GPU (time-slicing)
kubectl label node ctrl03 \
  gpu-pool=shared \
  run.ai/simulated-gpu-node-pool=shared \
  --overwrite

# Verify
kubectl get nodes -L gpu-pool,run.ai/simulated-gpu-node-pool
# NAME     STATUS   ROLES        AGE   VERSION    GPU-POOL    RUN.AI/SIMULATED-GPU-NODE-POOL
# ctrl01   Ready    ...          1h    v1.35.4    cpu-only    
# ctrl02   Ready    ...          1h    v1.35.4    whole       whole
# ctrl03   Ready    ...          1h    v1.35.4    shared      shared
```

Bước 3: cài fake-gpu-operator với 2 node pool (whole + shared):

```bash
# RKE2 tạo sẵn RuntimeClass "nvidia" với handler=runc — field này immutable, fake-gpu-operator
# cần handler khác nên không thể patch. Xóa trước để fake-gpu-operator tạo lại đúng.
kubectl delete runtimeclass nvidia --ignore-not-found

# Repo mới của run-ai chuyển sang ghcr.io
helm upgrade -i gpu-operator \
  oci://ghcr.io/run-ai/fake-gpu-operator/fake-gpu-operator \
  -n gpu-operator --create-namespace \
  --set 'topology.nodePools.whole.gpuCount=2' \
  --set 'topology.nodePools.whole.gpuProduct=NVIDIA-A100-SXM4-40GB' \
  --set 'topology.nodePools.shared.gpuCount=2' \
  --set 'topology.nodePools.shared.gpuProduct=NVIDIA-A100-SXM4-40GB'

kubectl -n gpu-operator get pods
# root@ctrl01:/root/# kubectl -n gpu-operator get pods -o wide
# NAME                                        READY   STATUS    RESTARTS   AGE    IP            NODE     NOMINATED NODE   READINESS GATES
# device-plugin-dw22c                         1/1     Running   0          112s   10.42.2.243   ctrl03   <none>           <none>
# device-plugin-zwc5c                         1/1     Running   0          112s   10.42.1.238   ctrl02   <none>           <none>
# kwok-gpu-device-plugin-58d44f6bff-6xmrq     1/1     Running   0          2m2s   10.42.0.206   ctrl01   <none>           <none>
# nvidia-dcgm-exporter-b88ht                  1/1     Running   0          112s   10.42.2.32    ctrl03   <none>           <none>
# nvidia-dcgm-exporter-kwok-94c9b7f66-nv2h9   1/1     Running   0          2m2s   10.42.1.32    ctrl02   <none>           <none>
# nvidia-dcgm-exporter-nn55z                  1/1     Running   0          112s   10.42.1.68    ctrl02   <none>           <none>
# status-updater-5c4f44ffcd-czltw             1/1     Running   0          2m2s   10.42.1.67    ctrl02   <none>           <none>
# topology-server-878d8fddc-6j28t             1/1     Running   0          2m2s   10.42.2.240   ctrl03   <none>           <none>
# root@ctrl01:/root/#
```

Quan trọng: fake-gpu-operator chỉ deploy device-plugin trên node có label `run.ai/simulated-gpu-node-pool=<pool>`. Vì ctrl01 không có label này, sẽ không có GPU pod nào chạy trên ctrl01.

Bước 4: configure time-slicing cho pool `shared`:

```bash
# Re-label ctrl03 sang pool shared
kubectl label node ctrl03 run.ai/simulated-gpu-node-pool=shared --overwrite

# fake-gpu-operator không tự nhân replicas vào nvidia.com/gpu trên real node —
# cần set gpuCount=8 thẳng (2 physical × 4 slices) và replicasPerGpu làm metadata
helm upgrade gpu-operator \
  oci://ghcr.io/run-ai/fake-gpu-operator/fake-gpu-operator \
  --namespace gpu-operator \
  --set topology.nodePools.whole.gpuCount=2 \
  --set topology.nodePools.whole.gpuProduct=NVIDIA-A100-SXM4-40GB \
  --set topology.nodePools.shared.gpuCount=8 \
  --set topology.nodePools.shared.gpuProduct=NVIDIA-A100-SXM4-40GB \
  --set topology.nodePools.shared.replicasPerGpu=4 \
  --wait

# Xóa topology ConfigMap cũ của ctrl03 để status-updater recreate với 8 GPU entries
kubectl -n gpu-operator delete configmap topology-ctrl03
kubectl -n gpu-operator rollout restart deployment/status-updater
sleep 15

# Restart device-plugin trên ctrl03 để kubelet cập nhật allocatable
kubectl -n gpu-operator delete pod -l app=device-plugin --field-selector spec.nodeName=ctrl03
sleep 15
```

Bước 5: verify GPU advertised đúng:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,POOL:.metadata.labels.run\\.ai/simulated-gpu-node-pool,GPU:.status.allocatable.nvidia\\.com/gpu

# root@ctrl01:/root/# kubectl get nodes -o custom-columns=NAME:.metadata.name,POOL:.metadata.labels.run\\.ai/simulated-gpu-node-pool,GPU:.status.allocatable.nvidia\\.com/gpu
# NAME     POOL     GPU
# ctrl01   <none>   <none>
# ctrl02   whole    2
# ctrl03   shared   8
# root@ctrl01:/root/#
```

Smoke test pod xin GPU trên từng pool:

```bash
# Pod xin GPU shared (1/4 GPU)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: gpu-smoke-shared
spec:
  nodeSelector:
    gpu-pool: shared
  restartPolicy: Never
  containers:
    - name: cuda
      image: nvcr.io/nvidia/cuda:12.6.0-base-ubuntu22.04
      command: ["sh", "-c"]
      args:
        - "nvidia-smi || true; sleep 30"
      resources:
        limits:
          nvidia.com/gpu: 1
      env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
EOF

kubectl logs gpu-smoke-shared
# root@ctrl01:/root/# kubectl logs gpu-smoke-shared
# Wed May 27 09:24:19 2026
# +------------------------------------------------------------------------------+
# | NVIDIA-SMI 470.129.06   Driver Version: 470.129.06   CUDA Version: 11.4      |
# +--------------------------------+----------------------+----------------------+
# | GPU  Name        Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
# | Fan  Temp  Perf  Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
# |                                |                      |               MIG M. |
# +--------------------------------+----------------------+----------------------+
# |   0  NVIDIA-A10..          Off | 00000001:00:00.0 Off |                  Off |
# | N/A   33C    P8    11W /  70W  |          0MiB / 0MiB |       0%     Default |
# |                                |                      |                  N/A |
# +--------------------------------+----------------------+----------------------+

# +------------------------------------------------------------------------------+
# | Processes:                                                                   |
# |  GPU   GI   CI        PID   Type   Process name                  GPU Memory  |
# |        ID   ID                                                   Usage       |
# +------------------------------------------------------------------------------+
# |    0   N/A  N/A       7        G   sh-cnvidia-smi || true; s..        0MiB   |
# +------------------------------------------------------------------------------+
# root@ctrl01:/root/#


# Verify schedule lên đúng ctrl03
kubectl get pod gpu-smoke-shared -o jsonpath='{.spec.nodeName}' && echo 
# ctrl03

kubectl delete pod gpu-smoke-shared
```

---

## 10. Capsule — Multi-tenancy với custom Role

Capsule enforce multi-tenancy đúng nghĩa khi tenant SA **không** được cấp `cluster-admin`. Nếu tenant có `cluster-admin`, họ có thể `kubectl delete node`, đọc secret của tenant khác, vô hiệu hóa toàn bộ multi-tenancy — Capsule chỉ enforce qua admission webhook, không revoke quyền đã grant.

Custom ClusterRole `tenant-owner` với verbs giới hạn là cách đúng. Capsule sẽ tự generate RoleBinding trong namespace của tenant.

### 10.1 Cài Capsule

```bash
helm install capsule oci://ghcr.io/projectcapsule/charts/capsule \
  --version 0.12.4 \
  -n capsule-system --create-namespace \
  --set "manager.resources.requests.cpu=20m" \
  --set "manager.resources.requests.memory=64Mi" \
  --set "manager.resources.limits.memory=256Mi"

kubectl -n capsule-system rollout status deployment/capsule-controller-manager

kubectl get pods -n capsule-system
# capsule-controller-manager-xxxxx   1/1 Running
```

### 10.2 Tạo custom ClusterRole `tenant-owner`

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tenant-owner
  labels:
    capsule.clastix.io/aggregation-to-tenant-owner: "true"
rules:
  # === Kubernetes core resources trong namespace ===
  - apiGroups: [""]
    resources:
      - pods
      - pods/log
      - pods/exec
      - pods/portforward
      - pods/attach
      - services
      - configmaps
      - secrets
      - persistentvolumeclaims
      - events
      - serviceaccounts
    verbs: ["get","list","watch","create","update","patch","delete"]
  
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get","list","watch","create","update","patch","delete"]
  
  # === Workload ===
  - apiGroups: ["apps"]
    resources: ["deployments","statefulsets","daemonsets","replicasets"]
    verbs: ["get","list","watch","create","update","patch","delete"]
  
  - apiGroups: ["batch"]
    resources: ["jobs","cronjobs"]
    verbs: ["get","list","watch","create","update","patch","delete"]
  
  # === KubeVirt VM ===
  - apiGroups: ["kubevirt.io"]
    resources:
      - virtualmachines
      - virtualmachineinstances
      - virtualmachineinstancemigrations
    verbs: ["get","list","watch","create","update","patch","delete"]
  
  - apiGroups: ["subresources.kubevirt.io"]
    resources:
      - virtualmachineinstances/console
      - virtualmachineinstances/vnc
      - virtualmachines/start
      - virtualmachines/stop
      - virtualmachines/restart
      - virtualmachines/migrate
    verbs: ["get","update"]
  
  # === CDI ===
  - apiGroups: ["cdi.kubevirt.io"]
    resources: ["datavolumes","dataimportcrons","datasources"]
    verbs: ["get","list","watch","create","update","patch","delete"]
  
  # === Networking trong namespace ===
  - apiGroups: ["networking.k8s.io"]
    resources: ["networkpolicies","ingresses"]
    verbs: ["get","list","watch","create","update","patch","delete"]
  
  # === Tenant chỉ được READ, không write ===
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get","list","watch"]
  
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses","volumesnapshotclasses"]
    verbs: ["get","list","watch"]
  
  # === Capsule Tenant (read-only — chỉ admin tạo) ===
  - apiGroups: ["capsule.clastix.io"]
    resources: ["tenants"]
    verbs: ["get","list","watch"]
EOF
```

Khi tenant tạo Tenant CR (do admin), Capsule sẽ tự sinh `RoleBinding` trong mỗi namespace của tenant với role `tenant-owner` cho ServiceAccount tenant — **không** cần ClusterRoleBinding manual.

### 10.3 Helper script — sinh kubeconfig tenant

```bash
cat > ~/create-tenant.sh <<'SCRIPT'
#!/usr/bin/env bash
# Usage: ./create-tenant.sh <tenant-name> <gpu-pool: cpu-only|whole|shared> <gpu-quota> <vm-quota>
# Ví dụ: ./create-tenant.sh cust-cpu cpu-only 0 1

set -euo pipefail
TENANT_NAME="$1"
GPU_POOL="$2"
GPU_QUOTA="${3:-0}"
VM_QUOTA="${4:-1}"

NS="${TENANT_NAME}"
SA="${TENANT_NAME}-owner"
SERVER="https://10.10.200.51:6443"

# Tạo Tenant CR — Capsule sẽ tự tạo namespace + RoleBinding
cat <<EOF | kubectl apply -f -
apiVersion: capsule.clastix.io/v1beta2
kind: Tenant
metadata:
  name: ${TENANT_NAME}
spec:
  owners:
    - name: system:serviceaccount:${NS}:${SA}
      kind: ServiceAccount
      clusterRoles:
        - tenant-owner    # ← Custom role thay vì cluster-admin
  namespaceOptions:
    quota: 1
  nodeSelector:
    gpu-pool: "${GPU_POOL}"
  ingressOptions:
    allowedClasses:
      allowed:
        - "cilium"
  storageClasses:
    allowed:
      - "rook-ceph-block"
      - "rook-ceph-block-rwx"
      - "rook-cephfs"
  resourceQuotas:
    scope: Tenant
    items:
      - hard:
          requests.cpu: "4"
          requests.memory: "8Gi"
          limits.cpu: "6"
          limits.memory: "10Gi"
          requests.nvidia.com/gpu: "${GPU_QUOTA}"
          limits.nvidia.com/gpu: "${GPU_QUOTA}"
          persistentvolumeclaims: "5"
          requests.storage: "100Gi"
          services.loadbalancers: "2"
          count/virtualmachines.kubevirt.io: "${VM_QUOTA}"
  networkPolicies:
    items:
      - policyTypes: [Ingress, Egress]
        ingress:
          - from:
              - namespaceSelector:
                  matchLabels:
                    capsule.clastix.io/tenant: "${TENANT_NAME}"
              - podSelector: {}
        egress:
          - to:
              - ipBlock:
                  cidr: 0.0.0.0/0
                  except:
                    - 169.254.169.254/32   # block metadata service
        podSelector: {}
EOF

# Tạo namespace và label cho Capsule
kubectl create namespace "${NS}" --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace "${NS}" "capsule.clastix.io/tenant=${TENANT_NAME}" --overwrite

# ServiceAccount tenant
kubectl create serviceaccount "${SA}" -n "${NS}" --dry-run=client -o yaml | kubectl apply -f -

# Capsule v0.12.x chỉ auto-tạo RoleBinding khi namespace được tạo bởi chính tenant owner qua webhook.
# Namespace tạo bởi admin bypass webhook → Capsule không track ownership (size=0) → không sinh RoleBinding.
# Phải tạo thủ công với tên convention capsule-<tenant>-0 để nhất quán với Capsule.
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: capsule-${TENANT_NAME}-0
  namespace: ${NS}
  labels:
    capsule.clastix.io/tenant: "${TENANT_NAME}"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tenant-owner
subjects:
  - kind: ServiceAccount
    name: ${SA}
    namespace: ${NS}
EOF

# Tạo Secret token vĩnh viễn cho SA (K8s 1.24+ không tự tạo)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: ${SA}-token
  namespace: ${NS}
  annotations:
    kubernetes.io/service-account.name: ${SA}
type: kubernetes.io/service-account-token
EOF

sleep 5

TOKEN=$(kubectl -n "${NS}" get secret "${SA}-token" -o jsonpath='{.data.token}' | base64 -d)
CA=$(kubectl -n "${NS}" get secret "${SA}-token" -o jsonpath='{.data.ca\.crt}')

# Sinh kubeconfig
mkdir -p ~/kubeconfigs
cat > ~/kubeconfigs/${TENANT_NAME}.kubeconfig <<EOF
apiVersion: v1
kind: Config
clusters:
  - name: lab-cluster
    cluster:
      server: ${SERVER}
      certificate-authority-data: ${CA}
users:
  - name: ${SA}
    user:
      token: ${TOKEN}
contexts:
  - name: ${TENANT_NAME}
    context:
      cluster: lab-cluster
      namespace: ${NS}
      user: ${SA}
current-context: ${TENANT_NAME}
EOF

echo "===================================="
echo "Tenant ${TENANT_NAME} created"
echo "  Namespace:    ${NS}"
echo "  GPU pool:     ${GPU_POOL}"
echo "  GPU quota:    ${GPU_QUOTA}"
echo "  VM quota:     ${VM_QUOTA}"
echo "  Kubeconfig:   ~/kubeconfigs/${TENANT_NAME}.kubeconfig"
echo "  Token (cho Headlamp):"
echo "  ${TOKEN}"
echo "===================================="
SCRIPT
chmod +x ~/create-tenant.sh
```

---

## 11. Golden Image Templates

Mình chuẩn bị golden image một lần — khi có KH mới, admin clone PVC ra là xong, không cài lại OS hay pull image từ đầu.

```bash
kubectl create namespace vm-images
```

### 11.1 Ubuntu 22.04 — VM golden image

Mình download Ubuntu 22.04 cloud image về ctrl01 và import vào namespace `vm-images`:

```bash
wget --progress=bar:force https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img \
  -O /tmp/jammy.img
python3 -m http.server 8888 --directory /tmp &

kubectl apply -f - <<EOF
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: ubuntu-2204-golden
  namespace: vm-images
spec:
  source:
    http:
      url: "http://10.10.200.11:8888/jammy.img"
  storage:
    accessModes: [ReadWriteMany]
    volumeMode: Filesystem
    resources:
      requests:
        storage: 10Gi
    storageClassName: rook-cephfs
EOF

kubectl get dv ubuntu-2204-golden -n vm-images -w
# NAME                 PHASE       PROGRESS   RESTARTS   AGE
# ubuntu-2204-golden   Succeeded   100.0%     0          3m
```

Khi có KH mua VM Ubuntu, admin clone PVC vào tenant namespace:

```bash
# Grant CDI cross-namespace clone permission
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cdi-cloner
  namespace: vm-images
subjects:
- kind: ServiceAccount
  name: default
  namespace: <tenant-namespace>
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f - <<EOF
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: ubuntu-2204-img
  namespace: <tenant-namespace>
spec:
  source:
    pvc:
      namespace: vm-images
      name: ubuntu-2204-golden
  storage:
    accessModes: [ReadWriteMany]
    volumeMode: Filesystem
    resources:
      requests:
        storage: 10Gi
    storageClassName: rook-cephfs
EOF
```

### 11.2 Windows Server 2022 — VM golden image

Mình SCP Windows ISO từ máy local lên ctrl01 rồi import vào `vm-images`. Bước này chỉ làm một lần.

```powershell
# Trên Windows laptop
scp "D:\anhle\soft\iso\2022SERVER_EVAL_x64FRE_en-us.iso" root@10.10.200.11:/tmp/win2022.iso
```

```bash
python3 -m http.server 8888 --directory /tmp &

kubectl apply -f - <<EOF
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: windows-2022-iso
  namespace: vm-images
spec:
  source:
    http:
      url: "http://10.10.200.11:8888/win2022.iso"
  storage:
    accessModes: [ReadWriteOnce]
    volumeMode: Filesystem
    resources:
      requests:
        storage: 6Gi
    storageClassName: rook-ceph-block
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: windows-2022-golden
  namespace: vm-images
spec:
  accessModes: [ReadWriteMany]
  storageClassName: rook-ceph-block-rwx
  volumeMode: Block
  resources:
    requests:
      storage: 60Gi
EOF
```

Tạo VM installer để cài Windows:

```bash
kubectl apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: windows-installer
  namespace: vm-images
spec:
  runStrategy: Manual
  template:
    spec:
      nodeSelector:
        gpu-pool: whole
      domain:
        cpu:
          cores: 4
          sockets: 1
          threads: 1
        memory:
          guest: 6Gi
        firmware:
          bootloader:
            efi:
              secureBoot: false
        clock:
          timezone: "Asia/Ho_Chi_Minh"
          timer:
            hpet:
              present: false
            hyperv: {}
            pit:
              tickPolicy: delay
            rtc:
              tickPolicy: catchup
        features:
          acpi: {}
          apic: {}
          smm:
            enabled: true
          hyperv:
            relaxed: {}
            vapic: {}
            spinlocks:
              spinlocks: 8191
        devices:
          tpm: {}
          disks:
            - name: rootdisk
              disk:
                bus: virtio
              bootOrder: 2
            - name: installer-iso
              cdrom:
                bus: sata
              bootOrder: 1
            - name: virtiocontainerdisk
              cdrom:
                bus: sata
              bootOrder: 3
          interfaces:
            - name: default
              masquerade: {}
              model: e1000
      networks:
        - name: default
          pod: {}
      volumes:
        - name: rootdisk
          persistentVolumeClaim:
            claimName: windows-2022-golden
        - name: installer-iso
          persistentVolumeClaim:
            claimName: windows-2022-iso
        - name: virtiocontainerdisk
          containerDisk:
            image: quay.io/kubevirt/virtio-container-disk:v1.8.0
EOF

virtctl start windows-installer -n vm-images
```

Mở VNC console để cài Windows (MobaXterm → VNC → `10.10.200.11:5900`):

```bash
virtctl vnc windows-installer -n vm-images --proxy-only --port=5900 --address=0.0.0.0
```

Trong installer:

1. Boot từ DVD → ngôn ngữ → edition **Standard Desktop Experience**
2. Custom install → Load driver → `viostor\2k22\amd64` → install → disk 60 GB hiện ra → Next → Install
3. Reboot → tạo Administrator password → login desktop
4. Mount virtio-win CD → `virtio-win-gt-x64.msi` → install tất cả driver → reboot

Bật RDP rồi chạy sysprep để hoàn tất golden image:

```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
Restart-Service -Name TermService -Force

C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown
```

Sau khi VMI chuyển sang phase `Succeeded`, xóa installer VM:

```bash
kubectl get pvc -n vm-images windows-2022-golden
# NAME                   STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
# windows-2022-golden    Bound    ...      60Gi       RWX            rook-ceph-block-rwx

kubectl delete vm windows-installer -n vm-images
kubectl delete dv windows-2022-iso -n vm-images
```

Khi có KH mua Windows VM, admin clone từ golden image:

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cdi-cloner
  namespace: vm-images
subjects:
- kind: ServiceAccount
  name: default
  namespace: <tenant-namespace>
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f - <<EOF
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: windows-2022-rootdisk
  namespace: <tenant-namespace>
spec:
  source:
    pvc:
      namespace: vm-images
      name: windows-2022-golden
  storage:
    accessModes: [ReadWriteMany]
    volumeMode: Block
    resources:
      requests:
        storage: 60Gi
    storageClassName: rook-ceph-block-rwx
EOF

kubectl get dv -n <tenant-namespace> windows-2022-rootdisk -w
# NAME                    PHASE       PROGRESS   RESTARTS   AGE
# windows-2022-rootdisk   Succeeded   100.0%                32s
```

### 11.3 Container template — CUDA workload (KH3)

Mình lưu sẵn manifest CUDA container vào ConfigMap để admin tái dùng cho các tenant GPU shared:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cuda-container-template
  namespace: vm-images
data:
  template.yaml: |
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: TENANT-data
      namespace: TENANT-NAMESPACE
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: rook-ceph-block
      resources:
        requests:
          storage: 10Gi
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: TENANT-gpu-app
      namespace: TENANT-NAMESPACE
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: TENANT-gpu-app
      template:
        metadata:
          labels:
            app: TENANT-gpu-app
        spec:
          containers:
            - name: cuda
              image: nvcr.io/nvidia/cuda:12.6.0-base-ubuntu22.04
              command: ["sh", "-c"]
              args:
                - |
                  apt-get update -qq && apt-get install -qq -y openssh-server > /dev/null 2>&1
                  mkdir -p /run/sshd
                  echo 'root:PASSWD' | chpasswd
                  sed -i 's/#PermitRootLogin .*/PermitRootLogin yes/' /etc/ssh/sshd_config
                  sed -i 's/#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
                  /usr/sbin/sshd -D
              resources:
                requests:
                  cpu: 500m
                  memory: 1Gi
                  nvidia.com/gpu: 1
                limits:
                  cpu: 2
                  memory: 2Gi
                  nvidia.com/gpu: 1
              ports:
                - containerPort: 22
              env:
                - name: NODE_NAME
                  valueFrom:
                    fieldRef:
                      fieldPath: spec.nodeName
              volumeMounts:
                - name: data
                  mountPath: /data
          volumes:
            - name: data
              persistentVolumeClaim:
                claimName: TENANT-data
EOF
```

---

## 12. Demo: Bán cho 3 khách hàng khác nhau

Đây là phần demo flow thật sự của GPU cloud. Mỗi khách hàng có nhu cầu khác nhau, lab này phục vụ cả 3 trên cùng một cluster — chính xác mô hình kinh doanh của cloud provider thương mại.

### 12.1 Tổng quan 3 khách hàng 

| KH | Tenant name | Sản phẩm mua | Node phục vụ | GPU pool | Login method |
|---|---|---|---|---|---|
| **KH1** | `cust-cpu` | 1 VM Ubuntu 22.04, 2 vCPU, 4 GB RAM, không GPU | ctrl01 | cpu-only | SSH qua Floating IP `10.10.200.66:22` |
| **KH2** | `cust-gpu-whole` | 1 VM Windows Server 2022, 4 vCPU, 8 GB RAM, **1 GPU nguyên** | ctrl02 | whole | RDP qua Floating IP `10.10.200.67:3389` |
| **KH3** | `cust-gpu-shared` | 1 container, **1/4 GPU shared** | ctrl03 | shared | `kubectl exec` qua Headlamp / kubeconfig |

Mỗi tenant có ResourceQuota riêng, NetworkPolicy isolation tự động qua Capsule, nodeSelector inject để workload land đúng pool.

### 12.2 Admin chuẩn bị Tenants

Trên ctrl01 (admin), chạy script đã tạo ở section 10:

```bash
# KH1 — cpu only, 0 GPU, 1 VM
~/create-tenant.sh cust-cpu cpu-only 0 1

# KH2 — whole GPU, 1 GPU, 1 VM
~/create-tenant.sh cust-gpu-whole whole 1 1

# KH3 — shared GPU, 1 GPU slice, 0 VM (chỉ container)
~/create-tenant.sh cust-gpu-shared shared 1 0
```

Verify cả 3 tenant lên:

```bash
kubectl get tenants.capsule.clastix.io
# NAME                STATE    NAMESPACE QUOTA    NAMESPACE COUNT
# cust-cpu            Active   1                  1
# cust-gpu-whole      Active   1                  1
# cust-gpu-shared     Active   1                  1

kubectl get ns -l 'capsule.clastix.io/tenant'
# NAME                STATUS   AGE
# cust-cpu            Active   30s
# cust-gpu-whole      Active   25s
# cust-gpu-shared     Active   20s

# Verify RoleBinding với role tenant-owner (không phải cluster-admin)
kubectl -n cust-cpu get rolebinding
# NAME                          ROLE                       AGE
# capsule-cust-cpu-0            ClusterRole/tenant-owner   30s

# Verify SA KHÔNG có cluster-admin
kubectl get clusterrolebinding | grep cust-cpu
# (empty — đúng, không có ClusterRoleBinding cluster-admin nào)
```

3 kubeconfig đã sinh ở `~/kubeconfigs/`:
- `cust-cpu.kubeconfig`
- `cust-gpu-whole.kubeconfig`
- `cust-gpu-shared.kubeconfig`

Mình clone golden image vào từng tenant namespace (thao tác admin, dùng pattern từ section 11):

```bash
# KH1 — clone Ubuntu golden image → cust-cpu
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cdi-cloner
  namespace: vm-images
subjects:
- kind: ServiceAccount
  name: default
  namespace: cust-cpu
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f - <<EOF
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: ubuntu-2204-img
  namespace: cust-cpu
spec:
  source:
    pvc:
      namespace: vm-images
      name: ubuntu-2204-golden
  storage:
    accessModes: [ReadWriteMany]
    volumeMode: Filesystem
    resources:
      requests:
        storage: 10Gi
    storageClassName: rook-cephfs
EOF

# KH2 — clone Windows golden image → cust-gpu-whole
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cdi-cloner
  namespace: vm-images
subjects:
- kind: ServiceAccount
  name: default
  namespace: cust-gpu-whole
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f - <<EOF
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: windows-2022-rootdisk
  namespace: cust-gpu-whole
spec:
  source:
    pvc:
      namespace: vm-images
      name: windows-2022-golden
  storage:
    accessModes: [ReadWriteMany]
    volumeMode: Block
    resources:
      requests:
        storage: 60Gi
    storageClassName: rook-ceph-block-rwx
EOF

# Chờ cả 2 clone xong
kubectl get dv -n cust-cpu ubuntu-2204-img -w
kubectl get dv -n cust-gpu-whole windows-2022-rootdisk -w
```

Trong demo dưới, mỗi customer dùng kubeconfig của họ — admin **không** đụng nữa.

### 12.3 KH1 — cust-cpu mua VM Ubuntu 22.04 (không GPU)

Khách hàng: agency làm web design, cần VM dev environment có Ubuntu. Không cần GPU.

KH1 dùng `cust-cpu.kubeconfig`:

```bash
export KUBECONFIG=~/kubeconfigs/cust-cpu.kubeconfig

# Verify quyền hạn — chỉ thấy namespace của mình
kubectl get pods -A 2>&1 | head -3
# Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:cust-cpu:cust-cpu-owner"
# cannot list resource "pods" in API group "" at the cluster scope

kubectl get pods   # default namespace = cust-cpu
# No resources found in cust-cpu namespace.

kubectl get ns
# Error from server (Forbidden): namespaces is forbidden: User "system:serviceaccount:cust-cpu:cust-cpu-owner"
# cannot list resource "namespaces" in API group "" at the cluster scope
```

Đây là isolation thật — không phải fake do cluster-admin. Cụ thể nếu thử với secret tenant khác:

```bash
kubectl get secrets -n cust-gpu-whole 2>&1 | head -2
# Error from server (Forbidden): secrets is forbidden: User "system:serviceaccount:cust-cpu:cust-cpu-owner"
# cannot list resource "secrets" in API group "" in the namespace "cust-gpu-whole"
```

PVC `ubuntu-2204-img` đã được admin clone sẵn vào namespace `cust-cpu` ở bước 12.2.

Bước 1: Tạo VM với cloud-init (password login, sudo):

```bash
# Gen password hash — thay <your-password> bằng pass thực
PASSWD_HASH=$(openssl passwd -6 '<your-password>')

kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: ubuntu-cloudinit
  namespace: cust-cpu
type: Opaque
stringData:
  userdata: |
    #cloud-config
    hostname: cust-cpu-vm
    users:
      - name: ubuntu
        sudo: ALL=(ALL) NOPASSWD:ALL
        shell: /bin/bash
        lock_passwd: false
        passwd: ${PASSWD_HASH}
    ssh_pwauth: true
    runcmd:
      - systemctl enable --now ssh
      - echo "KH1 lab cust-cpu — Ubuntu 22.04 VM ready" > /etc/motd
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: cust-cpu-vm
  namespace: cust-cpu
  labels:
    customer: cust-cpu
    workload-type: dev-vm
spec:
  runStrategy: Always
  instancetype:
    kind: VirtualMachineClusterInstancetype
    name: u1.small
  preference:
    kind: VirtualMachineClusterPreference
    name: ubuntu
  template:
    metadata:
      labels:
        customer: cust-cpu
        kubevirt.io/vm: cust-cpu-vm
    spec:
      # Không có GPU request — Capsule inject nodeSelector gpu-pool=cpu-only → land ctrl01
      domain:
        devices:
          disks:
            - name: rootdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
          networkInterfaceMultiqueue: true
      networks:
        - name: default
          pod: {}
      volumes:
        - name: rootdisk
          dataVolume:
            name: ubuntu-2204-img
        - name: cloudinitdisk
          cloudInitNoCloud:
            secretRef:
              name: ubuntu-cloudinit
EOF

# Chờ VM running (~30 giây sau khi DV ready)
kubectl get vm,vmi -n cust-cpu
# NAME                                     AGE   STATUS    READY
# virtualmachine.kubevirt.io/cust-cpu-vm   22s   Running   True
#
# NAME                                             AGE   PHASE     IP            NODENAME   READY
# virtualmachineinstance.kubevirt.io/cust-cpu-vm   22s   Running   10.42.0.219   ctrl01     True
```

Verify VM land đúng **ctrl01** (cpu-only node):

```bash
kubectl get vmi cust-cpu-vm -o jsonpath='{.status.nodeName}' && echo
# ctrl01
```

Bước 3: Expose SSH qua Floating IP `10.10.200.66`:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: cust-cpu-vm-ssh
  namespace: cust-cpu
  annotations:
    io.cilium/lb-ipam-ips: "10.10.200.66"
spec:
  type: LoadBalancer
  selector:
    customer: cust-cpu
    kubevirt.io/vm: cust-cpu-vm
  ports:
    - name: ssh
      port: 22
      targetPort: 22
      protocol: TCP
EOF

kubectl get svc -n cust-cpu cust-cpu-vm-ssh
# NAME              TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
# cust-cpu-vm-ssh   LoadBalancer   10.43.78.140   10.10.200.66   22:32051/TCP   8s
```

Bước 4: KH1 SSH vào VM **giống VPS thật**:

```bash
ssh ubuntu@10.10.200.66
```

KH1 SSH vào VM — Ubuntu 22.04, 1 vCPU, ~2 GB RAM (`u1.small`). Network bên trong VM là NAT masquerade (`10.0.2.2/24`) do KubeVirt quản lý; KH1 chỉ cần biết IP ngoài `10.10.200.66` để SSH, không cần quan tâm đến IP nội bộ pod:

```
ubuntu@cust-cpu-vm:~$ ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp1s0           UP             10.0.2.2/24 metric 100

ubuntu@cust-cpu-vm:~$ free -h
               total        used        free      shared  buff/cache   available
Mem:           1.9Gi       194Mi       923Mi       1.0Mi       780Mi       1.5Gi

ubuntu@cust-cpu-vm:~$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda     252:0    0   10G  0 disk
├─vda1  252:1    0  9.9G  0 part /
├─vda14 252:14   0    4M  0 part
└─vda15 252:15   0  106M  0 part /boot/efi
```

Serial console (recover khi network mất, không cần GUI):

```bash
virtctl console cust-cpu-vm -n cust-cpu --kubeconfig=/root/kubeconfigs/cust-cpu.kubeconfig
# Thoát: Ctrl+]
```

### 12.4 KH2 — cust-gpu-whole mua VM Windows Server 2022 + 1 GPU nguyên

Khách hàng: studio làm 3D rendering, cần Windows VM với GPU full power cho phần mềm như Blender, Maya, V-Ray.

KH2 dùng `cust-gpu-whole.kubeconfig`:

```bash
export KUBECONFIG=~/kubeconfigs/cust-gpu-whole.kubeconfig
```

PVC `windows-2022-rootdisk` đã được admin clone sẵn từ golden image vào namespace `cust-gpu-whole` ở bước 12.2.

Bước 1: KH2 tạo VM Windows (không cần ISO — OS đã có trong PVC):

```bash
kubectl apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: cust-gpu-whole-win
  namespace: cust-gpu-whole
  labels:
    customer: cust-gpu-whole
    workload-type: windows-gpu-vm
spec:
  runStrategy: Manual
  template:
    metadata:
      labels:
        customer: cust-gpu-whole
        kubevirt.io/vm: cust-gpu-whole-win
    spec:
      nodeSelector:
        gpu-pool: whole
      domain:
        cpu:
          cores: 4
          sockets: 1
          threads: 1
        memory:
          guest: 6Gi
        firmware:
          uuid: "6c8a93f1-2026-05-27-aaaa-b1b2b3b4b5b6"
          bootloader:
            efi:
              secureBoot: false
        clock:
          timezone: "Asia/Ho_Chi_Minh"
          timer:
            hpet:
              present: false
            hyperv: {}
            pit:
              tickPolicy: delay
            rtc:
              tickPolicy: catchup
        features:
          acpi: {}
          apic: {}
          smm:
            enabled: true
          hyperv:
            relaxed: {}
            vapic: {}
            spinlocks:
              spinlocks: 8191
        devices:
          tpm: {}
          disks:
            - name: rootdisk
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
              model: e1000
        resources:
          requests:
            nvidia.com/gpu: 1
          limits:
            nvidia.com/gpu: 1
      networks:
        - name: default
          pod: {}
      volumes:
        - name: rootdisk
          persistentVolumeClaim:
            claimName: windows-2022-rootdisk
EOF
```

Bước 2: Start VM — Windows boot thẳng từ golden image, chạy OOBE (setup SID mới sau sysprep):

```bash
virtctl --kubeconfig=/root/kubeconfigs/cust-gpu-whole.kubeconfig \
  start cust-gpu-whole-win -n cust-gpu-whole

kubectl get vmi -n cust-gpu-whole cust-gpu-whole-win
# NAME                 AGE   PHASE     IP           NODENAME   READY
# cust-gpu-whole-win   49s   Running   10.42.1.92   ctrl02     True
```

Mở VNC để hoàn tất OOBE (chọn region, tạo Administrator password — ~2 phút):

```bash
virtctl --kubeconfig=/root/kubeconfigs/cust-gpu-whole.kubeconfig \
  vnc cust-gpu-whole-win -n cust-gpu-whole --proxy-only --port=5900 --address=0.0.0.0
```

Bước 3: Enable RDP (PowerShell as Administrator trong VM):

```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
Restart-Service -Name TermService -Force
```

Bước 4: Expose RDP qua Floating IP `10.10.200.67`:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: cust-gpu-whole-win-rdp
  namespace: cust-gpu-whole
  annotations:
    io.cilium/lb-ipam-ips: "10.10.200.67"
spec:
  type: LoadBalancer
  selector:
    customer: cust-gpu-whole
    kubevirt.io/vm: cust-gpu-whole-win
  ports:
    - name: rdp
      port: 3389
      targetPort: 3389
      protocol: TCP
EOF
```

Sau khi RDP Service đã có EXTERNAL-IP, tắt VNC listener trên ctrl01 (Ctrl+C trong terminal đang chạy virtctl, hoặc kill process):

```bash
# Trên ctrl01 — tắt VNC proxy
pkill -f "virtctl.*vnc.*proxy-only"
```

Bước 5: KH2 RDP từ máy Windows / Mac / Linux:

```
# Trên máy KH2
mstsc /v:10.10.200.67       # Windows Remote Desktop
# Hoặc: Microsoft Remote Desktop trên Mac
# Hoặc: rdesktop / xfreerdp trên Linux

User: Administrator
Password: 
```

Login thành công vào Windows Server 2022 desktop — full UI, có thể mở Server Manager, Device Manager, install application, chạy phần mềm.

**Kiểm tra GPU trong Windows Device Manager — và limitation của lab:**

Mở Device Manager (`devmgmt.msc`):

```
Display adapters
  └─ QEMU Standard VGA
```

Đây là **expected behavior của fake-gpu-operator**:
- VM xin GPU ở K8s level ✅ (kubectl describe vmi show)
- Capsule deduct quota ✅
- VM land trên ctrl02 (whole pool) ✅
- Nhưng Device Manager **không thấy NVIDIA GPU** vì fake operator không thực hiện PCI passthrough qua VFIO

Để xác nhận GPU đã được allocate ở orchestration layer (KH2 hoặc admin có thể verify):

```bash
# Admin view
kubectl --kubeconfig=~/rke2.yaml describe vmi cust-gpu-whole-win -n cust-gpu-whole | grep -A2 "Resources:"
# Resources:
#   Limits:
#     Nvidia.com/gpu: 1
#   Requests:
#     Nvidia.com/gpu: 1

# KH2 cũng thấy được trong namespace của họ
kubectl --kubeconfig=~/kubeconfigs/cust-gpu-whole.kubeconfig describe vmi cust-gpu-whole-win | grep -i nvidia
```

Trong KubeVirt Manager UI (http://10.10.200.62, admin view): VM `cust-gpu-whole-win` hiển thị với badge "GPU: 1", node ctrl02.

**Production path:** thay `fake-gpu-operator` bằng **NVIDIA GPU Operator** + GPU thật trên ctrl02 + VFIO passthrough enabled. Khi đó VM Windows sẽ thấy A100/H100 thật trong Device Manager → nvidia-smi.exe work → CUDA workload chạy được. **Tất cả manifest VM, Capsule, nodeSelector topology ở lab này giữ nguyên không đổi.**

Đó là điểm mạnh của setup này: orchestration layer là production-ready, chỉ cần swap operator để có GPU thật.

### 12.5 KH3 — cust-gpu-shared mua 1 container + GPU chia nhỏ

Khách hàng: ML engineer làm inference NLP, chỉ cần fractional GPU (1/4 A100 đủ chạy small model). Không cần VM full OS, container đủ.

KH3 dùng `cust-gpu-shared.kubeconfig`:

```bash
export KUBECONFIG=~/kubeconfigs/cust-gpu-shared.kubeconfig
```

Bước 1: KH3 deploy container CUDA base image với GPU shared:

```bash
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cust-gpu-shared-data
  namespace: cust-gpu-shared
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: rook-ceph-block
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cust-gpu-shared-app
  namespace: cust-gpu-shared
  labels:
    customer: cust-gpu-shared
    workload-type: inference-container
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cust-gpu-shared-app
  template:
    metadata:
      labels:
        app: cust-gpu-shared-app
        customer: cust-gpu-shared
    spec:
      # Capsule inject nodeSelector cho regular Pod (không phải virt-launcher) → land ctrl03
      containers:
        - name: cuda
          image: nvcr.io/nvidia/cuda:12.6.0-base-ubuntu22.04
          command: ["sh", "-c"]
          args:
            - |
              # Setup env như KH3 dev environment
              apt-get update -qq && apt-get install -qq -y openssh-server vim htop curl > /dev/null 2>&1
              mkdir -p /run/sshd
              # SSH password
              echo 'root:GpuShared2026' | chpasswd
              sed -i 's/#PermitRootLogin .*/PermitRootLogin yes/' /etc/ssh/sshd_config
              sed -i 's/#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
              # Demo: chạy nvidia-smi mỗi 60s, log
              (while true; do
                echo "=== \$(date) ==="
                nvidia-smi 2>/dev/null || echo "nvidia-smi simulated output (fake-gpu-operator)"
                sleep 60
              done) > /var/log/gpu-monitor.log 2>&1 &
              # Start SSH
              /usr/sbin/sshd -D
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
              nvidia.com/gpu: 1
            limits:
              cpu: 2
              memory: 2Gi
              nvidia.com/gpu: 1
          ports:
            - name: ssh
              containerPort: 22
          env:
            - name: NODE_NAME      # fake-gpu-operator yêu cầu env này
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: cust-gpu-shared-data
EOF

# Chờ pod running
kubectl get pods -n cust-gpu-shared -w
# NAME                                       READY   STATUS    RESTARTS   AGE
# cust-gpu-shared-app-xxxxx-xxxxx            1/1     Running   0          1m
```

Verify pod land đúng ctrl03 (shared GPU):

```bash
kubectl get pod -n cust-gpu-shared -o wide
# NAME                                  ...   NODE     ...
# cust-gpu-shared-app-xxxxx-xxxxx       ...   ctrl03   ...

kubectl describe pod -n cust-gpu-shared -l app=cust-gpu-shared-app | grep -A1 "Limits:\|Requests:" | head -8
# Limits:
#   cpu: 2
#   memory: 2Gi
#   nvidia.com/gpu: 1
# Requests:
#   cpu: 500m
#   memory: 1Gi
#   nvidia.com/gpu: 1
```

Bước 2: KH3 access container — 2 cách

**Cách 1 — kubectl exec** (KH3 đã có kubeconfig):

```bash
POD=$(kubectl get pod -n cust-gpu-shared -l app=cust-gpu-shared-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n cust-gpu-shared -it ${POD} -- bash

# Inside container:
root@cust-gpu-shared-app-xxxxx:/# nvidia-smi
# +-----------------------------------------------------------------------------+
# | NVIDIA-SMI 535.xxx       Driver Version: 535.xxx       CUDA Version: 12.6   |
# |-------------------------------+----------------------+----------------------+
# | GPU  Name                | ... | NVIDIA A100-SXM4-40GB | ...                |
# (fake-gpu-operator binary nvidia-smi giả lập)

root@cust-gpu-shared-app-xxxxx:/# cat /var/log/gpu-monitor.log | head -5
# === Thu May 27 03:15:21 UTC 2026 ===
# (nvidia-smi output)

root@cust-gpu-shared-app-xxxxx:/# nproc
2
root@cust-gpu-shared-app-xxxxx:/# df -h /data
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/rbd0       9.8G  280K  9.3G   1% /data
```

KH3 thấy đúng spec: 2 CPU limit, 2GB RAM, 1 GPU (slice), 10GB persistent volume `/data`.

**Cách 2 — SSH qua Floating IP:**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: cust-gpu-shared-ssh
  namespace: cust-gpu-shared
  annotations:
    io.cilium/lb-ipam-ips: "10.10.200.68"
spec:
  type: LoadBalancer
  selector:
    app: cust-gpu-shared-app
  ports:
    - name: ssh
      port: 22
      targetPort: 22
      protocol: TCP
EOF
```

```bash
# KH3 SSH từ máy của họ
ssh root@10.10.200.68
# password: GpuShared2026

root@cust-gpu-shared-app-xxxxx:~# nvidia-smi
# (output fake-gpu-operator)
```

Bước 3: Verify time-slicing hoạt động — KH3 có 1 slice, vẫn còn 7 slice cho khách khác:

```bash
# Trên admin
kubectl describe node ctrl03 | grep -A1 "Allocated resources:" | head -10
# Allocated resources:
#   nvidia.com/gpu  1   8
#   (1 used / 8 total)
```

Nếu tạo thêm tenant `cust-gpu-shared-2` và xin 1 GPU slice, sẽ schedule được trên cùng ctrl03 — đó chính là multi-tenant GPU sharing.

### 12.6 Verify cross-tenant isolation

Kiểm tra 3 tenant không thấy được nhau:

```bash
# KH1 thử list VM của KH2 → Forbidden
kubectl --kubeconfig=~/kubeconfigs/cust-cpu.kubeconfig \
  get vm -n cust-gpu-whole
# Error from server (Forbidden): virtualmachines.kubevirt.io is forbidden:
# User "system:serviceaccount:cust-cpu:cust-cpu-owner" cannot list resource
# "virtualmachines" in API group "kubevirt.io" in the namespace "cust-gpu-whole"

# KH3 thử exec vào pod tenant khác → Forbidden
kubectl --kubeconfig=~/kubeconfigs/cust-gpu-shared.kubeconfig \
  exec -n cust-gpu-whole some-pod -- ls
# Error from server (Forbidden): ...

# KH2 thử tạo VM lên ctrl03 (vi phạm nodeSelector) → Capsule webhook block
cat > /tmp/wrong-node-vm.yaml <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: hack-attempt
  namespace: cust-gpu-whole
spec:
  runStrategy: Halted
  template:
    spec:
      nodeSelector:
        gpu-pool: shared    # KH2 thuộc pool whole, không được chọn shared
      domain:
        devices: {}
        resources:
          requests:
            memory: 256Mi
      volumes: []
EOF

kubectl --kubeconfig=~/kubeconfigs/cust-gpu-whole.kubeconfig \
  apply -f /tmp/wrong-node-vm.yaml
# Error from server: admission webhook "tenants.capsule.clastix.io" denied the request:
# nodeSelector key "gpu-pool" must match tenant's allowed value: "whole"
```

Capsule webhook chặn cross-pool. KH2 chỉ được dùng `gpu-pool: whole`. Đây là isolation thật, không phải fake do cluster-admin.

Verify quota enforcement — KH3 thử tạo container thứ 2 xin GPU:

```bash
kubectl apply --kubeconfig=~/kubeconfigs/cust-gpu-shared.kubeconfig -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: quota-test
  namespace: cust-gpu-shared
spec:
  containers:
    - name: cuda
      image: nvcr.io/nvidia/cuda:12.6.0-base-ubuntu22.04
      command: ["sleep","3600"]
      resources:
        limits:
          nvidia.com/gpu: 1
EOF
# Error from server (Forbidden): pods "quota-test" is forbidden:
# exceeded quota: capsule-cust-gpu-shared-...,
# requested: requests.nvidia.com/gpu=1, used: requests.nvidia.com/gpu=1, limited: requests.nvidia.com/gpu=1
```

KH3 đã hết quota 1 GPU. Để mua thêm, KH3 phải request admin tăng quota (hoặc trong production: tự upgrade qua billing portal).

Tổng quan cluster sau khi 3 KH lên hoàn chỉnh:

```bash
kubectl get vmi,pods -A -o wide | grep -E "cust-|customer" | grep -v Completed
# NAMESPACE          NAME                                              NODE
# cust-cpu           virtualmachineinstance.kubevirt.io/cust-cpu-vm    ctrl01
# cust-gpu-whole     virtualmachineinstance.kubevirt.io/cust-gpu-whole-win   ctrl02
# cust-gpu-shared    pod/cust-gpu-shared-app-xxxxx-xxxxx               ctrl03

kubectl top node
# NAME     CPU(cores)   MEMORY(bytes)
# ctrl01   850m         11200Mi   ← VM Ubuntu KH1 ~4GB + system
# ctrl02   1200m        13800Mi   ← VM Windows KH2 ~8GB + system
# ctrl03   650m         9500Mi    ← Container KH3 1GB + system
```

Mỗi node phục vụ đúng nhóm khách hàng tương ứng. GPU node (ctrl02, ctrl03) bận với workload có GPU; CPU node (ctrl01) chỉ phục vụ workload không GPU. Đây là pattern thực tế của GPU cloud thương mại — tận dụng phần cứng GPU đắt tiền cho workload cần GPU, không lãng phí cho CPU-only.

---

## 13. Service URLs & Credentials

Tổng hợp endpoint của lab:

| Service | URL / Endpoint | Credentials |
|---|---|---|
| Kubernetes API (VIP) | https://10.10.200.51:6443 | Admin: `~/rke2.yaml`; Tenant: `~/kubeconfigs/<tenant>.kubeconfig` |
| Ceph Dashboard | http://10.10.200.61:7000 | `admin` / `kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}"\|base64 -d` |
| KubeVirt Manager | http://10.10.200.62 | (SA cluster-admin, admin-only) |
| Hubble UI | `cilium hubble ui` (port-forward) | (no auth — lab) |
| **KH1 — cust-cpu (Ubuntu VM SSH)** | `ssh ubuntu@10.10.200.66` | `ubuntu` / `<your-password>` (set trong cloud-init section 12.3) |
| **KH2 — cust-gpu-whole (Windows RDP)** | `mstsc /v:10.10.200.67` | `Administrator` / `<your-oobe-password>` (set trong OOBE section 12.4) |
| **KH3 — cust-gpu-shared (Container SSH)** | `ssh root@10.10.200.68` | `root` / `GpuShared2026` |

Recovery commands quan trọng:

```bash
# Lấy kubeconfig admin từ ctrl01
cat /etc/rancher/rke2/rke2.yaml > ~/rke2.yaml
sed -i 's|127.0.0.1|10.10.200.51|' ~/rke2.yaml

# Reset password Ceph dashboard
ADMIN_PW='NewSecurePw2026!'
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- bash -c \
  "echo -n '${ADMIN_PW}' > /tmp/pw && ceph dashboard ac-user-set-password admin -i /tmp/pw"

# Lấy token tenant (dùng với kubectl, virtctl)
kubectl -n cust-gpu-shared get secret cust-gpu-shared-owner-token -o jsonpath='{.data.token}' | base64 -d
```

Network port quan trọng phải open trong firewall (nếu có) giữa node:

| Port | Protocol | Mục đích |
|---|---|---|
| 6443 | TCP | Kubernetes API |
| 9345 | TCP | RKE2 supervisor (node join) |
| 2379-2380 | TCP | etcd peer + client |
| 10250 | TCP | kubelet metrics |
| 8472 | UDP | VXLAN (Cilium tunnel) |
| 4240 | TCP | Cilium health check |
| 4244 | TCP | Hubble |
| 3300 | TCP | Ceph mon |
| 6789 | TCP | Ceph mon legacy |
| 6800-7300 | TCP | Ceph OSD |

Lab này dùng `disable: ufw` nên không cần manual rule, nhưng production phải audit firewall.

---

## 14. Lời kết

Lab này đã chỉ ra full pattern của một Next-Gen GPU Cloud thương mại trên Kubernetes:

**Cái lab demo được:**

1. **HA Kubernetes 3-node** với etcd quorum + kube-vip VIP API + Cilium L2 Announcements (không cần MetalLB)
2. **Distributed storage** Rook-Ceph với replication=3 an toàn, 3 StorageClass cho 3 use case (RBD RWO, RBD block-mode RWX, CephFS), dashboard
3. **VM lẫn container** trên cùng cluster qua KubeVirt \+ CDI — cùng scheduler, cùng network, cùng storage
4. **GPU multi-pool topology**: phân bổ 3 node theo loại khách hàng (CPU-only, whole, shared) — đúng cách GPU cloud thực tế tận dụng phần cứng
5. **Multi-tenancy hard isolation** qua Capsule với **custom ClusterRole `tenant-owner`** không cluster-admin, NetworkPolicy auto, nodeSelector inject, ResourceQuota hard limit
6. **Observability**: Hubble network visibility + Ceph dashboard
7. **KubeVirt Manager** — admin portal quản lý VM, console VNC, live migration
8. **Golden image workflow** — cài OS một lần, clone PVC cho mỗi KH mới (<30 giây/lần)
9. **3 chân dung khách hàng** chạy song song với 3 loại workload + 3 cách access (SSH, RDP, kubectl exec) — đúng business model GPU cloud

**Cái lab KHÔNG demo được (giới hạn fake-gpu-operator):**

- Windows Device Manager không thấy NVIDIA GPU (chỉ "QEMU Standard VGA")
- `nvidia-smi.exe` trong Windows không có device thật
- CUDA workload trong VM/container không tính toán thật, chỉ giả lập binary

Với 3 VM Ubuntu 24.04 trên ESXi và ~16GB RAM mỗi máy, lab này có thể chạy được trong vài giờ và demo full pattern của một GPU cloud production. Bài học quan trọng nhất: **Kubernetes thế hệ 2026 đủ trưởng thành để chạy production virtualization + GPU workload thay OpenStack**, đặc biệt khi KubeVirt đã có PCIe NUMA awareness và HAL chuẩn bị cho multi-hypervisor.

---


