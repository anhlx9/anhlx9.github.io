---
title: "OpenStack All-in-One trên 1 VM"
categories:
- Cloud
- OpenStack

feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Hướng dẫn cài đặt OpenStack All-in-One trên 1 VM sử dụng DevStack
---

### Mục lục

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Yêu cầu hệ thống](#2-yêu-cầu-hệ-thống)
- [3. Chuẩn bị VM](#3-chuẩn-bị-vm)
  - [3.1. Tạo user stack](#31-tạo-user-stack)
  - [3.2. Update hệ thống](#32-update-hệ-thống)
- [4. Cài đặt DevStack](#4-cài-đặt-devstack)
- [5. Cấu hình local.conf](#5-cấu-hình-localconf)
- [6. Chạy stack.sh](#6-chạy-stacksh)
- [7. Truy cập Horizon Dashboard](#7-truy-cập-horizon-dashboard)
- [8. Kiểm tra dịch vụ](#8-kiểm-tra-dịch-vụ)
  - [8.1. Load credentials](#81-load-credentials)
  - [8.2. Kiểm tra Compute](#82-kiểm-tra-compute)
  - [8.3. Kiểm tra Network](#83-kiểm-tra-network)
  - [8.4. Kiểm tra Image](#84-kiểm-tra-image)
  - [8.5. Tạo VM test](#85-tạo-vm-test)

---

## 1. Giới thiệu

**OpenStack** là nền tảng cloud infrastructure mã nguồn mở, cho phép xây dựng private cloud với đầy đủ tính năng: compute, networking, storage, identity...

**DevStack** là công cụ cài đặt OpenStack nhanh trên 1 máy (All-in-One), thường dùng cho môi trường lab/dev.

Bài viết này hướng dẫn triển khai OpenStack All-in-One trên 1 VM Ubuntu Server 24.04 chạy trên VMware ESXi sử dụng DevStack.

---

## 2. Yêu cầu hệ thống

| Thành phần | Cấu hình lab |
|------------|-------------------|
| Hypervisor | VMware ESXi |
| OS | Ubuntu Server 24.04.4 LTS (Noble Numbat) |
| Hostname | vm1 |
| CPU | 8 vCPU |
| RAM | 8 GB |
| Disk | 80 GB |
| IP | 10.10.200.11/24 (ens160) |

> **Lưu ý:** Cần bật **Hardware Virtualization** (VT-x/AMD-V) cho VM trên ESXi: VM Settings → CPU → Enable hardware virtualization.

---

## 3. Chuẩn bị VM

### 3.1. Tạo user stack

DevStack **không chạy với user root**. Cần tạo user riêng:

```bash
sudo useradd -s /bin/bash -d /opt/stack -m stack
sudo chmod +x /opt/stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo -u stack -i
```

### 3.2. Update hệ thống

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl
```

---

## 4. Cài đặt DevStack

Clone DevStack từ GitHub (branch **stable/2024.1** - Caracal):

```bash
cd /opt/stack
git clone https://opendev.org/openstack/devstack -b stable/2024.1
cd devstack
```

---

## 5. Cấu hình local.conf

Tạo file cấu hình `local.conf` trong thư mục devstack:

```bash
cat > /opt/stack/devstack/local.conf << 'EOF'
[[local|localrc]]
# Mật khẩu cho các service (đặt giống nhau cho đơn giản)
ADMIN_PASSWORD=Admin@123
DATABASE_PASSWORD=Admin@123
RABBIT_PASSWORD=Admin@123
SERVICE_PASSWORD=Admin@123

# IP của máy chủ
HOST_IP=10.10.200.11

# Bật các service cơ bản
enable_service rabbit
enable_service mysql
enable_service key

# Nova - Compute
enable_service n-api
enable_service n-cpu
enable_service n-cond
enable_service n-sch
enable_service n-novnc
enable_service n-api-meta
enable_service placement-api

# Glance - Image
enable_service g-api

# Neutron - Network
enable_service neutron
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
enable_service q-meta

# Horizon - Dashboard
enable_service horizon

# Cinder - Block Storage (tuỳ chọn)
# enable_service cinder
# enable_service c-api
# enable_service c-vol
# enable_service c-sch

# Log
LOGFILE=/opt/stack/logs/stack.sh.log
LOGDAYS=2

# Nova sử dụng KVM (ESXi đã bật Hardware Virtualization cho VM)
LIBVIRT_TYPE=kvm
EOF
```

> `HOST_IP=10.10.200.11` là IP của VM lab (interface `ens160`).

---

## 6. Chạy stack.sh

```bash
cd /opt/stack/devstack
./stack.sh
```

Quá trình cài đặt mất **30–60 phút** tuỳ tốc độ internet và cấu hình máy. Script sẽ tự động:

- Cài đặt các gói phụ thuộc
- Tải source code các service OpenStack
- Cấu hình database, message queue
- Khởi động tất cả service

Khi hoàn tất, output sẽ hiển thị:

```
This is your host IP address: 10.10.200.11
This is your host IPv6 address: ::1
Horizon is now available at http://10.10.200.11/dashboard
Keystone is serving at http://10.10.200.11/identity/
The default users are: admin and demo
The password: Admin@123

DevStack Version: 2024.1
Change: ...
OS Version: Ubuntu 24.04
```

---

## 7. Truy cập Horizon Dashboard

Mở trình duyệt truy cập:

```
http://10.10.200.11/dashboard
```

Đăng nhập với:
- **Domain:** `default`
- **Username:** `admin`
- **Password:** `Admin@123` (theo cấu hình `ADMIN_PASSWORD`)

![Horizon Login](../assets/img/2026-04-15-openstack-all-in-one/horizon-login.png)

---

## 8. Kiểm tra dịch vụ

### 8.1. Load credentials

```bash
source /opt/stack/devstack/openrc admin admin
```

### 8.2. Kiểm tra Compute

```bash
openstack compute service list
```

Output mẫu:

```
+--------------------------------------+----------------+------------+----------+---------+-------+
| ID                                   | Binary         | Host       | Zone     | Status  | State |
+--------------------------------------+----------------+------------+----------+---------+-------+
| ...                                  | nova-conductor | controller | internal | enabled | up    |
| ...                                  | nova-scheduler | controller | internal | enabled | up    |
| ...                                  | nova-compute   | controller | nova     | enabled | up    |
+--------------------------------------+----------------+------------+----------+---------+-------+
```

### 8.3. Kiểm tra Network

```bash
openstack network agent list
```

### 8.4. Kiểm tra Image

```bash
openstack image list
```

DevStack mặc định tải sẵn image **Cirros** (image nhỏ để test):

```
+--------------------------------------+--------------------------+--------+
| ID                                   | Name                     | Status |
+--------------------------------------+--------------------------+--------+
| ...                                  | cirros-0.6.2-x86_64-disk | active |
+--------------------------------------+--------------------------+--------+
```

### 8.5. Tạo VM test

```bash
# Lấy thông tin cần thiết
openstack flavor list
openstack network list

# Tạo VM
openstack server create \
  --image cirros-0.6.2-x86_64-disk \
  --flavor m1.tiny \
  --network private \
  test-vm

# Kiểm tra trạng thái
openstack server list
```

---

> **Lưu ý sau reboot:** DevStack không tự khởi động lại sau khi reboot VM. Muốn khởi động lại, chạy `./stack.sh` hoặc sử dụng `./rejoin-stack.sh` (nếu services còn config).
