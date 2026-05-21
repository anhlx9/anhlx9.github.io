---
title: " Private cloud với OpenStack 2025.1 + OVN + Ceph Squid — 3 Node All-in-One"
categories:
- OpenStack
tags:
- openstack
- ovn
- ceph
- kolla-ansible
- private-cloud
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### OpenStack 2025.1 Epoxy + OVN + Ceph Squid — 3 ctrl node all-in-one (Controller + Compute + OSD) + 1 Registry, deploy bằng kolla-ansible
---

Lab này kết hợp 3 thành phần: **OpenStack 2025.1 (Epoxy)** làm cloud orchestration, **Ceph Squid** làm distributed storage backend (Cinder block + Glance image + Nova ephemeral qua RBD), và **OVN** xử lý toàn bộ virtual networking — distributed L2/L3, security groups, floating IP — mà không cần thêm agent hay process bên ngoài. Toàn bộ deploy qua `kolla-ansible` chính thức.

Compact version này gom toàn bộ role vào 3 node — mỗi `ctrl01–03` chạy đồng thời OpenStack control plane, Nova KVM, OVN Controller và Ceph OSD — phù hợp cho private cloud team nhỏ, remote office, hoặc lab tài nguyên hạn chế muốn có HA control plane.

Stack gồm 4 VMs Ubuntu 24.04, deploy qua `kolla-ansible` 2025.1:

1. `registry01` — local Docker Registry v2, mirror toàn bộ images trước khi deploy
2. `ctrl01–03` — all-in-one: OpenStack HA + OVN + Ceph mon/mgr/OSD + Nova KVM, `ctrl01` kiêm deploy node

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/001.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/002.png"/>

---

### Mục lục

- [1. Stack \& Kiến trúc](#1-stack--kiến-trúc)
  - [1.1 Stack phiên bản](#11-stack-phiên-bản)
  - [1.2 Topology 4 nodes](#12-topology-4-nodes)
  - [1.3 Phân bổ RAM 16 GB per node](#13-phân-bổ-ram-16-gb-per-node)
  - [1.4 Trade-off AIO cần biết](#14-trade-off-aio-cần-biết)
  - [1.5 Luồng traffic OVN](#15-luồng-traffic-ovn)
- [2. Node Layout, Tài nguyên \& IP Planning](#2-node-layout-tài-nguyên--ip-planning)
  - [2.1 Tài nguyên](#21-tài-nguyên)
  - [2.2 IP planning](#22-ip-planning)
- [3. Base OS (Tất cả nodes)](#3-base-os-tất-cả-nodes)
  - [3.1 Set hostname + IP Config](#31-set-hostname--ip-config)
  - [3.2 Swap OFF](#32-swap-off)
  - [3.3 NTP (Chrony)](#33-ntp-chrony)
  - [3.4 Kernel params](#34-kernel-params)
  - [3.5 Base packages](#35-base-packages)
  - [3.6 /etc/hosts](#36-etchosts)
  - [3.7 Docker daemon — insecure local registry](#37-docker-daemon--insecure-local-registry)
- [4. Local Docker Registry (registry01)](#4-local-docker-registry-registry01)
  - [4.1 Khởi chạy Registry v2](#41-khởi-chạy-registry-v2)
  - [4.2 Pull \& push images Kolla + Ceph](#42-pull--push-images-kolla--ceph)
  - [4.3 Verify từ node khác](#43-verify-từ-node-khác)
- [5. Ceph Hyperconverged (cephadm) — trên ctrl01-03](#5-ceph-hyperconverged-cephadm--trên-ctrl01-03)
  - [5.1 Bootstrap từ ctrl01](#51-bootstrap-từ-ctrl01)
  - [5.2 Add host \& deploy OSD](#52-add-host--deploy-osd)
  - [5.3 Tạo pools \& keyrings cho OpenStack](#53-tạo-pools--keyrings-cho-openstack)
- [6. Setup kolla-ansible trên ctrl01](#6-setup-kolla-ansible-trên-ctrl01)
- [7. globals.yml \& inventory multinode](#7-globalsyml--inventory-multinode)
- [8. Copy Ceph configs vào Kolla](#8-copy-ceph-configs-vào-kolla)
- [9. Deploy OpenStack](#9-deploy-openstack)
- [10. Service URLs \& Credentials](#10-service-urls--credentials)
- [11. Workload đầu tiên — Demo Private Cloud multi tenant](#11-workload-đầu-tiên--demo-private-cloud-multi-tenant)
  - [11.1 Admin — Projects, users và shared resources](#111-admin--projects-users-và-shared-resources)
  - [11.2 Team IT — Ubuntu VM](#112-team-it--ubuntu-vm)
  - [11.3 Team Dev — Ubuntu VM](#113-team-dev--ubuntu-vm)
  - [11.4 Verify isolation](#114-verify-isolation)
- [12. Lời kết](#12-lời-kết)

---

## 1. Stack & Kiến trúc

### 1.1 Stack phiên bản

Mình dùng kolla-ansible chính thức từ PyPI — host OS và container base image đều là Ubuntu 24.04 Noble.

| Thành phần | Phiên bản | Ghi chú |
|---|---|---|
| **OpenStack** | **2025.1 (Epoxy)** | kolla-ansible chính thức |
| **OVN** | **bundled với Neutron** | `neutron_plugin_agent: ovn` trong globals.yml |
| **Kolla images** | `2025.1-ubuntu-noble` | `quay.io/openstack.kolla/<service>:2025.1-ubuntu-noble` |
| **Ubuntu host** | **24.04 LTS Noble** | Kolla Epoxy yêu cầu tối thiểu Noble |
| **Ceph** | **Squid 19.x** | `quay.io/ceph/ceph:v19` — deploy bằng `cephadm` |
| **Python** | 3.12 | Mặc định Ubuntu 24.04, trên ctrl01 |
| **Ansible** | 10.x (ansible-core 2.17) | Theo kolla-ansible requirements |
| **Docker** | 29.x | `docker-ce` từ Docker official repo |

### 1.2 Topology 4 nodes

```
┌──────────────────────────────────────────────────────────────────┐
│                  Hypervisor (ESXi)  —  ~60 GB RAM                │
│                                                                  │
│  ┌──────────────┐  REGISTRY                                      │
│  │ registry01   │  Docker Registry v2 (mirror quay.io)           │
│  │ 2vCPU, 4 GB  │  10.10.200.5                                   │
│  └──────────────┘                                                │
│                                                                  │
│  ─────── 3 NODE ALL-IN-ONE (Controller + Compute + Storage) ───  │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │  ctrl01     │  │   ctrl02    │  │   ctrl03    │               │
│  │  Deploy     │  │             │  │             │               │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤               │
│  │ OS Control  │  │ OS Control  │  │ OS Control  │  ← OpenStack  │
│  │ OVN NB/SB   │  │ OVN NB/SB   │  │ OVN NB/SB   │    HA         │
│  │ Ceph mon/mgr│  │ Ceph mon/mgr│  │ Ceph mon    │  + OVN SDN    │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤               │
│  │ Nova KVM    │  │ Nova KVM    │  │ Nova KVM    │  ← Compute    │
│  │ OVN Ctrl    │  │ OVN Ctrl    │  │ OVN Ctrl    │               │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤               │
│  │ Ceph OSD    │  │ Ceph OSD    │  │ Ceph OSD    │  ← Storage    │
│  │ /dev/sdb    │  │ /dev/sdb    │  │ /dev/sdb    │               │
│  │ 8vCPU,16 GB │  │ 8vCPU,16 GB │  │ 8vCPU,16 GB │               │
│  └─────────────┘  └─────────────┘  └─────────────┘               │
│                                                                  │
│  ╔══════════════════════════════════════════════════════════╗    │
│  ║  Management / API:   10.10.200.0/24                      ║    │
│  ║  VIP OpenStack API:  10.10.200.50 (HAProxy + Keepalived) ║    │
│  ╚══════════════════════════════════════════════════════════╝    │
│  ╔══════════════════════════════════════════════════════════╗    │
│  ║  Geneve Tunnel / Ceph:  10.10.201.0/24                   ║    │
│  ║  OVN Geneve tunnel + Ceph OSD replication                ║    │
│  ╚══════════════════════════════════════════════════════════╝    │
│  ╔══════════════════════════════════════════════════════════╗    │
│  ║  Provider / External:  ens224 (no IP, VLAN 200)          ║    │
│  ║  OVN br-ex — floating IP pool 10.10.200.61–69            ║    │
│  ╚══════════════════════════════════════════════════════════╝    │
└──────────────────────────────────────────────────────────────────┘

Resource tổng:
  vCPU: 2 + 3×8 = 26 vCPU
  RAM:  4 + 3×16 = 52 GB
  Disk OS:  200 GB (registry01) + 3 × 100 GB = 500 GB
  Disk OSD: 3 × 200 GB = 600 GB (raw) → usable ~300 GB (replication size=2)
```

> `ctrl01–03` cần **3 NIC**: `ens160` (mgmt/API), `ens192` (Geneve tunnel + Ceph), `ens224` (provider/external — no IP, OVN tạo bridge `br-ex` trên interface này).

### 1.3 Phân bổ RAM 16 GB per node

Phần quan trọng nhất khi làm AIO là hiểu RAM đang đi đâu. OVN footprint nhỏ — toàn bộ SDN layer chỉ chiếm ~1 GB, phần lớn RAM dành cho OpenStack services và VM tenants:

| Service group | Ước tính |
|---|---|
| OS + system daemons | ~0.5 GB |
| OpenStack services (MariaDB, RabbitMQ, Memcached, HAProxy, Keystone, Glance, Cinder, Nova api/scheduler/conductor, Heat, Horizon, Barbican, Placement) | ~4–5 GB |
| Prometheus + Grafana | ~1 GB |
| OVN (ovn-northd, ovn-nb-db, ovn-sb-db, ovn-controller) | ~0.5 GB |
| Neutron Server + Metadata Agent | ~0.5 GB |
| Ceph OSD + mon/mgr | ~1 GB |
| Nova compute + libvirt | ~0.5 GB |
| **Khả dụng cho VM tenant** | **~7–8 GB** |

> Tổng ~21–24 GB khả dụng toàn cluster — đủ chạy nhiều VM nhỏ đến vừa để lab.

### 1.4 Trade-off AIO cần biết

Mình liệt kê rõ để dễ quyết định trước khi bắt tay lab:

| Điểm | Ghi chú |
|---|---|
| HA control plane | 3 node → Galera/RabbitMQ/Ceph mon quorum, mất 1 node vẫn OK |
| OVN NB/SB HA | RAFT cluster 3 node — mất 1 node vẫn hoạt động |
| Blast radius | Mất 1 node = mất 1/3 compute + 1/3 storage đồng thời |
| Noisy neighbor | Nova KVM và OVN Controller chia sẻ core với OpenStack API |
| Ceph OSD trên ctrl | OSD replication traffic dùng `ens192` riêng — không ảnh hưởng management |
| VM capacity | ~21–24 GB khả dụng toàn cluster — lab OK, không dùng cho production nặng |

### 1.5 Luồng traffic OVN

```
VM tenant
   │
   ▼
OVS (Open vSwitch) — local br-int trên ctrl node
   │  Geneve tunnel (ens192 — 10.10.201.x)
   ▼
OVN Controller (ovn-controller)  ← sync flows từ OVN SB DB
   │  OVSDB protocol
   ▼
OVN Southbound DB  ←→  OVN Northbound DB  ←→  Neutron Server
                              │
                         OVN Northd
                  (compile logical → physical flows)

Floating IP:
VM → OVS br-int → OVN DNAT/SNAT → OVS br-ex (ens224) → external network
```

> OVN xử lý L2 (switching), L3 (routing), security groups và DNAT/SNAT cho floating IP — tất cả dưới dạng OpenFlow rules trên OVS, không cần Linux network namespace hay iptables chain như Neutron-OVS cũ.

---

## 2. Node Layout, Tài nguyên & IP Planning

### 2.1 Tài nguyên

Mình dùng template ubuntu-24.04.4 với disk thứ hai `/dev/sdb` riêng cho Ceph OSD. `ctrl01–03` cần 3 NIC — `ens224` không cần IP, OVN sẽ tạo bridge `br-ex` trên interface này.

| # | Hostname | Role | vCPU | RAM | Disk OS | Disk Ceph | NIC |
|---|---|---|---|---|---|---|---|
| 0 | `registry01` | Local Docker Registry | 2 | 4 GB | 200 GB | — | 1 |
| 1 | `ctrl01` | Controller + **Deploy** + OVN + Ceph mon/mgr + **Nova KVM** + **Ceph OSD** | 8 | 16 GB | 100 GB | 200 GB | 3 |
| 2 | `ctrl02` | Controller + OVN + Ceph mon/mgr + Nova KVM + Ceph OSD | 8 | 16 GB | 100 GB | 200 GB | 3 |
| 3 | `ctrl03` | Controller + OVN + Ceph mon + Nova KVM + Ceph OSD | 8 | 16 GB | 100 GB | 200 GB | 3 |
| — | **TỔNG** | | **26 vCPU** | **52 GB** | **500 GB** | **600 GB** | |

### 2.2 IP planning

Mình dùng 3 NIC trên ctrl nodes: `ens160` management/API, `ens192` tunnel+Ceph, `ens224` provider network (no IP).

| Hostname | ens160 (MGMT 200.x) | ens192 (TUNNEL 201.x) | ens224 (PROVIDER) |
|---|---|---|---|
| registry01 | 10.10.200.5 | — | — |
| ctrl01 | 10.10.200.11 | 10.10.201.11 | no IP |
| ctrl02 | 10.10.200.12 | 10.10.201.12 | no IP |
| ctrl03 | 10.10.200.13 | 10.10.201.13 | no IP |
| **VIP OpenStack API** | **10.10.200.50** | — | — |
| **Floating IP Pool** | **10.10.200.61–69** | — | — |
| Gateway | 10.10.200.1 | — | — |

| Network | Subnet | Traffic |
|---|---|---|
| Management / API | 10.10.200.0/24 | SSH, Kolla deploy, OpenStack API, Horizon |
| Geneve Tunnel / Ceph | 10.10.201.0/24 | OVN Geneve tunnel, Ceph OSD replication |
| Provider / External | VLAN 200 via ens224 | OVN br-ex — floating IP, external VM access |
| Tenant overlay | 192.168.10.0/24 | Tạo sau deploy qua Neutron API |

---

## 3. Base OS (Tất cả nodes)

Mình cài Ubuntu 24.04 LTS minimal trên 4 VMs (chọn `openssh-server` ở phase select packages), sau đó thực hiện các bước sau trên từng node.

### 3.1 Set hostname + IP Config

```bash
hostnamectl set-hostname ctrl01   # thay: registry01 | ctrl01 | ctrl02 | ctrl03
```

```bash
network:
  version: 2
  ethernets:
    ens160:
      dhcp4: false
      dhcp6: false
      addresses:
        - "10.10.200.11/24"
      routes:
      - to: "default"
        via: "10.10.200.1"
      nameservers:
        addresses: [8.8.8.8]
    ens192:
      dhcp4: false
      dhcp6: false
      addresses:
        - "10.10.201.11/24"
    ens224:
      dhcp4: false
      dhcp6: false
```
```
sed -i 's|http://archive.ubuntu.com/ubuntu|http://mirror.bizflycloud.vn/ubuntu|g; s|http://security.ubuntu.com/ubuntu|http://mirror.bizflycloud.vn/ubuntu|g' /etc/apt/sources.list.d/ubuntu.sources
```

### 3.2 Swap OFF

OpenStack và Ceph đều yêu cầu tắt swap — mình tắt luôn và xóa khỏi fstab để không bị bật lại sau reboot:

```bash
swapoff -a
sed -i '/swap/d' /etc/fstab
free -h   # Swap: phải là 0B
```

### 3.3 NTP (Chrony)

```bash
apt install -y chrony
timedatectl set-timezone Asia/Ho_Chi_Minh
systemctl enable --now chrony
chronyc tracking
```

### 3.4 Kernel params

```bash
cat >> /etc/sysctl.conf <<'EOF'
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.core.somaxconn = 1024
net.ipv4.tcp_max_syn_backlog = 2048
EOF
sysctl -p
```

### 3.5 Base packages

Mình cài Docker CE từ official repo thay vì để kolla-ansible tự bootstrap — cách này tránh conflict phiên bản và giữ daemon.json dưới kiểm soát:

```bash
apt update
apt install -y \
  python3 python3-pip python3-venv python3-dev \
  openssh-server openssh-client \
  net-tools curl wget git jq \
  vim htop tmux iotop \
  build-essential libssl-dev libffi-dev \
  qemu-guest-agent \
  ca-certificates gnupg

# Docker CE — official repo
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker

# OVS — OVN Controller cần ovsdb-server chạy trên host (chỉ ctrl nodes)
apt install -y openvswitch-switch
systemctl enable --now openvswitch-switch

```

### 3.6 /etc/hosts

```bash
cat >> /etc/hosts <<'EOF'
10.10.200.5    registry01
10.10.200.11   ctrl01
10.10.200.12   ctrl02
10.10.200.13   ctrl03
10.10.200.50   openstack-vip
EOF
```

### 3.7 Docker daemon — insecure local registry

```bash
cat > /etc/docker/daemon.json <<'EOF'
{
  "insecure-registries": ["10.10.200.5:5000"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" },
  "storage-driver": "overlay2"
}
EOF

systemctl restart docker
docker info | grep -A2 "Insecure Registries"
```

---

## 4. Local Docker Registry (registry01)

Mình pull images một lần duy nhất từ `quay.io`, lưu vào `registry01:5000`. 3 ctrl nodes pull từ đây — tránh rate-limit và kéo chậm lúc deploy. Tổng 54 images: 53 Kolla 2025.1 + 1 Ceph Squid.

### 4.1 Khởi chạy Registry v2

```bash
# Trên registry01 (10.10.200.5)
mkdir -p /opt/registry/data
cat > /opt/registry/docker-compose.yml <<'EOF'
services:
  registry:
    image: registry:2
    container_name: registry
    restart: always
    ports:
      - "5000:5000"
    volumes:
      - /opt/registry/data:/var/lib/registry
    environment:
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
      REGISTRY_LOG_LEVEL: info
EOF

cd /opt/registry
docker compose up -d
docker compose ps

curl http://localhost:5000/v2/_catalog       # → {"repositories":[]}
```

### 4.2 Pull & push images Kolla + Ceph

Mình chia script thành 2 phần: Ceph từ quay.io và Kolla images từ quay.io. Mỗi image pull → tag → push → rmi local để tiết kiệm disk trên registry01:

```bash
# Chạy trên registry01
set -e
LOCAL_REG="10.10.200.5:5000"

# ── PHẦN 1 — Ceph Squid v19 ───────────────────────────────────────────────
# cephadm dùng 1 image cho TẤT CẢ daemon types (mon, mgr, osd, crash, mds)
docker pull  quay.io/ceph/ceph:v19
docker tag   quay.io/ceph/ceph:v19 "${LOCAL_REG}/ceph/ceph:v19"
docker push  "${LOCAL_REG}/ceph/ceph:v19"
docker rmi   quay.io/ceph/ceph:v19 "${LOCAL_REG}/ceph/ceph:v19" || true

# ── PHẦN 2 — Kolla images  quay.io/openstack.kolla ───────────────────────
KOLLA_SRC="quay.io/openstack.kolla"
KOLLA_TAG="2025.1-ubuntu-noble"
KOLLA_NS="openstack.kolla"
kolla_push() {
  local img=$1
  local max=3 n=0
  until docker pull "${KOLLA_SRC}/${img}:${KOLLA_TAG}"; do
    n=$((n+1)); [ $n -ge $max ] && { echo "FAIL: $img after $max tries"; return 1; }
    echo "Retry $n/$max for $img ..."; sleep 5
  done
  docker tag   "${KOLLA_SRC}/${img}:${KOLLA_TAG}" "${LOCAL_REG}/${KOLLA_NS}/${img}:${KOLLA_TAG}"
  docker push  "${LOCAL_REG}/${KOLLA_NS}/${img}:${KOLLA_TAG}"
  docker rmi   "${KOLLA_SRC}/${img}:${KOLLA_TAG}" "${LOCAL_REG}/${KOLLA_NS}/${img}:${KOLLA_TAG}" || true
}

# Infra
for img in kolla-toolbox cron fluentd; do kolla_push "$img"; done
# DB / MQ / Cache
for img in mariadb-server mariadb-clustercheck rabbitmq memcached proxysql valkey-server valkey-sentinel; do kolla_push "$img"; done
# HA / LB
for img in haproxy haproxy-ssh keepalived; do kolla_push "$img"; done
# Keystone
for img in keystone keystone-fernet keystone-ssh; do kolla_push "$img"; done
# Placement
kolla_push placement-api
# Nova
for img in nova-api nova-scheduler nova-conductor nova-compute nova-libvirt nova-novncproxy nova-ssh; do kolla_push "$img"; done
# Glance
kolla_push glance-api
# Cinder
for img in cinder-api cinder-volume cinder-scheduler cinder-backup; do kolla_push "$img"; done
# Neutron
for img in neutron-server neutron-metadata-agent; do kolla_push "$img"; done
# OVN — 5 images riêng biệt: controller, northd, nb-db-server, sb-db-server, sb-db-relay
for img in ovn-controller ovn-northd ovn-nb-db-server ovn-sb-db-server ovn-sb-db-relay; do kolla_push "$img"; done
# Heat
for img in heat-api heat-api-cfn heat-engine; do kolla_push "$img"; done
# Horizon
kolla_push horizon
# Barbican
for img in barbican-api barbican-keystone-listener barbican-worker; do kolla_push "$img"; done
# Prometheus + Grafana
for img in prometheus-server prometheus-node-exporter prometheus-mysqld-exporter \
           prometheus-memcached-exporter prometheus-cadvisor prometheus-alertmanager \
           prometheus-openstack-exporter prometheus-blackbox-exporter \
           prometheus-libvirt-exporter grafana; do
  kolla_push "$img"
done

TOTAL=$(curl -s "http://${LOCAL_REG}/v2/_catalog" | python3 -c 'import sys,json;print(len(json.load(sys.stdin)["repositories"]))')
echo "Done. Total repos: ${TOTAL}  (expect 54)"
```

### 4.3 Verify từ node khác

```bash
# Trên ctrl01
ping -c2 registry01
curl -s http://10.10.200.5:5000/v2/_catalog | jq '.repositories | length'  # → 54
docker pull 10.10.200.5:5000/openstack.kolla/keystone:2025.1-ubuntu-noble
```

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/01.png"/>

---

## 5. Ceph Hyperconverged (cephadm) — trên ctrl01-03

Mình đặt OSD trực tiếp trên ctrl node để tận dụng `/dev/sdb` 100 GB mỗi node. Thiết kế Ceph cho lab này:

- 3 monitors + 2 managers trên ctrl01-03 → quorum HA
- 3 OSD (1 OSD / ctrl node) trên `/dev/sdb` 200 GB → hyperconverged
- Replication `size=2`, `min_size=1` → usable ≈ 300 GB (600 GB raw / 2)

### 5.1 Bootstrap từ ctrl01

Mình bootstrap Ceph từ ctrl01 — node này sẽ là mon/mgr chính. 

```bash
# Trên ctrl01 (10.10.200.11)
apt install -y cephadm ceph-common

cephadm --image 10.10.200.5:5000/ceph/ceph:v19 bootstrap \
  --mon-ip 10.10.201.11 \
  --cluster-network 10.10.201.0/24 \
  --initial-dashboard-user admin \
  --initial-dashboard-password 'admin' \
  --skip-monitoring-stack \
  --allow-mismatched-release

ceph -s      # 1 mon up (ctrl01), 0 OSD, HEALTH_WARN — bình thường

# Verify bootstrap dùng image local
docker images | grep ceph

# Set lại container_image_base TRƯỚC khi add host
ceph config set mgr mgr/cephadm/container_image_base 10.10.200.5:5000/ceph/ceph
ceph config get mgr mgr/cephadm/container_image_base
# → 10.10.200.5:5000/ceph/ceph  ← phải là local registry

# Dashboard bind 0.0.0.0 → truy cập https://10.10.200.11:8443  (admin/admin)
echo -n 'admin' | ceph dashboard ac-user-set-password admin --force-password -i -
```

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/02.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/03.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/04.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/05.png"/>

### 5.2 Add host & deploy OSD

Mình pause orchestrator trước khi add host để ngăn cephadm auto-deploy mon lên compute node:

```bash
# Chia ssh-pubkey cephadm tới ctrl02, ctrl03
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ctrl02
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ctrl03

# Pause orchestrator
ceph orch pause

# ctrl01 đã tự đăng ký sau bootstrap — thêm labels
ceph orch host label add ctrl01 mon
ceph orch host label add ctrl01 mgr
ceph orch host label add ctrl01 osd

# Add ctrl02, ctrl03 với IP tunnel network
ceph orch host add ctrl02 10.10.201.12 --labels mon,mgr,osd
ceph orch host add ctrl03 10.10.201.13 --labels mon,osd

ceph orch host ls --detail   # 3 hosts: ctrl01, ctrl02, ctrl03

# Lock placement — mon chỉ trên ctrl nodes, mgr trên 2 node đầu
ceph orch apply mon --placement="ctrl01,ctrl02,ctrl03"
ceph orch apply mgr --placement="ctrl01,ctrl02"

# Resume — cephadm bắt đầu deploy theo spec đã lock
ceph orch resume

watch -n3 ceph -s   # Chờ "mon: 3 daemons, quorum ctrl01,ctrl02,ctrl03"
```

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/06.png"/>

```bash
# Xem disk có sẵn
ceph orch device ls

# Deploy 3 OSD trên /dev/sdb
ceph orch daemon add osd ctrl01:/dev/sdb
ceph orch daemon add osd ctrl02:/dev/sdb
ceph orch daemon add osd ctrl03:/dev/sdb

watch -n5 ceph -s    # Chờ "3 osds: 3 up, 3 in"

# Verify toàn bộ cluster
ceph -s
ceph health detail
ceph orch host ls --detail
ceph orch ps
ceph config get mgr mgr/cephadm/container_image_base  # phải là local registry
```

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/07.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/08.png"/>

### 5.3 Tạo pools & keyrings cho OpenStack

Mình tạo 4 pool với replication factor 2 (phù hợp 3 OSD nhỏ) và 4 keyring tương ứng cho từng service OpenStack:

```bash
ceph config set global osd_pool_default_size     2
ceph config set global osd_pool_default_min_size 1

ceph osd pool create volumes 64    # Cinder
ceph osd pool create images  32    # Glance
ceph osd pool create vms     64    # Nova ephemeral
ceph osd pool create backups 32    # Cinder backups

for p in volumes images vms backups; do rbd pool init "$p"; done

ceph auth get-or-create client.glance \
  mon 'allow r' \
  osd 'allow class-read object_prefix rbd_children, allow rwx pool=images' \
  > /etc/ceph/ceph.client.glance.keyring

ceph auth get-or-create client.cinder \
  mon 'profile rbd' \
  osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images' \
  > /etc/ceph/ceph.client.cinder.keyring

ceph auth get-or-create client.nova \
  mon 'profile rbd' \
  osd 'profile rbd pool=vms, profile rbd-read-only pool=images' \
  > /etc/ceph/ceph.client.nova.keyring

ceph auth get-or-create client.cinder-backup \
  mon 'profile rbd' \
  osd 'profile rbd pool=backups' \
  > /etc/ceph/ceph.client.cinder-backup.keyring

ceph -s             # HEALTH_OK · 3 osds: 3 up, 3 in
ceph osd lspools
ceph df
```

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/09.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/10.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/11.png"/>

---

## 6. Setup kolla-ansible trên ctrl01

Mình dùng kolla-ansible từ PyPI. Python venv riêng tránh conflict với system Python:

```bash
# Trên ctrl01 — kiêm Deploy node
apt update && apt install -y python3-venv python3-pip git sshpass

# SSH key từ ctrl01 tới các node còn lại
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519 -q || true
for node in ctrl01 ctrl02 ctrl03 registry01; do
  ssh-copy-id -o StrictHostKeyChecking=no anhlx@$node
done

# Venv riêng cho deploy
python3 -m venv /opt/kolla-venv
source /opt/kolla-venv/bin/activate
pip install --upgrade pip setuptools wheel

# kolla-ansible 2025.1 (version 20.x)
pip install 'kolla-ansible>=20.0.0,<21.0.0'

kolla-ansible install-deps

kolla-ansible --version    # kolla-ansible 20.x.x
ansible --version          # core 2.17.x (≥ 2.16 là OK)
```

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/12.png"/>

---

## 7. globals.yml & inventory multinode

Thực hiện trên **`ctrl01`**. 

```bash
mkdir -p /etc/kolla

cat > /etc/kolla/globals.yml << 'EOF'
---
# Docker Registry
docker_registry: "10.10.200.5:5000"
docker_registry_insecure: "yes"
docker_namespace: "openstack.kolla"

# Base
kolla_base_distro: "ubuntu"
openstack_release: "2025.1"
openstack_tag: "2025.1-ubuntu-noble"

# Network interfaces
network_interface: "ens160"              # Management / API (VLAN 200)
tunnel_interface:  "ens192"              # OVN Geneve + Ceph (VLAN 201)
neutron_external_interface: "ens224"     # Provider network (no IP, OVN br-ex)

# VIP & HA
kolla_internal_vip_address: "10.10.200.50"
kolla_external_vip_address: "10.10.200.50"
enable_haproxy:    "yes"
enable_keepalived: "yes"

# Neutron + OVN
enable_neutron:                   "yes"
neutron_plugin_agent:             "ovn"
enable_neutron_provider_networks: "yes"
neutron_ovn_distributed_fip:      "yes"

# External Ceph (cephadm — deploy riêng ở bước 5)
enable_ceph:            "no"
glance_backend_ceph:    "yes"
cinder_backend_ceph:    "yes"
nova_backend_ceph:      "yes"
cinder_backup_driver:   "ceph"
ceph_glance_user:        "glance"
ceph_cinder_user:        "cinder"
ceph_nova_user:          "nova"
ceph_cinder_backup_user: "cinder-backup"

# Compute — AIO: Nova chạy cùng node với controller
nova_compute_virt_type:       "kvm"
nova_reserved_host_memory_mb: "512"

# Services
enable_cinder:         "yes"
enable_cinder_backup:  "yes"
cinder_cluster_name:   "cinder-cluster"
enable_heat:           "yes"
enable_horizon:        "yes"
enable_barbican:       "yes"
enable_chrony:         "no"
enable_octavia:        "no"
enable_designate:      "no"

# Coordination backend (bắt buộc khi dùng Cinder Ceph backend)
enable_valkey: "yes"

# Monitoring
enable_prometheus: "yes"
enable_grafana:    "yes"
EOF
```

```bash
cat > /etc/kolla/multinode << 'EOF'
[control]
ctrl01  ansible_host=10.10.200.11
ctrl02  ansible_host=10.10.200.12
ctrl03  ansible_host=10.10.200.13

[network]
ctrl01
ctrl02
ctrl03

[loadbalancer]
ctrl01
ctrl02
ctrl03

# AIO: compute + storage = ctrl nodes
[compute]
ctrl01
ctrl02
ctrl03

[storage]
ctrl01
ctrl02
ctrl03

[monitoring]
ctrl01

[deployment]
localhost  ansible_connection=local

[baremetal:children]
control
network
compute
storage
monitoring

# ── Nova sub-groups ────────────────────────────────────────────────────────
[nova:children]
control

[nova-api:children]
nova

[nova-metadata:children]
nova

[nova-conductor:children]
nova

[nova-scheduler:children]
nova

[nova-super-conductor:children]
nova

[nova-novncproxy:children]
nova

[nova-spicehtml5proxy:children]

[nova-serialproxy:children]

[nova-compute:children]
compute

[nova-compute-ironic:children]

[nova-libvirt:children]
compute

[nova-ssh:children]
compute

# ── Placement ──────────────────────────────────────────────────────────────
[placement:children]
control

[placement-api:children]
placement

# ── Keystone ───────────────────────────────────────────────────────────────
[keystone:children]
control

# ── Glance ─────────────────────────────────────────────────────────────────
[glance:children]
control

[glance-api:children]
glance

# ── Cinder ─────────────────────────────────────────────────────────────────
[cinder:children]
control

[cinder-api:children]
cinder

[cinder-scheduler:children]
cinder

[cinder-volume:children]
storage

[cinder-backup:children]
storage

# ── Neutron ────────────────────────────────────────────────────────────────
[neutron:children]
control

[neutron-server:children]
neutron

[neutron-dhcp-agent:children]
network

[neutron-l3-agent:children]
network

[neutron-metadata-agent:children]
network

[neutron-ovn-metadata-agent:children]
network

[neutron-ovn-agent:children]

[neutron-bgp-dragent:children]

[neutron-infoblox-ipam-agent:children]

[neutron-metering-agent:children]

[ironic-neutron-agent:children]

# ── OVN ────────────────────────────────────────────────────────────────────
# ovn-controller = ovn-controller-compute + ovn-controller-network
# Phải khai báo đủ 3 sub-group, thiếu bất kỳ group nào → lỗi deploy:
#   'dict object' has no attribute 'ovn-controller-compute/network'
[ovn-controller:children]
ovn-controller-compute
ovn-controller-network

[ovn-controller-compute:children]
compute

[ovn-controller-network:children]
network

# ovn-database gom NB DB + SB DB + northd — chạy trên control nodes
[ovn-database:children]
control

[ovn-northd:children]
ovn-database

[ovn-nb-db:children]
ovn-database

[ovn-sb-db:children]
ovn-database

[ovn-sb-db-relay:children]
ovn-database

# ── Heat ───────────────────────────────────────────────────────────────────
[heat:children]
control

[heat-api:children]
heat

[heat-api-cfn:children]
heat

[heat-engine:children]
heat

# ── Horizon ────────────────────────────────────────────────────────────────
[horizon:children]
control

# ── Barbican ───────────────────────────────────────────────────────────────
[barbican:children]
control

[barbican-api:children]
barbican

[barbican-keystone-listener:children]
barbican

[barbican-worker:children]
barbican

# ── DB / MQ / Cache ────────────────────────────────────────────────────────
[valkey:children]
control

[valkey-server:children]
valkey

[valkey-sentinel:children]
valkey

[mariadb:children]
control

[rabbitmq:children]
control

[memcached:children]
control

# ── HA / LB ────────────────────────────────────────────────────────────────
[haproxy:children]
loadbalancer

# ── Monitoring ─────────────────────────────────────────────────────────────
[grafana:children]
monitoring

[prometheus:children]
monitoring

[prometheus-server:children]
monitoring

[prometheus-node-exporter:children]
baremetal

[prometheus-mysqld-exporter:children]
mariadb

[prometheus-memcached-exporter:children]
memcached

[prometheus-cadvisor:children]
baremetal

[prometheus-alertmanager:children]
monitoring

[prometheus-openstack-exporter:children]
monitoring

[prometheus-elasticsearch-exporter:children]

[prometheus-blackbox-exporter:children]
monitoring

[prometheus-libvirt-exporter:children]
nova-libvirt

# ── Kolla infrastructure ────────────────────────────────────────────────────
[kolla-toolbox:children]
control

[cron:children]
control

[fluentd:children]
control

[kolla-logs:children]
control

# Required by etc_hosts role — để trống nếu không dùng Bifrost
[bifrost]

[all:vars]
ansible_become_method=sudo
EOF
```

> `passwords.yml` sinh tự động ở bước `kolla-genpwd`. Kolla tự fan-out các service group con từ `[control]`, `[compute]`, `[storage]`, `[network]`.

---

## 8. Copy Ceph configs vào Kolla

Mình copy ceph.conf và các keyring vào từng service directory. Lưu ý: Kolla INI parser không chịu leading TAB trong ceph.conf (sinh ra khi dùng `ceph config generate-minimal-conf`) — phải sed trước khi copy:

```bash
# Trên ctrl01 — vừa là deploy node vừa là ceph bootstrap node
mkdir -p /etc/kolla/config/{glance,cinder/cinder-volume,cinder/cinder-backup,nova/nova-compute}

cp /etc/ceph/ceph.conf                          /etc/kolla/config/
cp /etc/ceph/ceph.client.glance.keyring         /etc/kolla/config/glance/
cp /etc/ceph/ceph.client.cinder.keyring         /etc/kolla/config/cinder/cinder-volume/
cp /etc/ceph/ceph.client.cinder-backup.keyring  /etc/kolla/config/cinder/cinder-backup/
cp /etc/ceph/ceph.client.nova.keyring           /etc/kolla/config/nova/

# cinder-backup cần cả 2 keyring: cinder (đọc volume) và cinder-backup (ghi backup pool)
cp /etc/kolla/config/cinder/cinder-volume/ceph.client.cinder.keyring \
   /etc/kolla/config/cinder/cinder-backup/ceph.client.cinder.keyring

# nova-cell role dùng ceph_nova_keyring = ceph_cinder_keyring → tìm ceph.client.cinder.keyring trong nova/
cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/nova/

# Strip leading TAB
sed -i 's/^\t//' /etc/kolla/config/ceph.conf

# Copy ceph.conf vào từng service directory
for d in glance cinder/cinder-volume cinder/cinder-backup nova; do
  cp /etc/kolla/config/ceph.conf /etc/kolla/config/$d/
done
```
```bash
# Verify — mỗi service dir phải có ceph.conf + keyring tương ứng
(kolla-venv) root@ctrl01:/root/#  tree /etc/kolla/
/etc/kolla/
├── config
│   ├── ceph.conf
│   ├── cinder
│   │   ├── cinder-backup
│   │   │   ├── ceph.client.cinder-backup.keyring
│   │   │   ├── ceph.client.cinder.keyring
│   │   │   └── ceph.conf
│   │   └── cinder-volume
│   │       ├── ceph.client.cinder.keyring
│   │       └── ceph.conf
│   ├── glance
│   │   ├── ceph.client.glance.keyring
│   │   └── ceph.conf
│   └── nova
│       ├── ceph.client.cinder.keyring
│       ├── ceph.client.nova.keyring
│       ├── ceph.conf
│       └── nova-compute
├── globals.yml
└── multinode

8 directories, 13 files
(kolla-venv) root@ctrl01:/root/#
```

---

## 9. Deploy OpenStack

Mình chạy deploy theo thứ tự: `bootstrap-servers` → **OVS pre-config** → `prechecks` → `pull` → `deploy` → `post-deploy`. Bước pull mất 5–10 phút, deploy mất 20–40 phút.

```bash
source /opt/kolla-venv/bin/activate

# 1. Sinh passwords.yml
cp /opt/kolla-venv/share/kolla-ansible/etc_examples/kolla/passwords.yml /etc/kolla/passwords.yml
kolla-genpwd
# → /etc/kolla/passwords.yml — sao lưu file này ra nơi an toàn

# 2. ansible.cfg
cat > /etc/kolla/ansible.cfg << 'EOF'
[defaults]
host_key_checking = False
interpreter_python = auto_silent
remote_user = anhlx
private_key_file = /root/.ssh/id_ed25519

[privilege_escalation]
become = True
become_method = sudo
EOF
export ANSIBLE_CONFIG=/etc/kolla/ansible.cfg

# 3. Test connectivity — expect: 3 ctrl nodes + localhost → SUCCESS
ansible -i /etc/kolla/multinode all -m ping

# 4. Bootstrap — cài Docker + python deps, apply daemon.json trên tất cả node
kolla-ansible bootstrap-servers -i /etc/kolla/multinode

# bootstrap-servers đổi daemon.json và restart Docker → Ceph daemons bị dead
# Start lại thủ công trên tất cả ctrl nodes
FSID=$(cephadm shell -- ceph fsid)
ansible -i /etc/kolla/multinode control -m shell \
  -a "systemctl start ceph-${FSID}.target" --become

# Verify HEALTH_OK trước khi tiếp tục
ceph -s

# bootstrap-servers restart Docker nhưng không set OVS manager và br-ex — kolla openvswitch
# role chạy sau đó nhưng hay race với ovn-controller start. Set thủ công để chắc chắn:
ansible -i /etc/kolla/multinode baremetal --become -m shell -a "
  ovs-vsctl set-manager ptcp:6640:127.0.0.1
  ovs-vsctl --may-exist add-br br-ex
  ovs-vsctl --may-exist add-port br-ex ens224
"

# 5. Prechecks
kolla-ansible prechecks -i /etc/kolla/multinode

# 6. Pull images từ local registry (~5–10 phút)
kolla-ansible pull -i /etc/kolla/multinode

# 7. Deploy (~20–40 phút)
kolla-ansible deploy -i /etc/kolla/multinode

# 8. Post-deploy → sinh admin-openrc.sh
kolla-ansible post-deploy -i /etc/kolla/multinode

# 9. Verify OpenStack core services
pip install python-openstackclient
source /etc/kolla/admin-openrc.sh
openstack service list           
openstack compute service list   # expect 3 nova-compute up, host = ctrl01/02/03
openstack network agent list     # expect 3 "OVN Controller Gateway agent" trên ctrl01/02/03 — Alive :-)

# 10. Verify OVN containers (trên ctrl01 — đại diện cho cả cluster)
docker ps --format "{{.Names}}" | grep -E "^neutron|^ovn" | sort

# OVN logical topology — phải empty trước khi tạo network
docker exec ovn_northd ovn-nbctl show
# OVN chassis đã đăng ký — phải thấy 3 entry (ctrl01, ctrl02, ctrl03)
docker exec ovn_northd ovn-sbctl list Chassis | grep -E "hostname|name\s+:"
```

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/13.png"/>

**Nếu deploy lỗi:**

```bash
# Log của 1 service
docker logs -f neutron_server

# Reconfigure — re-render config, restart containers (KHÔNG mất data)
kolla-ansible -i /etc/kolla/multinode reconfigure

# Reset toàn bộ — XOÁ HẾT, chỉ dùng khi muốn làm lại từ đầu
kolla-ansible -i /etc/kolla/multinode destroy --yes-i-really-really-mean-it
docker volume prune -f
```

---

## 10. Service URLs & Credentials

Mình tổng hợp các endpoint để dễ bookmark — password lấy từ `passwords.yml` sau khi deploy xong.

| Service | URL | Credentials |
|---|---|---|
| Horizon | `http://10.10.200.50` | `admin` / `grep keystone_admin_password /etc/kolla/passwords.yml` |
| Ceph Dashboard | `https://10.10.200.11:8443` | `admin` / password đặt lúc bootstrap |
| Grafana | `http://10.10.200.50:3000` | `admin` / `grep grafana_admin_password /etc/kolla/passwords.yml` |
| Prometheus | `http://10.10.200.50:9091` | `admin` / `grep prometheus_password /etc/kolla/passwords.yml` |
| HAProxy Stats | `http://10.10.200.11:1984/` | `openstack` / `grep haproxy_password /etc/kolla/passwords.yml` |

> HAProxy stats bind trên **management IP của từng node** (không phải VIP) — mỗi ctrl node có trang riêng: `.11:1984`, `.12:1984`, `.13:1984`.
>
> Grafana deploy **không kèm dashboard** — Prometheus data source được Kolla auto-provision, nhưng dashboard phải import thủ công. Vào **Connections → Data sources → Prometheus → Dashboards** để import Node Exporter mặc định, hoặc import từ grafana.com: Node Exporter Full (ID `1860`), RabbitMQ (ID `10991`), HAProxy 2 Full (ID `12693`).
>
> Ceph Dashboard chạy trên mgr active — kiểm tra node đang chạy: `ceph mgr services`.

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/14.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/15.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/16.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/17.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/18.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/19.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/20.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/21.png"/>

---

## 11. Workload đầu tiên — Demo Private Cloud multi tenant

Mình mô phỏng một công ty nhỏ với hai phòng ban **IT** và **Dev** — mỗi team có project riêng, quota độc lập, và tự quản lý network + VM của mình.

### 11.1 Admin — Projects, users và shared resources

Mình đăng nhập với quyền admin để tạo hạ tầng dùng chung trước.

```bash
source /etc/kolla/admin-openrc.sh

# Projects và users
openstack project create it-dept  --description "Phong IT"
openstack project create dev-dept --description "Phong Dev"

openstack user create user-it  --password <password-it>  --project it-dept
openstack user create user-dev --password <password-dev> --project dev-dept

openstack role add --project it-dept  --user user-it  member
openstack role add --project dev-dept --user user-dev member

# Quota cho mỗi team
openstack quota set --cores 8 --ram 8192 --instances 5 --volumes 5 --gigabytes 100 it-dept
openstack quota set --cores 12 --ram 16384 --instances 5 --volumes 5 --gigabytes 100 dev-dept
```

```bash
# External / provider network — flat trên br-ex (ens224 → VLAN 200)
openstack network create \
  --share --external \
  --provider-network-type flat \
  --provider-physical-network physnet1 \
  external-net

openstack subnet create external-subnet \
  --network external-net \
  --subnet-range 10.10.200.0/24 \
  --allocation-pool start=10.10.200.61,end=10.10.200.69 \
  --gateway 10.10.200.1 \
  --dns-nameserver 8.8.8.8 \
  --no-dhcp

# Flavor và image dùng chung cho cả hai team
openstack flavor create --vcpus 1 --ram 1024 --disk 20 m1.small

wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img \
  -O /root/jammy.img
openstack image create "Ubuntu-22.04" \
  --file /root/jammy.img --disk-format qcow2 --container-format bare --public
```

```bash
# Verify
openstack project list
openstack user list
openstack network list
openstack subnet list
openstack flavor list
openstack image list
openstack quota show it-dept
openstack quota show dev-dept
```

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/22.png"/>


### 11.2 Team IT — Ubuntu VM

Mình tạo openrc riêng cho user-it rồi deploy network và VM trong project it-dept.

```bash
cat > /etc/kolla/it-openrc.sh << 'EOF'
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=it-dept
export OS_USERNAME=user-it
export OS_PASSWORD=<password-it>
export OS_AUTH_URL=http://10.10.200.50:5000
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF

source /etc/kolla/it-openrc.sh
```

```bash
# Tenant network của IT
openstack network create it-network
openstack subnet create it-subnet \
  --network it-network \
  --subnet-range 192.168.10.0/24 \
  --allocation-pool start=192.168.10.10,end=192.168.10.250 \
  --dns-nameserver 8.8.8.8

openstack router create it-router
openstack router set it-router --external-gateway external-net
openstack router add subnet it-router it-subnet

# Keypair + security group
openstack keypair create --public-key ~/.ssh/id_ed25519.pub it-key
openstack security group rule create --protocol icmp default
openstack security group rule create --protocol tcp --dst-port 22 default

# Launch VM + floating IP
NET=$(openstack network show it-network -f value -c id)
openstack server create \
  --flavor m1.small --image Ubuntu-22.04 \
  --network $NET --key-name it-key \
  --security-group default \
  vm-it-01

FIP_IT=$(openstack floating ip create external-net -f value -c floating_ip_address)
openstack server add floating ip vm-it-01 $FIP_IT

openstack server show vm-it-01 -c status -c addresses
ssh -i ~/.ssh/id_ed25519 ubuntu@$FIP_IT
```
<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/23.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/24.png"/>

### 11.3 Team Dev — Ubuntu VM

Mình switch sang project dev-dept — network và VM hoàn toàn tách biệt với it-dept.

```bash
cat > /etc/kolla/dev-openrc.sh << 'EOF'
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=dev-dept
export OS_USERNAME=user-dev
export OS_PASSWORD=<password-dev>
export OS_AUTH_URL=http://10.10.200.50:5000
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF

source /etc/kolla/dev-openrc.sh
```

```bash
# Tenant network của Dev — dải khác hoàn toàn
openstack network create dev-network
openstack subnet create dev-subnet \
  --network dev-network \
  --subnet-range 192.168.20.0/24 \
  --allocation-pool start=192.168.20.10,end=192.168.20.250 \
  --dns-nameserver 8.8.8.8

openstack router create dev-router
openstack router set dev-router --external-gateway external-net
openstack router add subnet dev-router dev-subnet

# Keypair + security group
openstack keypair create --public-key ~/.ssh/id_ed25519.pub dev-key
openstack security group rule create --protocol icmp default
openstack security group rule create --protocol tcp --dst-port 22 default

# Launch VM + floating IP
NET=$(openstack network show dev-network -f value -c id)
openstack server create \
  --flavor m1.small --image Ubuntu-22.04 \
  --network $NET --key-name dev-key \
  --security-group default \
  vm-dev-01

FIP_DEV=$(openstack floating ip create external-net -f value -c floating_ip_address)
openstack server add floating ip vm-dev-01 $FIP_DEV

openstack server show vm-dev-01 -c status -c addresses
ssh -i ~/.ssh/id_ed25519 ubuntu@$FIP_DEV
```

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/25.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/26.png"/>

### 11.4 Verify isolation

Mình kiểm tra nhanh để xác nhận mỗi project chỉ thấy tài nguyên của mình.

```bash
source /etc/kolla/it-openrc.sh
openstack server list   # chỉ thấy vm-it-01

source /etc/kolla/dev-openrc.sh
openstack server list   # chỉ thấy vm-dev-01

source /etc/kolla/admin-openrc.sh
openstack server list --all-projects   # thấy cả hai
```

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/27.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/28.png"/>

<img src="/assets/img/2026-05-21-openstack-epoxy-ovn-ceph-aio/29.png"/>

---

## 12. Lời kết

Lab này cho mình thấy rõ điểm mạnh của OVN trong AIO setup: toàn bộ SDN layer — distributed L3, security groups qua OVS ACL, DNAT/SNAT cho floating IP — chỉ cần vài container nhẹ tích hợp sẵn trong kolla-ansible, không cần component ngoài nào thêm. Kết hợp với Ceph hyperconverged, 3 node này đủ chạy một private cloud HA hoàn chỉnh trên tài nguyên vừa phải. 

---
