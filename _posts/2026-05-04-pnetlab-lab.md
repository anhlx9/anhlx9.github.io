---
title: "PNetLab — Network Emulator Lab"
categories:
- Network

feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Xây dựng môi trường lab mạng với PNetLab trên VMware ESXi / Proxmox
---

Bài viết hướng dẫn cài đặt và sử dụng **PNetLab** — nền tảng giả lập mạng mã nguồn mở cho phép chạy các thiết bị mạng ảo (Cisco IOS, Cisco XE, Cisco XR, Juniper, Palo Alto, MikroTik, v.v.) trực tiếp trên trình duyệt web mà không cần cài đặt thêm phần mềm.

<img src="/assets/img/2026-05-04-pnetlab-lab/00.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/28.png"/>

### Mục lục

- [Mục lục](#mục-lục)
- [1. Giới thiệu](#1-giới-thiệu)
- [2. Yêu cầu hệ thống](#2-yêu-cầu-hệ-thống)
- [3. Cài đặt PNetLab VM](#3-cài-đặt-pnetlab-vm)
  - [3.1. Import OVA vào VMware ESXi / Workstation](#31-import-ova-vào-vmware-esxi--workstation)
  - [3.2. Cấu hình ban đầu](#32-cấu-hình-ban-đầu)
- [4. Truy cập Web UI](#4-truy-cập-web-ui)
- [5. Upload Node Images](#5-upload-node-images)
  - [5.1. Cisco vIOS L3 (Router)](#51-cisco-vios-l3-router)
  - [5.2. Cisco vIOS L2 (Switch)](#52-cisco-vios-l2-switch)
  - [5.3. Cisco CSR1000v (IOS XE)](#53-cisco-csr1000v-ios-xe)
  - [5.4. FortiGate](#54-fortigate)
- [6. Tạo Lab Topology](#6-tạo-lab-topology)
  - [6.1. Tạo Lab mới](#61-tạo-lab-mới)
  - [6.2. Thêm Node vào Lab](#62-thêm-node-vào-lab)
  - [6.3. Thêm Cloud0 (Internet)](#63-thêm-cloud0-internet)
  - [6.4. Kết nối các Node](#64-kết-nối-các-node)
  - [6.5. Khởi động và cấu hình](#65-khởi-động-và-cấu-hình)
- [7. Lời kết](#7-lời-kết)

---

### 1. Giới thiệu

**PNetLab** (viết tắt của *Packet Network Emulator Lab*) là nền tảng giả lập mạng kế thừa từ dự án UNetLab/EVE-NG, được phát triển lại với nhiều cải tiến:

- **Web UI hiện đại** — Kéo thả topology trực tiếp trên trình duyệt, không cần Java hay Flash
- **Multi-user** — Hỗ trợ nhiều user đồng thời làm lab trên cùng một server
- **Store tích hợp** — Download node image trực tiếp từ PNetLab Store
- **Docker node** — Hỗ trợ chạy Linux container (FRRouting, Ubuntu, v.v.) không cần license
- **Wireshark tích hợp** — Bắt gói tin trực tiếp từ Web UI

So sánh PNetLab với các giải pháp tương tự:

| Tiêu chí | PNetLab | EVE-NG CE | GNS3 |
|---|---|---|---|
| Giao diện | Web | Web | Desktop App |
| Multi-user | Có | Không (CE) | Không |
| Docker node | Có | Không (CE) | Có |
| Giá | Miễn phí | Miễn phí | Miễn phí |
| Image store | Có | Không | Không |
| Cài đặt | OVA/ISO | OVA/ISO | App + Server |


---

### 2. Yêu cầu hệ thống

| Thành phần | Tối thiểu | Khuyến nghị |
|---|---|---|
| CPU | 4 cores (VT-x/AMD-V) | 8+ cores |
| RAM | 8 GB | 16–32 GB |
| Disk | 100 GB | 200+ GB (SSD) |
| Network | 1 NIC | 2 NICs |
| Hypervisor | VMware ESXi 6.5+, Proxmox 7+, VMware Workstation 16+ | ESXi 7/8 hoặc Proxmox 8 |

> **Lưu ý:** PNetLab yêu cầu **nested virtualization** được bật trên hypervisor để chạy các node KVM bên trong.

---

### 3. Cài đặt PNetLab VM

Download OVA từ trang chính thức: [https://pnetlab.com/pages/download](https://pnetlab.com/pages/download)

#### 3.1. Import OVA vào VMware ESXi / Workstation

1. Truy cập **vSphere Client** → **File** → **Deploy OVF Template**
2. Chọn file `pnetlab-x.x.x.ova` vừa download
3. Chọn Datastore và Network phù hợp
4. Sau khi import xong, vào **VM Settings** → **CPU**:
   - Tick **Expose hardware assisted virtualization to the guest OS** (Enable nested VT-x)
5. Chỉnh RAM và disk theo nhu cầu
6. Power on VM

<img src="/assets/img/2026-05-04-pnetlab-lab/01.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/02.png"/>



#### 3.2. Cấu hình ban đầu

Khi VM khởi động lần đầu, console hiển thị địa chỉ IP (DHCP). Đăng nhập vào VM để cấu hình:

```
Username: root
Password: pnet
```

<img src="/assets/img/2026-05-04-pnetlab-lab/03.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/04.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/05.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/06.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/07.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/08.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/09.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/10.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/11.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/12.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/13.png"/>

---

### 4. Truy cập Web UI

Mở trình duyệt, truy cập địa chỉ IP của PNetLab:

```
http://10.10.200.31
```

Đăng nhập với tài khoản mặc định:
- **Username:** `admin`
- **Password:** `pnet`

<img src="/assets/img/2026-05-04-pnetlab-lab/14.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/15.png"/>

> **Lưu ý bảo mật:** Đổi mật khẩu `admin` ngay sau lần đăng nhập đầu tiên. Vào **Users** → chọn `admin` → **Change Password**.

<img src="/assets/img/2026-05-04-pnetlab-lab/16.png"/>

Sau khi đăng nhập, giao diện **Workspace** hiển thị với thanh menu trên cùng:

- **Main** — Trang chủ Workspace, quản lý lab file
- **Running Labs** — Xem các lab đang chạy hiện tại
- **Accounts** — Quản lý tài khoản người dùng
- **System** — Cấu hình hệ thống PNetLab
- **Download Labs** — Tải lab mẫu từ cộng đồng
- **Devices** — Quản lý node image đã upload

Giao diện chính chia 2 cột:
- **Trái:** danh sách lab file (Search Labs, các nút tạo/xoá/import/export lab)
- **Phải:** Lab Preview — xem trước topology của lab được chọn

<img src="/assets/img/2026-05-04-pnetlab-lab/17.png"/>


---

### 5. Upload Node Images

PNetLab cần image của thiết bị mạng để chạy. Có 2 cách:

Cách 1 - **PNetLab Store** — Download trực tiếp từ Web UI (một số free, một số cần tài khoản)
   
<img src="/assets/img/2026-05-04-pnetlab-lab/18a.png"/>
   
Cách 2 - **Upload thủ công** — Upload qua SCP/SFTP 

#### 5.1. Cisco vIOS L3 (Router)

**Cisco VIRL vIOS** là image router Cisco dùng cho lab routing (OSPF, BGP, EIGRP, v.v.). File image: `vios-15.5.3M.rar` — Pass giải nén: `Cnttshop`.

```bash
# Giải nén và đặt đúng tên file
mkdir -p /opt/unetlab/addons/qemu/vios-15.5.3M/

# Upload image (từ máy local dùng WinSCP/SCP)
scp vios-adventerprisek9-m15.5.3M.qcow2 \
    root@10.10.200.31:/opt/unetlab/addons/qemu/vios-15.5.3M/virtioa.qcow2

# Fix permissions
chown -R www-data:www-data /opt/unetlab/addons/qemu/
```

Quy tắc đặt tên thư mục image: `<template-name>-<version>/virtioa.qcow2`

#### 5.2. Cisco vIOS L2 (Switch)

**Cisco vIOS L2** là image switch layer 2/3 hỗ trợ VLANs, STP, EtherChannel. File image: `viosl2-15.2.4.55e.rar` — Pass giải nén: `Cnttshop2022`.

```bash
mkdir -p /opt/unetlab/addons/qemu/viosl2-15.2.4.55E/

scp vios_l2-adventerprisek9-m.SSA.high_iron_20180510.qcow2 \
    root@10.10.200.31:/opt/unetlab/addons/qemu/viosl2-15.2.4.55E/virtioa.qcow2

chown -R www-data:www-data /opt/unetlab/addons/qemu/
```

#### 5.3. Cisco CSR1000v (IOS XE)

**Cisco CSR1000v** là router ảo chạy IOS XE, phù hợp lab SD-WAN, DMVPN, BGP nâng cao. File image: `csr1000vng-ucmk9.16.12.3.rar` — Pass giải nén: `Cnttshop`.

```bash
mkdir -p /opt/unetlab/addons/qemu/csr1000vng-ucmk9.16.12.3/

scp csr1000v-universalk9.16.12.3.qcow2 \
    root@10.10.200.31:/opt/unetlab/addons/qemu/csr1000vng-ucmk9.16.12.3/virtioa.qcow2

chown -R www-data:www-data /opt/unetlab/addons/qemu/
```

#### 5.4. FortiGate

**FortiGate** VM là firewall NGFW của Fortinet, phù hợp lab security policy, IPS, SSL inspection, VPN. File image: `fortinet-fortigate7.2.0.rar` — Pass giải nén: `Cnttshop2022`.

- Tài khoản mặc định: `admin` / không có password (nhấn Enter)

```bash
mkdir -p /opt/unetlab/addons/qemu/fortinet-fortigate7.2.0/

scp fortios.qcow2 \
    root@10.10.200.31:/opt/unetlab/addons/qemu/fortinet-fortigate7.2.0/virtioa.qcow2

chown -R www-data:www-data /opt/unetlab/addons/qemu/
```


> **Troubleshoot — Node đỏ / không start được:**
> PNetLab yêu cầu file disk đầu tiên của **mọi** node QEMU phải tên **`virtioa.qcow2`**. Nếu upload file tên khác (ví dụ `fortios.qcow2`) thì PNetLab copy sai tên vào thư mục tmp → QEMU crash ngay.


<img src="/assets/img/2026-05-04-pnetlab-lab/18.png"/>

---

### 6. Tạo Lab Topology

Bài lab mẫu xây dựng topo đơn giản gồm 1 Linux VM — 1 Switch — 1 Router — 1 Firewall kết nối ra Internet qua Cloud0.

#### 6.1. Tạo Lab mới

1. Từ màn hình **Main** → click **+** (Add new lab) hoặc **New Lab**
2. Điền thông tin:
   - **Name:** `basic-internet-lab`
   - **Description:** VM - Switch - Router - Firewall - Internet
3. Click **Save** — canvas trống hiện ra

<img src="/assets/img/2026-05-04-pnetlab-lab/19.png"/>

#### 6.2. Thêm Node vào Lab

**Chuột phải** trên canvas → **Add new object** → **Node**, thêm lần lượt 4 node:

| Node | Template | Image | vCPU | RAM |
|---|---|---|---|---|
| `Ubuntu 20.04` | Linux | ubuntu 20.04 | 1 | 256 MB |
| `Switch` | Cisco vIOS L2 | `viosl2-15.2.4.55E` | 1 | 512 MB |
| `Router` | Cisco vIOS | `vios-15.5.3M` | 1 | 512 MB |
| `FortiGate` | fortinet | `fortinet-fortigate7.2.0` | **1** | **1024 MB** |


Sau khi thêm xong 4 node, canvas hiển thị đầy đủ topology. Click đúp vào node để mở **HTML5 Console** — terminal hiện ra trực tiếp trong trình duyệt.

<img src="/assets/img/2026-05-04-pnetlab-lab/20.png"/>


#### 6.3. Thêm Cloud0 (Internet)

1. **Chuột phải** trên canvas → **Add new object** → **Network**
2. Dialog **Edit network** hiện ra, điền thông tin:

   | Trường | Giá trị |
   |---|---|
   | **Name/Prefix** | `Internet` |
   | **Type** | `Management(Cloud0)` |
   | **Icon** | `cloud.png` |

3. Click **Save**

Cloud0 (đặt tên `Internet`) đại diện cho bridge `pnet0` của PNetLab — ánh xạ NIC vật lý `eth0` (ESXi: **Lab-vlan200**). Lưu lượng từ lab node qua Cloud0 đi ra mạng `10.10.200.0/24`, sau đó NAT ra Internet .


<img src="/assets/img/2026-05-04-pnetlab-lab/21.png"/>

#### 6.4. Kết nối các Node

Kéo link giữa các interface theo sơ đồ:

| Từ | Interface | Đến | Interface |
|---|---|---|---|
| Ubuntu 20.04 | eth1 | Switch | Gi0/0 |
| Switch | Gi0/1 | Router | Gi0/1 |
| Router | Gi0/0 | FortiGate | port2 |
| FortiGate | port1 | Internet (Cloud0) | — |

<img src="/assets/img/2026-05-04-pnetlab-lab/22.png"/>

#### 6.5. Khởi động và cấu hình

Start tất cả node (**chuột phải** → **Start**), sau đó vào console từng thiết bị:

**FortiGate** (cấu hình WAN static IP + NAT ra Internet):

```
FortiGate login: admin
Password: (Enter)

# Đặt hostname
config system global
    set hostname FortiGate-AnhLX-Lab
end

# Cấu hình port1 (WAN) — kết nối với Internet (Cloud0)
config system interface
    edit port1
        set mode static
        set ip 10.10.200.40 255.255.255.0
        set allowaccess ping http https
    next
end

# Cấu hình port2 (LAN) — kết nối với Router Gi0/0
config system interface
    edit port2
        set mode static
        set ip 192.168.30.2 255.255.255.0
        set allowaccess ping https ssh
    next
end

# Bỏ trusted-host giới hạn IP admin (mặc định chặn truy cập từ IP lạ)
config system admin
    edit admin
        set trusthost1 0.0.0.0 0.0.0.0
    next
end

# Cấu hình default route qua pfSense
config router static
    edit 1
        set gateway 10.10.200.1
        set device port1
    next
end

# Thêm route về mạng LAN Ubuntu (192.168.20.0/24) qua Router
# Nếu thiếu route này: Ubuntu ping FortiGate → FortiGate reply ra port1 (WAN)
# theo default route → Ubuntu không nhận được reply → ping fail
config router static
    edit 2
        set dst 192.168.20.0 255.255.255.0
        set gateway 192.168.30.1
        set device port2
    next
end

# Bật NAT (firewall policy LAN → WAN)
config firewall policy
    edit 1
        set name "LAN-to-WAN"
        set srcintf "port2"
        set dstintf "port1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set nat enable
        set schedule "always"
        set service "ALL"
    next
end

# Kiểm tra IP WAN và routing
get system interface physical
get router info routing-table all

# Verify admin trusted-host và allowaccess
show system admin
show system interface port1

# Sau đó truy cập Web GUI:
#   http://10.10.200.40   → tự redirect sang HTTPS
#   https://10.10.200.40  → chấp nhận SSL warning → đăng nhập admin

```

<img src="/assets/img/2026-05-04-pnetlab-lab/23.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/24.png"/>

<img src="/assets/img/2026-05-04-pnetlab-lab/25.png"/>


**Router — vIOS L3** (IP + default route qua FortiGate):

```
Router# conf t
Router(config)# interface GigabitEthernet0/0
Router(config-if)#  ip address 192.168.30.1 255.255.255.0
Router(config-if)#  no shutdown
Router(config)# interface GigabitEthernet0/1
Router(config-if)#  ip address 192.168.20.1 255.255.255.0
Router(config-if)#  no shutdown
Router(config)# ip route 0.0.0.0 0.0.0.0 192.168.30.2
```

<img src="/assets/img/2026-05-04-pnetlab-lab/26.png"/>

**Switch — vIOS L2** (VLAN + access port):

```
Switch# conf t
Switch(config)# vlan 10
Switch(config-vlan)# name CLIENT
Switch(config)# interface GigabitEthernet0/0
Switch(config-if)#  switchport mode access
Switch(config-if)#  switchport access vlan 10
Switch(config-if)#  no shutdown
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)#  switchport mode access
Switch(config-if)#  switchport access vlan 10
Switch(config-if)#  no shutdown
Switch(config)# end
Switch# wr
```

<img src="/assets/img/2026-05-04-pnetlab-lab/27.png"/>

**ubuntu 20.04** (IP tĩnh + default gateway + DNS):

```bash
ip addr add 192.168.20.10/24 dev eth1
ip link set eth1 up
ip route add default via 192.168.20.1
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```


**Kiểm tra ra được Internet:**

```
curl -I http://google.com               # test HTTP port 80
curl -Is https://google.com | head -1   # test HTTPS port 443
apt update                              # test DNS (53) + HTTP/HTTPS thực tế
ntpdate vn.pool.ntp.org                 # test NTP port 123 (UDP)
```

<img src="/assets/img/2026-05-04-pnetlab-lab/28.png"/>



---


### 7. Lời kết

PNetLab là công cụ mạnh mẽ và linh hoạt cho việc học và lab mạng. Một số điểm nổi bật:

- **Miễn phí** cho cá nhân với đầy đủ tính năng cơ bản
- **Web-based** — Không cần cài đặt client, dùng được từ bất kỳ máy nào trong mạng
- **Multi-vendor** — Hỗ trợ hầu hết vendor phổ biến (Cisco, Juniper, MikroTik, Palo Alto, v.v.)
- **Docker node** — Thêm Linux container để lab routing protocol, BGP, v.v. mà không cần image Cisco

Sau khi nắm được cơ bản, bạn có thể mở rộng lab với các chủ đề nâng cao như **BGP full-mesh**, **MPLS/VPN**, **SD-WAN**, hoặc tích hợp với **Ansible/Python** để thực hành network automation.

---

*Tham khảo:*
- [PNetLab Official](https://pnetlab.com)
- [PNetLab Documentation](https://pnetlab.com/pages/documentation)
- [Cisco Image Compatibility](https://pnetlab.com/pages/store)
