---
title: "Private Cloud - Kolla Deploy OpenStack Bobcat (2023.2) + Ceph trên 3 Node All-in-One"
categories:
- Cloud
- OpenStack
- Ceph
- Linux

feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Xây dựng Private Cloud cho doanh nghiệp với OpenStack Bobcat + Ceph - Kolla-Ansible trên 3 node All-in-One
---

### Mục lục

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Kiến trúc & Mô hình triển khai](#2-kiến-trúc--mô-hình-triển-khai)
- [3. Cấu hình cơ bản trên 3 node](#3-cấu-hình-cơ-bản-trên-3-node)
- [4. Cài đặt Kolla-Ansible](#4-cài-đặt-kolla-ansible)
- [5. Deploy OpenStack + Ceph](#5-deploy-openstack--ceph)
- [6. Kiểm tra dịch vụ](#6-kiểm-tra-dịch-vụ)
- [7. Cấu hình Multi-tenant cho doanh nghiệp](#7-cấu-hình-multi-tenant-cho-doanh-nghiệp)
- [8. Demo - Tạo VM trong OpenStack](#8-demo---tạo-vm-trong-openstack)
- [9. Scale & Best Practices](#9-scale--best-practices)
- [10. Kết luận](#10-kết-luận)

---

### 1. Giới thiệu

Private Cloud là mô hình điện toán đám mây được triển khai nội bộ trong doanh nghiệp, cung cấp khả năng tự phục vụ (self-service) tài nguyên compute, storage, network cho các phòng ban mà vẫn đảm bảo kiểm soát bảo mật và chi phí.

**OpenStack** là nền tảng Private Cloud mã nguồn mở phổ biến nhất, được sử dụng bởi nhiều tổ chức lớn trên thế giới. Kết hợp với **Ceph** - hệ thống lưu trữ phân tán (Software-Defined Storage), ta có một giải pháp Private Cloud hoàn chỉnh với tính sẵn sàng cao.

**Mục tiêu bài lab:**
- Triển khai OpenStack Bobcat (2023.2) trên 3 node All-in-One bằng **Kolla-Ansible**
- Tích hợp **Ceph** làm storage backend cho Glance, Cinder, Nova
- Sử dụng **OVN** cho Neutron networking (self-service + provider network)
- Mô phỏng môi trường production cho doanh nghiệp với 2 phòng ban: **IT-Helpdesk** và **Dev**
- Demo tạo VM Ubuntu Server và Windows Server 2022 trong OpenStack

**Tại sao dùng 3 node All-in-One?**
- Đây là mô hình PoC (Proof of Concept) - mỗi node chạy tất cả role (control, network, compute, storage)
- Đảm bảo HA cho control plane (MariaDB Galera, RabbitMQ cluster, HAProxy + Keepalived)
- Sau này có thể scale bằng cách thêm compute node riêng hoặc tách role

<img src="/assets/img/2026-04-17-openstack-bobcat-kolla-private-cloud/01-topology.png"/>

---

### 2. Kiến trúc & Mô hình triển khai

#### 2.1 Topology

```
   ┌─────────────────────────────────────────────────────────────────────┐
   │  Network-Mgmt          Network-External       Network-Storage       │
   │  VLAN 200              VLAN 200               Host-Only             │
   │  10.10.200.0/24        10.10.200.0/24         10.10.201.0/24        │
   │  (NAT → Internet)      (Floating IP)                                │
   └──┬─────┬─────┬─────────┬─────┬─────┬─────────┬─────┬─────┬─────────┘
      │     │     │         │     │     │         │     │     │
   ┌──┴──┐┌─┴──┐┌─┴──┐  ┌──┴──┐┌─┴──┐┌─┴──┐  ┌──┴──┐┌─┴──┐┌─┴──┐
   │ens  ││ens ││ens │  │ens  ││ens ││ens │  │ens  ││ens ││ens │
   │160  ││160 ││160 │  │192  ││192 ││192 │  │224  ││224 ││224 │
   │     ││    ││    │  │     ││    ││    │  │     ││    ││    │
   │cloud││clou││clou│  │cloud││clou││clou│  │cloud││clou││clou│
   │ -01 ││d-02││d-03│  │ -01 ││d-02││d-03│  │ -01 ││d-02││d-03│
   └─────┘└────┘└────┘  └─────┘└────┘└────┘  └─────┘└────┘└────┘
```

#### 2.2 Tài nguyên mỗi node

| Thành phần | Cấu hình |
|------------|----------|
| vCPU | 12 cores |
| RAM | 24 GB |
| Disk OS | 1 x 100 GB (sda) |
| Disk OSD | 2 x 500 GB (sdb, sdc) |
| NIC 1 | Management/API |
| NIC 2 | External/Provider |
| NIC 3 | Storage/Ceph |
| OS | Ubuntu 24.04 LTS Server |

**Tổng tài nguyên ESXi host cần:** 36 vCPU, 72 GB RAM, 3.3 TB disk

#### 2.3 Network Planning

| Network | Subnet | VLAN | Interface | Mục đích |
|---------|--------|------|-----------|----------|
| Management | 10.10.200.0/24 | VLAN 200 | ens160 | API, internal, tunnel (Geneve), SSH - NAT ra internet |
| External | 10.10.200.0/24 | VLAN 200 | ens192 | Provider network, Floating IP (.150-.160) |
| Storage | 10.10.201.0/24 | - (Host-Only) | ens224 | Ceph public + cluster network |

> **Lưu ý:** Management và External cùng subnet 10.10.200.0/24 (VLAN 200). NIC1 (ens160) có IP để quản lý, NIC2 (ens192) không gán IP - Neutron quản lý trực tiếp cho floating IP.

#### 2.4 IP Planning

| Node | Hostname | Mgmt (ens160) | External (ens192) | Storage (ens224) |
|------|----------|---------------|-------------------|------------------|
| VIP | - | 10.10.200.10 | - | - |
| Node 1 | cloud-01 | 10.10.200.11 | No IP (Neutron) | 10.10.201.11 |
| Node 2 | cloud-02 | 10.10.200.12 | No IP (Neutron) | 10.10.201.12 |
| Node 3 | cloud-03 | 10.10.200.13 | No IP (Neutron) | 10.10.201.13 |

> **Lưu ý:** NIC 2 (ens192) **không gán IP** trên host - interface này được Neutron sử dụng trực tiếp làm provider bridge cho external network.

#### 2.5 OpenStack Services

| Service | Vai trò | Backend |
|---------|---------|---------|
| Keystone | Identity & Authentication | MariaDB |
| Glance | Image Service | **Ceph RBD** (pool: images) |
| Nova | Compute Service | **Ceph RBD** (pool: vms) |
| Neutron (OVN) | Networking | OVN Northbound/Southbound DB |
| Cinder | Block Storage | **Ceph RBD** (pool: volumes) |
| Horizon | Web Dashboard | - |
| Heat | Orchestration | MariaDB |
| Placement | Resource Tracking | MariaDB |
| HAProxy + Keepalived | Load Balancing & VIP | - |
| MariaDB Galera | Database Cluster | - |
| RabbitMQ | Message Queue Cluster | - |
| Memcached | Caching | - |

#### 2.6 Ceph Architecture

| Thành phần | Số lượng | Ghi chú |
|------------|----------|---------|
| MON (Monitor) | 3 | 1 per node |
| MGR (Manager) | 3 | 1 per node |
| OSD (Object Storage Daemon) | 6 | 2 per node (sdb, sdc) |
| Tổng raw capacity | 3 TB | 6 x 500 GB |
| Usable capacity (3x replica) | ~1 TB | Replication factor 3 |

**Ceph Pools:**

| Pool | Mục đích | PG Count |
|------|----------|----------|
| images | Glance images | 128 |
| volumes | Cinder volumes | 128 |
| vms | Nova ephemeral disks | 128 |
| backups | Cinder backups | 128 |

---

### 3. Cấu hình cơ bản trên 3 node

> **Thực hiện trên cả 3 node** (trừ khi ghi chú khác)

#### 3.1 Hostname & /etc/hosts

**Node 1:**
```bash
sudo hostnamectl set-hostname cloud-01
```

**Node 2:**
```bash
sudo hostnamectl set-hostname cloud-02
```

**Node 3:**
```bash
sudo hostnamectl set-hostname cloud-03
```

**Trên cả 3 node**, thêm vào `/etc/hosts`:
```bash
cat << 'EOF' | sudo tee -a /etc/hosts
10.10.200.10 cloud-vip
10.10.200.11 cloud-01
10.10.200.12 cloud-02
10.10.200.13 cloud-03
EOF
```

#### 3.2 Cấu hình Network (Netplan)

**Node 1** - `/etc/netplan/50-cloud-init.yaml`:
```yaml
network:
  version: 2
  ethernets:
    ens160:
      addresses:
        - "10.10.200.11/24"
      routes:
      - to: "default"
        via: "10.10.200.1"
      nameservers:
        addresses: [8.8.8.8]
    ens192: {}
    ens224:
      addresses:
        - "10.10.201.11/24"
```

**Node 2** - thay IP tương ứng `.12`:
```yaml
network:
  version: 2
  ethernets:
    ens160:
      addresses:
        - "10.10.200.12/24"
      routes:
      - to: "default"
        via: "10.10.200.1"
      nameservers:
        addresses: [8.8.8.8]
    ens192: {}
    ens224:
      addresses:
        - "10.10.201.12/24"
```

**Node 3** - thay IP tương ứng `.13`:
```yaml
network:
  version: 2
  ethernets:
    ens160:
      addresses:
        - "10.10.200.13/24"
      routes:
      - to: "default"
        via: "10.10.200.1"
      nameservers:
        addresses: [8.8.8.8]
    ens192: {}
    ens224:
      addresses:
        - "10.10.201.13/24"
```

Apply cấu hình:
```bash
sudo netplan apply
```

> **Lưu ý:** `ens192: {}` khai báo interface nhưng **không gán IP** - Neutron sẽ quản lý interface này.

Kiểm tra:
```bash
ip addr show
ping -c 3 cloud-01
ping -c 3 cloud-02
ping -c 3 cloud-03
```

#### 3.3 Chuẩn bị disk cho Ceph OSD

Kiểm tra disk trên mỗi node:
```bash
lsblk
```

Output mong đợi:
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0  100G  0 disk
├─sda1   8:1    0    1G  0 part /boot/efi
├─sda2   8:2    0    2G  0 part [SWAP]
└─sda3   8:3    0   97G  0 part /
sdb      8:16   0  500G  0 disk
sdc      8:32   0  500G  0 disk
```

Tạo label cho Ceph OSD (BlueStore):
```bash
# Xóa sạch partition table cũ (nếu có)
sudo wipefs -a /dev/sdb
sudo wipefs -a /dev/sdc

# Tạo GPT label với partition name KOLLA_CEPH_OSD_BOOTSTRAP_BS
sudo parted /dev/sdb -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS 1 -1
sudo parted /dev/sdc -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS 1 -1
```

Kiểm tra:
```bash
lsblk -o NAME,SIZE,TYPE,PARTLABEL
```

Output:
```
NAME   SIZE TYPE PARTLABEL
sda    100G disk
├─sda1   1G part
├─sda2   2G part
└─sda3  97G part
sdb    500G disk
└─sdb1 500G part KOLLA_CEPH_OSD_BOOTSTRAP_BS
sdc    500G disk
└─sdc1 500G part KOLLA_CEPH_OSD_BOOTSTRAP_BS
```

> Kolla-Ansible sẽ tự động nhận diện các partition có label `KOLLA_CEPH_OSD_BOOTSTRAP_BS` để tạo OSD.

#### 3.4 Kiểm tra Nested Virtualization

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

Nếu output > 0 → KVM hoạt động (dùng `kvm` cho Nova).
Nếu output = 0 → Cần bật nested virt trên ESXi hoặc dùng `qemu` (chậm hơn).

#### 3.5 Tắt firewall & cập nhật hệ thống

```bash
sudo systemctl disable --now ufw
sudo apt update && sudo apt upgrade -y
sudo reboot
```

---

### 4. Cài đặt Kolla-Ansible

> **Thực hiện trên cloud-01** - node này đóng vai trò deploy node

#### 4.1 SSH Key-based Authentication

Tạo SSH key trên cloud-01 và copy sang tất cả node:
```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

# Copy key sang cả 3 node (bao gồm chính nó)
ssh-copy-id sysadmin@cloud-01
ssh-copy-id sysadmin@cloud-02
ssh-copy-id sysadmin@cloud-03
```

Kiểm tra SSH không cần password:
```bash
ssh cloud-02 hostname
ssh cloud-03 hostname
```

Cấu hình sudo không cần password (trên cả 3 node):
```bash
echo "sysadmin ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/sysadmin
```

#### 4.2 Cài đặt Python & Kolla-Ansible

```bash
# Cài đặt dependencies
sudo apt install -y python3-dev python3-venv python3-pip libffi-dev gcc libssl-dev git

# Tạo virtual environment
python3 -m venv ~/kolla-venv
source ~/kolla-venv/bin/activate

# Upgrade pip
pip install -U pip setuptools

# Cài đặt Ansible (phiên bản tương thích với Kolla Bobcat)
pip install 'ansible-core>=2.14,<2.16'

# Cài đặt Kolla-Ansible phiên bản Bobcat (2023.2)
pip install 'kolla-ansible==17.2.0'
```

#### 4.3 Chuẩn bị thư mục cấu hình

```bash
# Tạo thư mục /etc/kolla
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla

# Copy file cấu hình mẫu
cp -r ~/kolla-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/

# Copy inventory mẫu
cp ~/kolla-venv/share/kolla-ansible/ansible/inventory/multinode ~/multinode

# Tạo thư mục cho Ansible
sudo mkdir -p /etc/ansible
```

Tạo `/etc/ansible/ansible.cfg`:
```bash
cat << 'EOF' | sudo tee /etc/ansible/ansible.cfg
[defaults]
host_key_checking = False
pipelining = True
forks = 100
EOF
```

#### 4.4 Inventory Multinode

Chỉnh sửa file `~/multinode`:

```ini
[control]
cloud-01 ansible_user=sysadmin ansible_become=true
cloud-02 ansible_user=sysadmin ansible_become=true
cloud-03 ansible_user=sysadmin ansible_become=true

[network]
cloud-01 ansible_user=sysadmin ansible_become=true
cloud-02 ansible_user=sysadmin ansible_become=true
cloud-03 ansible_user=sysadmin ansible_become=true

[compute]
cloud-01 ansible_user=sysadmin ansible_become=true
cloud-02 ansible_user=sysadmin ansible_become=true
cloud-03 ansible_user=sysadmin ansible_become=true

[storage]
cloud-01 ansible_user=sysadmin ansible_become=true
cloud-02 ansible_user=sysadmin ansible_become=true
cloud-03 ansible_user=sysadmin ansible_become=true

[monitoring]
cloud-01 ansible_user=sysadmin ansible_become=true

[deployment]
localhost ansible_connection=local become=true

# Các group thừa kế (giữ nguyên từ sample)
[baremetal:children]
control
network
compute
storage
monitoring

[chrony-server:children]
control

[chrony:children]
baremetal

[hacluster:children]
control

[hacluster-remote:children]

[mariadb:children]
control

[rabbitmq:children]
control

[keystone:children]
control

[glance:children]
control

[nova:children]
control

[neutron:children]
network

[openvswitch:children]
network
compute

[ovn:children]
network
compute

[ovn-controller:children]
ovn

[ovn-nb-db:children]
control

[ovn-sb-db:children]
control

[ovn-northd:children]
control

[cinder:children]
control

[horizon:children]
control

[heat:children]
control

[placement:children]
control

[nova-compute:children]
compute

[ceph:children]
storage

[ceph-mon:children]
control

[ceph-mgr:children]
control

[ceph-osd:children]
storage

[ceph-rgw:children]

[memcached:children]
control

[haproxy:children]
network

[keepalived:children]
haproxy
```

Kiểm tra inventory:
```bash
source ~/kolla-venv/bin/activate
ansible -i ~/multinode all -m ping
```

Tất cả node phải trả về `SUCCESS`.

#### 4.5 globals.yml

Đây là file cấu hình quan trọng nhất. Chỉnh sửa `/etc/kolla/globals.yml`:

```yaml
---
# Kolla options
kolla_base_distro: "ubuntu"
openstack_release: "2023.2"
kolla_internal_vip_address: "10.10.200.10"
node_custom_config: "/etc/kolla/config"

# Docker
docker_registry: "quay.io"
docker_namespace: "openstack.kolla"

# Network interfaces
network_interface: "ens160"
neutron_external_interface: "ens192"
storage_interface: "ens224"
tunnel_interface: "ens160"

# Neutron - OVN
neutron_plugin_agent: "ovn"
neutron_ovn_distributed_fip: "yes"

# Nova
nova_compute_virt_type: "kvm"
nova_backend_ceph: "yes"

# Ceph (deployed by Kolla)
enable_ceph: "yes"
enable_ceph_dashboard: "no"
ceph_pool_type: "replicated"
ceph_target_max_bytes: ""
ceph_target_max_objects: ""

# Glance
glance_backend_ceph: "yes"
glance_backend_file: "no"

# Cinder
enable_cinder: "yes"
cinder_backend_ceph: "yes"
cinder_backup_driver: "ceph"

# Horizon
enable_horizon: "yes"

# Heat
enable_heat: "yes"

# HAProxy + Keepalived
enable_haproxy: "yes"
enable_keepalived: "yes"

# Logging
enable_central_logging: "no"

# Misc
enable_openstack_core: "yes"
```

> **Lưu ý:** Nếu nested virtualization không hoạt động, thay `nova_compute_virt_type: "kvm"` thành `"qemu"`.

#### 4.6 Ceph Configuration

Tạo thư mục config cho Ceph:
```bash
mkdir -p /etc/kolla/config/ceph
```

Tạo file `/etc/kolla/config/ceph.conf`:
```ini
[global]
# Ceph network
public network = 10.10.201.0/24
cluster network = 10.10.201.0/24

# Performance tuning cho lab
osd pool default size = 3
osd pool default min size = 2
osd pool default pg num = 128
osd pool default pgp num = 128

# BlueStore
osd objectstore = bluestore
```

> Trong lab này, **public network** và **cluster network** dùng chung subnet 10.10.201.0/24 (Storage network - Host-Only). Trong production, nên tách riêng 2 network này.

#### 4.7 Generate Passwords

```bash
source ~/kolla-venv/bin/activate
kolla-genpwd
```

File password được tạo tại `/etc/kolla/passwords.yml`. Lưu giữ file này cẩn thận.

Kiểm tra password Horizon admin:
```bash
grep keystone_admin_password /etc/kolla/passwords.yml
```

---

### 5. Deploy OpenStack + Ceph

> **Tất cả lệnh chạy trên cloud-01** trong virtual environment

```bash
source ~/kolla-venv/bin/activate
```

#### 5.1 Bootstrap Servers

Bootstrap cài đặt Docker, configure hệ thống trên tất cả node:
```bash
kolla-ansible -i ~/multinode bootstrap-servers
```

> Bước này mất khoảng 5-10 phút. Kolla sẽ cài Docker, cấu hình Docker daemon, cài các package cần thiết trên tất cả node.

Kiểm tra Docker đã chạy trên tất cả node:
```bash
ansible -i ~/multinode all -m shell -a "docker --version && systemctl is-active docker"
```

#### 5.2 Prechecks

Kiểm tra tất cả điều kiện trước khi deploy:
```bash
kolla-ansible -i ~/multinode prechecks
```

> Nếu có lỗi, đọc kỹ thông báo và sửa trước khi tiếp tục. Các lỗi phổ biến:
> - Docker chưa start → `systemctl start docker`
> - NIC không tồn tại → kiểm tra lại tên interface
> - Disk label không đúng → kiểm tra `parted` lại
> - Port đã bị chiếm → `ss -tlnp | grep <port>`

#### 5.3 Pull Images (tùy chọn)

Pull Docker images trước để giảm thời gian deploy:
```bash
kolla-ansible -i ~/multinode pull
```

#### 5.4 Deploy

Deploy toàn bộ OpenStack + Ceph:
```bash
kolla-ansible -i ~/multinode deploy
```

> **Thời gian deploy:** 30-60 phút tùy tốc độ mạng và disk I/O. Kolla sẽ deploy theo thứ tự:
> 1. Infrastructure (MariaDB, RabbitMQ, Memcached, HAProxy, Keepalived)
> 2. Ceph (MON → MGR → OSD)
> 3. Keystone
> 4. Glance, Nova, Neutron, Cinder, Heat, Horizon

Nếu deploy thất bại giữa chừng, sửa lỗi rồi chạy lại:
```bash
kolla-ansible -i ~/multinode deploy
```
Kolla-Ansible có tính idempotent - chạy lại sẽ không ảnh hưởng các service đã deploy thành công.

#### 5.5 Post-deploy

Tạo file cấu hình OpenStack client:
```bash
kolla-ansible -i ~/multinode post-deploy
```

File `admin-openrc.sh` được tạo tại `/etc/kolla/admin-openrc.sh`.

Cài đặt OpenStack CLI:
```bash
pip install python-openstackclient python-heatclient
```

Source credentials:
```bash
source /etc/kolla/admin-openrc.sh
```

---

### 6. Kiểm tra dịch vụ

#### 6.1 Ceph Health

```bash
sudo docker exec ceph_mon ceph -s
```

Output mong đợi:
```
  cluster:
    id:     xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum cloud-01,cloud-02,cloud-03
    mgr: cloud-01(active), standbys: cloud-02, cloud-03
    osd: 6 osds: 6 up, 6 in

  data:
    pools:   4 pools, 512 pgs
    objects: 0 objects, 0 B
    usage:   6.0 GiB used, 2.93 TiB / 2.93 TiB avail
    pgs:     512 active+clean
```

Kiểm tra OSD:
```bash
sudo docker exec ceph_mon ceph osd tree
```

Kiểm tra pools:
```bash
sudo docker exec ceph_mon ceph osd pool ls detail
```

#### 6.2 OpenStack Services

```bash
source /etc/kolla/admin-openrc.sh

# Kiểm tra service catalog
openstack service list

# Kiểm tra endpoints
openstack endpoint list

# Kiểm tra compute services
openstack compute service list

# Kiểm tra network agents
openstack network agent list

# Kiểm tra volume services
openstack volume service list

# Kiểm tra hypervisors
openstack hypervisor list
```

Tất cả services phải ở trạng thái `enabled` và `up`.

#### 6.3 Horizon Dashboard

Truy cập Horizon qua trình duyệt:
```
http://10.10.200.10
```

- **Domain:** default
- **User:** admin
- **Password:** (lấy từ `grep keystone_admin_password /etc/kolla/passwords.yml`)

<img src="/assets/img/2026-04-17-openstack-bobcat-kolla-private-cloud/04-horizon-login.png"/>

<img src="/assets/img/2026-04-17-openstack-bobcat-kolla-private-cloud/05-horizon-overview.png"/>

---

### 7. Cấu hình Multi-tenant cho doanh nghiệp

#### 7.1 Tạo Project

```bash
source /etc/kolla/admin-openrc.sh

# Tạo project cho phòng IT-Helpdesk
openstack project create --domain default \
  --description "Phong IT-Helpdesk - Quan ly ha tang noi bo" \
  IT-Helpdesk

# Tạo project cho phòng Dev
openstack project create --domain default \
  --description "Phong Dev - Moi truong phat trien" \
  Dev
```

#### 7.2 Tạo User & gán Role

```bash
# Tạo user cho IT-Helpdesk
openstack user create --domain default \
  --project IT-Helpdesk \
  --password 'ITHelpdesk@2026' \
  it-admin

# Gán role member cho user it-admin trong project IT-Helpdesk
openstack role add --project IT-Helpdesk --user it-admin member

# Tạo user cho Dev
openstack user create --domain default \
  --project Dev \
  --password 'DevTeam@2026' \
  dev-admin

# Gán role member cho user dev-admin trong project Dev
openstack role add --project Dev --user dev-admin member
```

#### 7.3 Cấu hình Quota

```bash
# Quota cho IT-Helpdesk: 8 vCPU, 16GB RAM, 200GB storage
openstack quota set --cores 8 --ram 16384 \
  --gigabytes 200 --volumes 10 \
  --instances 5 --floating-ips 3 \
  --secgroups 5 --secgroup-rules 50 \
  IT-Helpdesk

# Quota cho Dev: 8 vCPU, 16GB RAM, 200GB storage
openstack quota set --cores 8 --ram 16384 \
  --gigabytes 200 --volumes 10 \
  --instances 5 --floating-ips 3 \
  --secgroups 5 --secgroup-rules 50 \
  Dev
```

Kiểm tra quota:
```bash
openstack quota show IT-Helpdesk
openstack quota show Dev
```

#### 7.4 Tạo External Network (Admin)

External network chỉ admin mới có thể tạo, dùng chung cho tất cả project:

```bash
source /etc/kolla/admin-openrc.sh

# Tạo external network (provider flat)
openstack network create --external \
  --provider-network-type flat \
  --provider-physical-network physnet1 \
  --share \
  external-net

# Tạo subnet cho external network
openstack subnet create --network external-net \
  --subnet-range 10.10.200.0/24 \
  --gateway 10.10.200.1 \
  --allocation-pool start=10.10.200.150,end=10.10.200.160 \
  --dns-nameserver 8.8.8.8 \
  --no-dhcp \
  external-subnet
```

#### 7.5 Tạo Internal Network cho từng Project

**IT-Helpdesk network:**
```bash
# Switch sang project IT-Helpdesk (dùng admin)
openstack network create --project IT-Helpdesk \
  it-internal-net

openstack subnet create --project IT-Helpdesk \
  --network it-internal-net \
  --subnet-range 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  --dns-nameserver 8.8.8.8 \
  it-internal-subnet

# Tạo router kết nối internal ↔ external
openstack router create --project IT-Helpdesk it-router
openstack router set --external-gateway external-net it-router
openstack router add subnet it-router it-internal-subnet
```

**Dev network:**
```bash
openstack network create --project Dev \
  dev-internal-net

openstack subnet create --project Dev \
  --network dev-internal-net \
  --subnet-range 192.168.200.0/24 \
  --gateway 192.168.200.1 \
  --dns-nameserver 8.8.8.8 \
  dev-internal-subnet

# Tạo router kết nối internal ↔ external
openstack router create --project Dev dev-router
openstack router set --external-gateway external-net dev-router
openstack router add subnet dev-router dev-internal-subnet
```

<img src="/assets/img/2026-04-17-openstack-bobcat-kolla-private-cloud/06-network-topology.png"/>

---

### 8. Demo - Tạo VM trong OpenStack

#### 8.1 Upload Image

```bash
source /etc/kolla/admin-openrc.sh

# Download Ubuntu Server 22.04 cloud image
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Upload lên Glance
openstack image create "Ubuntu-22.04" \
  --file jammy-server-cloudimg-amd64.img \
  --disk-format qcow2 \
  --container-format bare \
  --public
```

Cho Windows Server 2022, cần tạo image từ ISO hoặc dùng image có sẵn:
```bash
# Download virtio driver ISO (cần cho Windows trên KVM)
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso

# Upload Windows Server 2022 image (sau khi tạo từ ISO)
openstack image create "Windows-Server-2022" \
  --file windows-server-2022.qcow2 \
  --disk-format qcow2 \
  --container-format bare \
  --public \
  --property os_type=windows \
  --property hw_disk_bus=virtio \
  --property hw_vif_model=virtio
```

> **Tạo Windows image:** Cần boot VM từ ISO Windows Server 2022 + virtio driver, cài đặt xong rồi export thành qcow2. Có thể dùng `virt-install` hoặc tạo trực tiếp trong OpenStack.

#### 8.2 Tạo Flavor

```bash
# Flavor cho Ubuntu Server
openstack flavor create --vcpus 2 --ram 2048 --disk 20 m1.small

# Flavor cho Windows Server
openstack flavor create --vcpus 4 --ram 4096 --disk 60 m1.medium

# Flavor nhỏ cho test
openstack flavor create --vcpus 1 --ram 1024 --disk 10 m1.tiny
```

#### 8.3 Tạo Security Group

```bash
# Security Group cho IT-Helpdesk
openstack security group create --project IT-Helpdesk \
  --description "Allow SSH, ICMP, HTTP, HTTPS, RDP" \
  it-secgroup

openstack security group rule create --project IT-Helpdesk \
  --protocol icmp it-secgroup
openstack security group rule create --project IT-Helpdesk \
  --protocol tcp --dst-port 22 it-secgroup
openstack security group rule create --project IT-Helpdesk \
  --protocol tcp --dst-port 80 it-secgroup
openstack security group rule create --project IT-Helpdesk \
  --protocol tcp --dst-port 443 it-secgroup

# Security Group cho Dev
openstack security group create --project Dev \
  --description "Allow SSH, ICMP, HTTP, HTTPS, RDP" \
  dev-secgroup

openstack security group rule create --project Dev \
  --protocol icmp dev-secgroup
openstack security group rule create --project Dev \
  --protocol tcp --dst-port 22 dev-secgroup
openstack security group rule create --project Dev \
  --protocol tcp --dst-port 80 dev-secgroup
openstack security group rule create --project Dev \
  --protocol tcp --dst-port 443 dev-secgroup
openstack security group rule create --project Dev \
  --protocol tcp --dst-port 3389 dev-secgroup
```

#### 8.4 Tạo SSH Keypair

```bash
# Keypair cho IT-Helpdesk
openstack keypair create --type ssh \
  --user it-admin it-keypair > it-keypair.pem
chmod 600 it-keypair.pem

# Keypair cho Dev
openstack keypair create --type ssh \
  --user dev-admin dev-keypair > dev-keypair.pem
chmod 600 dev-keypair.pem
```

#### 8.5 Tạo VM Ubuntu Server (IT-Helpdesk)

```bash
# Tạo VM Ubuntu trong project IT-Helpdesk
openstack server create \
  --image "Ubuntu-22.04" \
  --flavor m1.small \
  --network it-internal-net \
  --security-group it-secgroup \
  --key-name it-keypair \
  --os-project-name IT-Helpdesk \
  --os-username it-admin \
  --os-password 'ITHelpdesk@2026' \
  --os-auth-url http://10.10.200.10:5000/v3 \
  --os-project-domain-name default \
  --os-user-domain-name default \
  it-ubuntu-srv01

# Gán Floating IP
openstack floating ip create --project IT-Helpdesk external-net
# Lấy floating IP vừa tạo
FLOAT_IP=$(openstack floating ip list --project IT-Helpdesk -f value -c "Floating IP Address" | head -1)
openstack server add floating ip it-ubuntu-srv01 $FLOAT_IP
```

Kiểm tra:
```bash
openstack server list --project IT-Helpdesk
openstack server show it-ubuntu-srv01
```

Truy cập VM:
```bash
ssh -i it-keypair.pem ubuntu@$FLOAT_IP
```

<img src="/assets/img/2026-04-17-openstack-bobcat-kolla-private-cloud/07-vm-ubuntu.png"/>

#### 8.6 Tạo VM Windows Server 2022 (Dev)

```bash
# Tạo volume từ Windows image (boot from volume cho disk lớn)
openstack volume create --size 60 \
  --image "Windows-Server-2022" \
  --bootable \
  --os-project-name Dev \
  win2022-boot-vol

# Tạo VM Windows trong project Dev
openstack server create \
  --volume win2022-boot-vol \
  --flavor m1.medium \
  --network dev-internal-net \
  --security-group dev-secgroup \
  --os-project-name Dev \
  --os-username dev-admin \
  --os-password 'DevTeam@2026' \
  --os-auth-url http://10.10.200.10:5000/v3 \
  --os-project-domain-name default \
  --os-user-domain-name default \
  dev-win2022-srv01

# Gán Floating IP
openstack floating ip create --project Dev external-net
FLOAT_IP_WIN=$(openstack floating ip list --project Dev -f value -c "Floating IP Address" | head -1)
openstack server add floating ip dev-win2022-srv01 $FLOAT_IP_WIN
```

Truy cập Windows VM qua RDP:
```
mstsc /v:$FLOAT_IP_WIN
```

Hoặc truy cập qua Horizon Console:

<img src="/assets/img/2026-04-17-openstack-bobcat-kolla-private-cloud/08-vm-windows.png"/>

#### 8.7 Tạo Cinder Volume & Attach

Demo tạo thêm volume data cho VM:
```bash
# Tạo volume 50GB trong project Dev
openstack volume create --size 50 \
  --os-project-name Dev \
  dev-data-vol01

# Attach volume vào Windows VM
openstack server add volume dev-win2022-srv01 dev-data-vol01
```

Kiểm tra volume trên Ceph:
```bash
sudo docker exec ceph_mon rbd ls volumes
sudo docker exec ceph_mon ceph df
```

<img src="/assets/img/2026-04-17-openstack-bobcat-kolla-private-cloud/09-ceph-dashboard.png"/>

---

### 9. Scale & Best Practices

#### 9.1 Thêm Compute Node

Khi cần thêm tài nguyên compute, chỉ cần:

1. Tạo VM mới trên ESXi (cùng cấu hình, không cần disk OSD)
2. Cài Ubuntu 24.04, cấu hình network
3. Thêm node vào inventory group `[compute]`:
   ```ini
   [compute]
   cloud-01 ansible_user=sysadmin ansible_become=true
   cloud-02 ansible_user=sysadmin ansible_become=true
   cloud-03 ansible_user=sysadmin ansible_become=true
   cloud-04 ansible_user=sysadmin ansible_become=true  # node mới
   ```
4. Chạy lại:
   ```bash
   kolla-ansible -i ~/multinode bootstrap-servers --limit cloud-04
   kolla-ansible -i ~/multinode deploy
   ```

#### 9.2 Thêm Ceph OSD Node

Tương tự, thêm node mới vào `[storage]` và `[ceph-osd]`, chuẩn bị disk label, rồi deploy lại.

#### 9.3 Tách Role (Production)

Khi scale lên production, nên tách role:

| Role | Số node khuyến nghị | Ghi chú |
|------|---------------------|---------|
| Control | 3 | HA cho API, DB, MQ |
| Network | 2-3 | Neutron agents |
| Compute | N (theo nhu cầu) | Nova compute |
| Storage (Ceph) | 3+ | MON + OSD |

#### 9.4 Best Practices

- **Backup:** Định kỳ backup `/etc/kolla/` (đặc biệt `passwords.yml` và `globals.yml`)
- **Monitoring:** Bật `enable_prometheus: "yes"` và `enable_grafana: "yes"` trong globals.yml
- **Ceph tuning:** Điều chỉnh PG count theo số OSD, sử dụng `ceph balancer` 
- **Security:** Bật TLS cho API endpoints (`kolla_enable_tls_external: "yes"`)
- **Update:** Kolla-Ansible hỗ trợ rolling upgrade giữa các phiên bản OpenStack

---

### 10. Kết luận

Trong bài lab này, chúng ta đã triển khai thành công một hệ thống **Private Cloud** hoàn chỉnh với:

- **OpenStack Bobcat (2023.2)** trên 3 node All-in-One với HA cho control plane
- **Ceph** tích hợp làm unified storage backend (Glance, Cinder, Nova)
- **OVN** cho networking với self-service + provider network
- **Multi-tenant** cho 2 phòng ban IT-Helpdesk và Dev với quota riêng biệt
- Demo tạo VM **Ubuntu Server** và **Windows Server 2022**

Mô hình 3 node All-in-One phù hợp cho PoC và môi trường nhỏ. Khi doanh nghiệp cần scale, chỉ cần thêm compute node hoặc tách role mà không ảnh hưởng đến hệ thống đang chạy.

**Tài nguyên tham khảo:**
- [OpenStack Kolla-Ansible Documentation](https://docs.openstack.org/kolla-ansible/2023.2/)
- [Ceph Documentation](https://docs.ceph.com/en/reef/)
- [OVN Architecture](https://docs.openstack.org/neutron/2023.2/admin/ovn/index.html)
