---
title: "OpenStack All-in-One với Kolla-Ansible"
categories:
- Cloud
- OpenStack

feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Triển khai OpenStack All-in-One trên 1 VM sử dụng Kolla-Ansible — gần với production thực tế
---

### Mục lục

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Cấu hình lab](#2-cấu-hình-lab)
- [3. Chuẩn bị hệ thống](#3-chuẩn-bị-hệ-thống)
  - [3.1. Chuẩn bị NIC2 cho external network](#31-chuẩn-bị-nic2-cho-external-network)
  - [3.2. Chuẩn bị LVM cho Cinder](#32-chuẩn-bị-lvm-cho-cinder)
  - [3.3. Cài Docker](#33-cài-docker)
  - [3.4. Cài Python dependencies](#34-cài-python-dependencies)
- [4. Cài đặt Kolla-Ansible](#4-cài-đặt-kolla-ansible)
- [5. Cấu hình Kolla-Ansible](#5-cấu-hình-kolla-ansible)
  - [5.1. globals.yml](#51-globalsyml)
  - [5.2. passwords.yml](#52-passwordsyml)
- [6. Deploy OpenStack](#6-deploy-openstack)
  - [Bước 1 — Bootstrap servers](#bước-1--bootstrap-servers)
  - [Bước 2 — Pre-checks](#bước-2--pre-checks)
  - [Bước 3 — Pull images](#bước-3--pull-images)
  - [Bước 4 — Deploy](#bước-4--deploy)
- [7. Post-deploy](#7-post-deploy)
- [8. Truy cập Horizon Dashboard](#8-truy-cập-horizon-dashboard)
- [9. Kiểm tra dịch vụ](#9-kiểm-tra-dịch-vụ)
  - [9.1. Kiểm tra containers](#91-kiểm-tra-containers)
  - [9.2. Kiểm tra Compute](#92-kiểm-tra-compute)
  - [9.3. Tạo network và VM test](#93-tạo-network-và-vm-test)
- [10. Quản lý sau deploy](#10-quản-lý-sau-deploy)
  - [Containers tự khởi động sau reboot](#containers-tự-khởi-động-sau-reboot)
  - [Upgrade lên version mới](#upgrade-lên-version-mới)
  - [Destroy và deploy lại từ đầu](#destroy-và-deploy-lại-từ-đầu)

---

## 1. Giới thiệu

**Kolla-Ansible** là công cụ deploy OpenStack chính thức, được dùng rộng rãi trong production. Mỗi service OpenStack chạy trong **Docker container** riêng biệt, managed bởi Ansible playbook.

So với DevStack, Kolla-Ansible có các ưu điểm:
- Tự khởi động lại sau reboot (systemd + Docker)
- Kiến trúc container giống production
- Hỗ trợ rolling upgrade
- Có thể mở rộng từ AIO lên multi-node chỉ bằng cách thêm host vào inventory

Bài viết này triển khai **OpenStack 2025.2 (Flamingo)** — All-in-One trên 1 VM.

![](/assets/img/2026-04-15-openstack-all-in-one/00.png)

---

## 2. Cấu hình lab

| Thành phần | Cấu hình |
|------------|----------|
| Hypervisor | VMware ESXi |
| OS | Ubuntu Server 24.04.4 LTS (Noble Numbat) |
| Hostname | openstack-AIO |
| CPU | 16 vCPU |
| RAM | 32 GB |
| Disk OS | 100 GB (`/dev/sda`) |
| Disk Cinder | 500 GB (`/dev/sdb`) |
| NIC1 — Management | 10.10.200.11/24 (`ens160`, VLAN 200) |
| NIC2 — External | no IP (`ens192`, VLAN 200) — Neutron provider |
| Floating IP pool | 10.10.200.100 – 10.10.200.200 (VLAN 200) |
| Gateway / NAT | 10.10.200.1 (pfSense → internet) |
| OpenStack version | 2025.2 — Flamingo |

> **Bật Hardware Virtualization** trên ESXi: VM Settings → CPU → Enable hardware virtualization (để dùng `virt_type=kvm`).

---

## 3. Chuẩn bị hệ thống

### 3.1. Chuẩn bị NIC2 cho external network

Trên ESXi, thêm **Network Adapter 2** cho VM, gắn vào portgroup **VLAN 200** (cùng portgroup với NIC1).

Sau khi boot VM, `ens192` sẽ xuất hiện — **không cần cấu hình IP**, để trống cho OVN bridge:

```bash
# Kiểm tra NIC2 đã nhận diện
ip -br a
# Output mong đợi:
# ens160  UP  10.10.200.11/24
# ens192  UP  (no IP)

# Đảm bảo ens192 up
sudo ip link set ens192 up
```

Để `ens192` tự up sau reboot, tạo file netplan:

```bash
sudo tee /etc/netplan/99-ens192.yaml << 'EOF'
network:
  version: 2
  ethernets:
    ens192:
      dhcp4: false
      dhcp6: false
EOF

sudo chmod 600 /etc/netplan/99-ens192.yaml
sudo netplan apply
```

### 3.2. Chuẩn bị LVM cho Cinder

Cinder sử dụng **LVM backend** với disk `/dev/sdb` (500 GB):

```bash
# Tạo Physical Volume và Volume Group
sudo pvcreate /dev/sdb
sudo vgcreate cinder-volumes /dev/sdb

# Kiểm tra
sudo vgs
# Output:
# VG             #PV #LV #SN Attr   VSize    VFree
# cinder-volumes   1   0   0 wz--n- 500.00g  500.00g
```

![](/assets/img/2026-04-15-openstack-all-in-one/01.png)

### 3.3. Cài Docker

```bash
# Cài dependencies
sudo apt update
sudo apt install -y ca-certificates curl

# Thêm Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Thêm Docker repo
echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Thêm user hiện tại vào group docker
sudo usermod -aG docker $USER
newgrp docker

# Kiểm tra
docker info
```

### 3.4. Cài Python dependencies

```bash
sudo apt install -y python3-dev python3-pip python3-venv \
  libffi-dev gcc libssl-dev git

# Tạo virtual environment cho Kolla-Ansible
python3 -m venv /opt/kolla-venv
source /opt/kolla-venv/bin/activate

# Upgrade pip
pip install -U pip
```

---

## 4. Cài đặt Kolla-Ansible

```bash
# Vẫn trong venv
source /opt/kolla-venv/bin/activate

# Cài Kolla-Ansible stable/2025.2
pip install git+https://opendev.org/openstack/kolla-ansible@stable/2025.2

# Cài Ansible collections cần thiết
kolla-ansible install-deps

# Kiểm tra version
kolla-ansible --version
```
![](/assets/img/2026-04-15-openstack-all-in-one/02.png)

Tạo thư mục cấu hình:

```bash
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla

# Copy file template
cp /opt/kolla-venv/share/kolla-ansible/etc_examples/kolla/globals.yml /etc/kolla/globals.yml
cp /opt/kolla-venv/share/kolla-ansible/etc_examples/kolla/passwords.yml /etc/kolla/passwords.yml

# Copy inventory all-in-one
cp /opt/kolla-venv/share/kolla-ansible/ansible/inventory/all-in-one ~/all-in-one
```

---

## 5. Cấu hình Kolla-Ansible

### 5.1. globals.yml

```bash
sudo tee /etc/kolla/globals.yml << 'EOF'
---
kolla_base_distro: "ubuntu"
openstack_release: "2025.2"

network_interface: "ens160"
neutron_external_interface: "ens192"
kolla_internal_vip_address: "10.10.200.11"

nova_compute_virt_type: "kvm"
neutron_plugin_agent: "ovn"
enable_neutron_provider_networks: "yes"
enable_haproxy: "no"
enable_proxysql: "no"

enable_openstack_core: "yes"
enable_glance: "yes"
enable_nova: "yes"
enable_neutron: "yes"
enable_horizon: "yes"

enable_cinder: "yes"
enable_cinder_backup: "no"
enable_cinder_backend_lvm: "yes"
cinder_volume_group: "cinder-volumes"

enable_swift: "no"
enable_heat: "no"
enable_barbican: "no"
enable_designate: "no"
enable_octavia: "no"
enable_manila: "no"
EOF
```

### 5.2. passwords.yml

Sinh password tự động:

```bash
kolla-genpwd
```

Kiểm tra password admin:

```bash
grep keystone_admin_password /etc/kolla/passwords.yml
```
![](/assets/img/2026-04-15-openstack-all-in-one/03.png)

---

## 6. Deploy OpenStack

### Bước 1 — Bootstrap servers

Cài đặt các package cần thiết trên host:

```bash
source /opt/kolla-venv/bin/activate
kolla-ansible bootstrap-servers -i ~/all-in-one
```

### Bước 2 — Pre-checks

Kiểm tra điều kiện trước khi deploy:

```bash
kolla-ansible prechecks -i ~/all-in-one
```

Nếu có lỗi, đọc kỹ thông báo và sửa trước khi tiếp tục.

```bash
# Lỗi ModuleNotFoundError: No module named 'docker'
pip install docker

# Lỗi No module named 'dbus'
sudo apt install -y python3-dbus

# Symlink dbus vào venv
PYVER=$(python3 -c "import sys; print(f'{sys.version_info.major}.{sys.version_info.minor}')")
ln -s /usr/lib/python3/dist-packages/dbus /opt/kolla-venv/lib/python${PYVER}/site-packages/dbus
ln -s /usr/lib/python3/dist-packages/_dbus_bindings*.so /opt/kolla-venv/lib/python${PYVER}/site-packages/
ln -s /usr/lib/python3/dist-packages/_dbus_glib_bindings*.so /opt/kolla-venv/lib/python${PYVER}/site-packages/
```

### Bước 3 — Pull images

```bash
kolla-ansible pull -i ~/all-in-one
```
![](/assets/img/2026-04-15-openstack-all-in-one/04.png)

### Bước 4 — Deploy

```bash
kolla-ansible deploy -i ~/all-in-one
```

![](/assets/img/2026-04-15-openstack-all-in-one/05.png)

---

## 7. Post-deploy

```bash
kolla-ansible post-deploy -i ~/all-in-one
```

Lệnh này tạo file `/etc/kolla/admin-openrc.sh`. Load credentials và cài CLI:

```bash
source /etc/kolla/admin-openrc.sh
pip install python-openstackclient


# Kiểm tra kết nối
openstack token issue
```
![](/assets/img/2026-04-15-openstack-all-in-one/06.png)
![](/assets/img/2026-04-15-openstack-all-in-one/07.png)

---

## 8. Truy cập Horizon Dashboard

Mở trình duyệt truy cập:

```
http://10.10.200.11/
```

Đăng nhập với:
- **Username:** `admin`
- **Password:** giá trị `keystone_admin_password` trong `/etc/kolla/passwords.yml`
  
```bash
grep keystone_admin_password /etc/kolla/passwords.yml
```

![](/assets/img/2026-04-15-openstack-all-in-one/08.png)

![](/assets/img/2026-04-15-openstack-all-in-one/09.png)

---

## 9. Kiểm tra dịch vụ

### 9.1. Kiểm tra containers

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

Tất cả container phải ở trạng thái `Up`:

![](/assets/img/2026-04-15-openstack-all-in-one/10.png)

### 9.2. Kiểm tra Compute

```bash
openstack compute service list
```

![](/assets/img/2026-04-15-openstack-all-in-one/11.png)


### 9.3. Tạo network và VM test

```bash
# Tạo external provider network
openstack network create \
  --provider-network-type flat \
  --provider-physical-network physnet1 \
  --external \
  public

openstack subnet create \
  --network public \
  --subnet-range 10.10.200.0/24 \
  --gateway 10.10.200.1 \
  --dns-nameserver 8.8.8.8 \
  --allocation-pool start=10.10.200.100,end=10.10.200.200 \
  --no-dhcp \
  public-subnet

# Tạo private network
openstack network create private
openstack subnet create \
  --network private \
  --subnet-range 192.168.100.0/24 \
  private-subnet

# Tạo router
openstack router create router1
openstack router set router1 --external-gateway public
openstack router add subnet router1 private-subnet

# Upload cirros image
wget https://github.com/cirros-dev/cirros/releases/download/0.6.2/cirros-0.6.2-x86_64-disk.img
openstack image create cirros \
  --disk-format qcow2 \
  --container-format bare \
  --public \
  --file cirros-0.6.2-x86_64-disk.img

# Tạo flavor
openstack flavor create m1.tiny --vcpus 1 --ram 512 --disk 1

# Tạo VM test
openstack server create \
  --image cirros \
  --flavor m1.tiny \
  --network private \
  test-vm

openstack server list
```

![](/assets/img/2026-04-15-openstack-all-in-one/12.png)
![](/assets/img/2026-04-15-openstack-all-in-one/13.png)
![](/assets/img/2026-04-15-openstack-all-in-one/14.png)
![](/assets/img/2026-04-15-openstack-all-in-one/15.png)
![](/assets/img/2026-04-15-openstack-all-in-one/16.png)
![](/assets/img/2026-04-15-openstack-all-in-one/17.png)
![](/assets/img/2026-04-15-openstack-all-in-one/18.png)
![](/assets/img/2026-04-15-openstack-all-in-one/19.png)
![](/assets/img/2026-04-15-openstack-all-in-one/20.png)
![](/assets/img/2026-04-15-openstack-all-in-one/21.png)


---

## 10. Quản lý sau deploy

### Containers tự khởi động sau reboot

```bash
# Kiểm tra sau reboot
docker ps --format "table {{.Names}}\t{{.Status}}"
```

### Upgrade lên version mới

```bash
# Đổi openstack_release trong globals.yml, sau đó:
kolla-ansible pull -i ~/all-in-one
kolla-ansible upgrade -i ~/all-in-one
```

### Destroy và deploy lại từ đầu

```bash
kolla-ansible destroy --yes-i-really-really-mean-it -i ~/all-in-one
kolla-ansible deploy -i ~/all-in-one
```
