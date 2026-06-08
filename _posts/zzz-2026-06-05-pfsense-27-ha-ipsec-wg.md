---
title: "pfSense HA — IPsec Site-to-Site FortiGate + WireGuard Client VPN"
categories:
- Network
tags:
- pfsense
- ipsec
- wireguard
- vpn
- ha
- fortigate
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### pfSense HA (CARP) — IPsec IKEv2 với FortiGate VM, WireGuard Client-to-Site
---

![](/assets/img/2026-06-05-pfsense-27-ha-ipsec-wg/topo.png)

### Mục lục

- [Mục lục](#mục-lục)
- [1. Giới thiệu](#1-giới-thiệu)
- [2. Topology \& Quy hoạch IP](#2-topology--quy-hoạch-ip)
- [3. Cài đặt pfSense 2.7](#3-cài-đặt-pfsense-27)
- [4. Update pfSense lên bản mới nhất](#4-update-pfsense-lên-bản-mới-nhất)
- [5. Cấu hình cơ bản pfSense](#5-cấu-hình-cơ-bản-pfsense)
- [6. Firewall Rules](#6-firewall-rules)
  - [6.1. WAN Rules](#61-wan-rules)
  - [6.2. LAN Rules](#62-lan-rules)
  - [6.3. SYNC Rules](#63-sync-rules)
- [7. Cấu hình HA — CARP, pfsync, XMLRPC Sync](#7-cấu-hình-ha--carp-pfsync-xmlrpc-sync)
  - [7.1. Tạo CARP VIP](#71-tạo-carp-vip)
  - [7.2. Cấu hình pfsync](#72-cấu-hình-pfsync)
  - [7.3. XMLRPC Config Sync *(chỉ trên pfsense01)*](#73-xmlrpc-config-sync-chỉ-trên-pfsense01)
  - [7.4. Cấu hình pfsense02](#74-cấu-hình-pfsense02)
- [8. Deploy FortiGate VM 8.0.0 trên ESXi](#8-deploy-fortigate-vm-800-trên-esxi)
- [9. Cấu hình IPsec Site-to-Site: pfSense ↔ FortiGate](#9-cấu-hình-ipsec-site-to-site-pfsense--fortigate)
  - [9.0. Giới hạn mã hóa của FortiGate Permanent Trial](#90-giới-hạn-mã-hóa-của-fortigate-permanent-trial)
  - [9.1. Cấu hình Phase 1 \& Phase 2 trên pfSense](#91-cấu-hình-phase-1--phase-2-trên-pfsense)
  - [9.2. Cấu hình IPsec trên FortiGate](#92-cấu-hình-ipsec-trên-fortigate)
  - [9.3. Bring up tunnel](#93-bring-up-tunnel)
- [10. WireGuard Client-to-Site](#10-wireguard-client-to-site)
  - [10.1. Cài WireGuard package](#101-cài-wireguard-package)
  - [10.2. Tạo WireGuard Tunnel](#102-tạo-wireguard-tunnel)
  - [10.3. Tạo Peer anhlx](#103-tạo-peer-anhlx)
  - [10.4. Assign Interface \& Firewall Rules](#104-assign-interface--firewall-rules)
  - [10.5. Cấu hình Windows Client](#105-cấu-hình-windows-client)
- [11. Chuẩn bị VM test](#11-chuẩn-bị-vm-test)
- [12. Kiểm tra \& Test Failover](#12-kiểm-tra--test-failover)
  - [12.1. Test SSH vm01 → vm02 qua IPsec tunnel](#121-test-ssh-vm01--vm02-qua-ipsec-tunnel)
  - [12.2. Test SSH WireGuard user → vm02 qua IPsec tunnel](#122-test-ssh-wireguard-user--vm02-qua-ipsec-tunnel)
  - [12.3. Test WireGuard connectivity](#123-test-wireguard-connectivity)
  - [12.4. Test CARP Failover](#124-test-carp-failover)
- [13. Lời kết](#13-lời-kết)

---

### 1. Giới thiệu

Khi cung cấp dịch vụ quản trị hạ tầng hoặc kết nối vào hệ thống của khách hàng, yêu cầu phổ biến nhất là VPN IPsec tương thích với các firewall doanh nghiệp — FortiGate, Cisco ASA, Palo Alto đều mặc định chạy IKEv2/IPsec chuẩn RFC. pfSense đáp ứng tốt vai trò này: hỗ trợ đầy đủ IKEv2, không tốn license, và có thể chạy HA với CARP để đảm bảo tunnel VPN không bị ngắt khi một node lỗi.

Lab này triển khai một cặp pfSense 2.7 chạy High Availability, tunnel IPsec IKEv2 kết nối sang FortiGate VM 8.0.0 đóng vai site khách hàng, và WireGuard client-to-site để kỹ thuật viên remote vào được mạng nội bộ.

Các thành phần:
- **pfSense CE** — firewall/gateway phía mình, chạy 1 cặp để HA
- **CARP** — Virtual IP failover cho WAN và LAN, IPsec bind vào CARP VIP để survive khi một node down
- **pfsync** — đồng bộ connection state table giữa 2 pfSense node theo real-time
- **FortiGate VM 8.0.0** — đóng vai enterprise firewall phía site khách hàng
- **IPsec IKEv2** — site-to-site tunnel tương thích chuẩn doanh nghiệp
- **WireGuard** — client-to-site VPN cho remote user kết nối từ Windows

### 2. Topology & Quy hoạch IP

Mình dùng VLAN 200 làm đường "internet" chung giữa 2 site — đủ để test IPsec IKEv2 tunnel trong lab. VLAN 201 là mạng nội bộ phía pfSense (Site A), VLAN 203 là mạng nội bộ phía FortiGate (Site B, customer site). VLAN 202 dành riêng cho kết nối HA sync giữa 2 pfSense node, không có uplink ra ngoài.

| Host | Interface | IP | VLAN | Ghi chú |
|------|-----------|-----|------|---------|
| pfsense01 | WAN (em0) | 10.10.200.11/24 | 200 | |
| pfsense01 | LAN (em1) | 10.10.201.1/24 | 201 | |
| pfsense01 | SYNC (em2) | 10.10.202.1/24 | 202 | HA sync, isolated |
| pfsense02 | WAN (em0) | 10.10.200.12/24 | 200 | |
| pfsense02 | LAN (em1) | 10.10.201.2/24 | 201 | |
| pfsense02 | SYNC (em2) | 10.10.202.2/24 | 202 | HA sync, isolated |
| CARP VIP | WAN | 10.10.200.50/24 | 200 | IPsec local endpoint |
| CARP VIP | LAN | 10.10.201.50/24 | 201 | Default gateway LAN |
| FortiGate VM | port1 (WAN) | 10.10.200.21/24 | 200 | IPsec remote endpoint |
| FortiGate VM | port2 (LAN) | 10.10.203.1/24 | 203 | Customer site LAN |
| vm01 | ens160 | 10.10.201.11/24 | 201 | Ubuntu test, pfSense LAN |
| vm02 | ens160 | 10.10.203.11/24 | 203 | Ubuntu test, FortiGate LAN |
| WireGuard server | wg0 | 192.168.10.1/24 | — | pfSense WG interface |
| WireGuard peer anhlx | — | 192.168.10.2/32 | — | Windows laptop |

VM specs:

| VM | vCPU | RAM | Disk | NIC |
|----|------|-----|------|-----|
| pfsense01 | 2 | 4 GB | 20 GB | em0 (VLAN 200), em1 (VLAN 201), em2 (VLAN 202) |
| pfsense02 | 2 | 4 GB | 20 GB | em0 (VLAN 200), em1 (VLAN 201), em2 (VLAN 202) |
| fortigate01 | 2 | 4 GB | — | port1 (VLAN 200), port2 (VLAN 203) |
| vm01 | 2 | 2 GB | 20 GB | ens160 (VLAN 201) |
| vm02 | 2 | 2 GB | 20 GB | ens160 (VLAN 203) |

### 3. Cài đặt pfSense 2.7

Mình cài pfSense trên cả 2 node theo cùng quy trình. Download ISO `pfSense-CE-2.7.x-RELEASE-amd64.iso` từ trang chủ `pfsense.org/download`.

Tạo VM trên ESXi:
- Guest OS: **FreeBSD 14 (64-bit)**
- vCPU: 2, RAM: 4 GB, Disk: 20 GB (thin provision)
- Add 3 network adapters: VLAN 200 (WAN), VLAN 201 (LAN), VLAN 202 (SYNC)
- Mount ISO, Power On

Trình cài đặt pfSense boot lên, mình chọn theo thứ tự:
- **Accept** → **Install pfSense** → Keymap: **Continue with default keymap**
- Partitioning: **Auto (UFS)** → chọn disk `da0` → **Finish** → **Commit**
- Cài đặt xong, **Reboot**

Sau reboot, console wizard hỏi assign interfaces:

```
Should VLANs be set up now? n
Enter the WAN interface name:      em0
Enter the LAN interface name:      em1
Enter the Optional 1 interface:    em2
Do you want to proceed?            y
```

Tiếp theo đặt IP qua console menu (option 2 — Set interface(s) IP address):

**pfsense01:**
- WAN: `10.10.200.11/24`, upstream gateway `10.10.200.1`
- LAN: `10.10.201.1/24`

**pfsense02:**
- WAN: `10.10.200.12/24`, upstream gateway `10.10.200.1`
- LAN: `10.10.201.2/24`

Truy cập WebGUI tại `https://10.10.201.1` (pfsense01). Setup wizard: đặt hostname `pfsense01`, domain, DNS `8.8.8.8`, timezone `Asia/Ho_Chi_Minh`, NTP `vn.pool.ntp.org`, password admin.

### 4. Update pfSense lên bản mới nhất

Sau khi cài xong, mình update ngay trước khi bắt đầu cấu hình.

**System > Update > Update Settings:**
- Branch: **Latest stable version**
- Save

**System > Update > System Update:**
- Nếu có bản cập nhật, click **Confirm**
- pfSense download patch, apply và reboot tự động

Sau reboot, vào lại **System > Update** — kiểm tra thông báo *Your system is up to date*.

> Lặp lại bước update cho **pfsense02** trước khi cấu hình HA.

### 5. Cấu hình cơ bản pfSense

Mình thực hiện trên **pfsense01** — sau khi cấu hình XMLRPC ở bước tiếp theo, config sẽ tự sync sang pfsense02.

**Interface SYNC (em2):**

**Interfaces > OPT1** (em2):
- Enable: **checked**
- Description: `SYNC`
- IPv4 Configuration Type: Static IPv4
- IPv4 Address: `10.10.202.1/24`
- Save, Apply Changes

*(pfsense02 dùng `10.10.202.2/24` — đặt thủ công trên node đó)*

**DNS Resolver:**

**Services > DNS Resolver:**
- Enable: checked
- Listen Port: 53
- Network Interfaces: LAN, Localhost
- Outgoing Network Interfaces: WAN
- Enable DNSSEC Support: checked
- Save, Apply

**NTP:**

**Services > NTP:**
- Time Servers: `vn.pool.ntp.org`
- Interface: LAN, SYNC
- Save

**SSH:**

**System > Advanced > Admin Access:**
- Secure Shell Server: **Enable Secure Shell**
- Save

### 6. Firewall Rules

Mình cấu hình firewall rules trên **pfsense01** — XMLRPC sẽ sync toàn bộ sang pfsense02 sau khi HA được thiết lập. pfSense mặc định block tất cả inbound trên WAN và allow all outbound trên LAN, đủ để chạy nhưng chưa phù hợp môi trường doanh nghiệp.

#### 6.1. WAN Rules

Trước tiên bật 2 tùy chọn chặn sẵn tại **Interfaces > WAN**:
- **Block private networks and loopback addresses**: checked
- **Block bogon networks**: checked

pfSense mặc định block toàn bộ inbound WAN — mình chỉ mở đúng những port cần thiết. **Firewall > Rules > WAN**, Add:

| # | Action | Protocol | Destination | Port | Mô tả |
|---|--------|----------|-------------|------|--------|
| 1 | Pass | UDP | This Firewall | 500, 4500 | IPsec IKE / NAT-T |
| 2 | Pass | UDP | This Firewall | 51820 | WireGuard |

Tất cả inbound còn lại bị block ngầm định (implicit deny cuối ruleset).

#### 6.2. LAN Rules

pfSense tạo sẵn rule *Default allow LAN to any* — mình disable rule đó và thay bằng whitelist explicit để kiểm soát traffic ra ngoài.

**Firewall > Rules > LAN** — disable rule default, sau đó Add theo thứ tự từ trên xuống:

| # | Action | Protocol | Source | Destination | Port | Mô tả |
|---|--------|----------|--------|-------------|------|--------|
| 1 | Pass | TCP | LAN net | any | 80, 443 | HTTP/HTTPS |
| 2 | Pass | TCP/UDP | LAN net | 8.8.8.8 | 53 | DNS (chỉ ra 8.8.8.8) |
| 3 | Pass | UDP | LAN net | any | 123 | NTP |
| 4 | Pass | ICMP | LAN net | any | — | Ping cho monitoring |
| 5 | Block | TCP | LAN net | any | 25 | Chặn SMTP thẳng ra ngoài |
| 6 | Block | any | LAN net | any | — | Deny all còn lại (bật Log) |

> Rule 2 chỉ cho phép DNS đến `8.8.8.8` — ngăn client dùng DNS tùy ý để bypass content filtering. Nếu dùng DNS Resolver nội bộ, đổi destination thành `This Firewall` port 53 và block DNS ra ngoài hoàn toàn.

#### 6.3. SYNC Rules

Interface SYNC phải lock down chặt — chỉ cho phép pfsync replication và XMLRPC config sync giữa 2 node, không cho traffic nào khác đi qua.

**Firewall > Rules > SYNC** — Add:

| # | Action | Protocol | Source | Destination | Port | Mô tả |
|---|--------|----------|--------|-------------|------|--------|
| 1 | Pass | pfsync | 10.10.202.2 | This Firewall | — | State table sync |
| 2 | Pass | TCP | 10.10.202.2 | This Firewall | 443 | XMLRPC config sync |
| 3 | Block | any | any | any | — | Deny all còn lại (bật Log) |

*(pfsense02 cấu hình tương tự, đổi source thành `10.10.202.1`)*

### 7. Cấu hình HA — CARP, pfsync, XMLRPC Sync

HA pfSense dựa trên 3 cơ chế: **CARP** (VIP failover), **pfsync** (state table sync real-time), và **XMLRPC** (config sync khi thay đổi rule). Thực hiện trên **pfsense01** trước.

#### 7.1. Tạo CARP VIP

**Interfaces > Virtual IPs > Add:**

VIP WAN:

| Tham số | Giá trị |
|---------|---------|
| Type | CARP |
| Interface | WAN |
| Address(es) | `10.10.200.50/24` |
| Virtual IP Password | `<vip_pass>` |
| VHID Group | `1` |
| Advertising Frequency | Base `1`, Skew `0` *(master)* |
| Description | CARP-WAN |

VIP LAN:

| Tham số | Giá trị |
|---------|---------|
| Type | CARP |
| Interface | LAN |
| Address(es) | `10.10.201.50/24` |
| Virtual IP Password | `<vip_pass>` |
| VHID Group | `2` |
| Advertising Frequency | Base `1`, Skew `0` |
| Description | CARP-LAN |

Save, **Apply Changes**.

#### 7.2. Cấu hình pfsync

**System > High Availability Sync:**

| Tham số | Giá trị |
|---------|---------|
| Synchronize States | checked |
| Synchronize Interface | SYNC |
| pfsync Synchronize Peer IP | `10.10.202.2` *(SYNC IP của pfsense02)* |

#### 7.3. XMLRPC Config Sync *(chỉ trên pfsense01)*

Cùng trang **System > High Availability Sync**, phần *Configuration Synchronization Settings:*

| Tham số | Giá trị |
|---------|---------|
| Synchronize Config to IP | `10.10.202.2` |
| Remote System Username | `admin` |
| Remote System Password | `<pfsense02_password>` |
| Sync items | Tích chọn tất cả: User Manager, Certificates, Firewall Rules, NAT, Aliases, Interfaces, DHCP, DNS Resolver, IPsec, WireGuard, Static Routes |

Save → click **Sync** để đẩy config lần đầu sang pfsense02.

#### 7.4. Cấu hình pfsense02

Trên **pfsense02** chỉ cần cấu hình pfsync (XMLRPC đã đẩy toàn bộ config còn lại):

**System > High Availability Sync:**
- Synchronize States: checked
- Synchronize Interface: SYNC
- pfsync Synchronize Peer IP: `10.10.202.1`
- Save

Sau khi XMLRPC sync, các CARP VIP đã xuất hiện trên pfsense02. Mình edit cả 2 VIP CARP (VHID 1 và 2) trên pfsense02, chỉnh **Advertising Frequency Skew** từ `0` → `100` để pfsense01 luôn giữ vai Master.

**Kiểm tra:** Vào **Status > CARP (failover)** trên cả 2 node:
- pfsense01: `MASTER` cho VHID 1 và 2
- pfsense02: `BACKUP` cho VHID 1 và 2

### 8. Deploy FortiGate VM 8.0.0 trên ESXi

FortiGate VM đóng vai enterprise firewall phía site khách hàng. Mình dùng file `FGT_VM64-v8.0.0.F-build0167-FORTINET.out.ovf.zip` (121.56 MB) — download từ `support.fortinet.com` → **FortiGate** → **VMware ESXi** → version **8.0.0** → chọn mục *"New deployment of FortiGate for VMware"*.

**Import OVF lên ESXi:**

Trên vSphere Client, **Actions > Deploy OVF Template:**
- Upload file `.ovf.zip` vừa download
- Đặt tên VM: `fortigate01`
- Network mapping:
  - Network 1 → **VLAN 200** (WAN)
  - Network 2 → **VLAN 203** (tạo portgroup mới nếu chưa có)
- Finish, Power On VM

**Cấu hình ban đầu qua Console:**

FortiGate boot lần đầu, login `admin` / *(blank)* — hệ thống yêu cầu đổi password ngay:

```
FortiGate-VM64 login: admin
Password:
New Password: <fortigate_password>
Confirm Password: <fortigate_password>
```

Cấu hình interface và routing:

```
config system interface
    edit port1
        set ip 10.10.200.21 255.255.255.0
        set allowaccess http https ssh ping
        set description "WAN"
    next
    edit port2
        set ip 10.10.203.1 255.255.255.0
        set allowaccess ping
        set description "LAN-CustomerSite"
    next
end

config router static
    edit 1
        set device port1
        set gateway 10.10.200.1
    next
end
```

Truy cập WebGUI tại `https://10.10.200.21`. FortiGate VM không có license sẽ chạy ở chế độ **Permanent Trial** — không giới hạn thời gian nhưng bị khóa các tính năng mã hóa mạnh (xem chi tiết ở phần tiếp theo).

**System > Settings:** đặt timezone `(GMT+7:00) Bangkok, Hanoi, Jakarta`, NTP server `vn.pool.ntp.org`.

### 9. Cấu hình IPsec Site-to-Site: pfSense ↔ FortiGate

#### 9.0. Giới hạn mã hóa của FortiGate Permanent Trial

Trước khi cấu hình, cần nắm rõ một hạn chế quan trọng của bản FortiGate VM không có license. Fortinet khóa toàn bộ các thuật toán mã hóa mạnh trên bản Permanent Trial để ngăn dùng miễn phí cho môi trường thật:

| | Thuật toán |
|---|---|
| **Bị khóa** | AES-128, AES-256, SHA-256, SHA-512, DH Group 14/19/20+ |
| **Được phép** | 3DES, DES, MD5, SHA-1, DH Group 2, DH Group 5 |

pfSense mặc định ưu tiên AES/SHA-256 — nếu giữ nguyên config mặc định, 2 bên sẽ không bao giờ negotiate được Phase 1 và báo lỗi *Crypto Exchange*. Cách xử lý duy nhất khi dùng bản Permanent Trial là **chủ động downgrade** crypto trên cả 2 đầu xuống 3DES + SHA-1 + DH Group 2.

Ngoài ra, bản Permanent Trial giới hạn **3 Interfaces, 3 Policies, 3 Routes**. Khi tạo IPsec tunnel kiểu route-based, FortiGate tự sinh ra 1 interface ảo (`TO-PFSENSE`) — cộng với port1 và port2, vậy là hết slot interface. Với policies, mình cần 2 rule cho tunnel (LAN→IPsec và IPsec→LAN), còn dư 1 rule cho LAN ra internet.

> Nếu muốn test AES-256/IKEv2 chuẩn doanh nghiệp, cần xin **Evaluation License** (full feature 30–60 ngày) từ đối tác Fortinet hoặc qua FortiCloud trial. Config AES-256 trong trường hợp đó giống hệt section này nhưng thay proposal thành `aes256-sha256` và `dhgrp 14`.

#### 9.1. Cấu hình Phase 1 & Phase 2 trên pfSense

**VPN > IPsec > Tunnels > Add P1:**

| Tham số | Giá trị |
|---------|---------|
| Key Exchange version | **IKEv1** |
| Internet Protocol | IPv4 |
| Interface | WAN |
| Remote Gateway | `10.10.200.21` |
| Description | `IPsec-to-FortiGate` |
| Authentication Method | Mutual PSK |
| My Identifier | My IP address |
| Peer Identifier | Peer IP address |
| Pre-Shared Key | `<ipsec_psk>` |
| Encryption Algorithm | **3DES** |
| Hash Algorithm | **SHA1** |
| DH Group | **2 (1024 bit)** |
| Lifetime (Seconds) | 28800 |

Save → **Show Phase 2 Entries**, mình thêm 2 P2 — một cho LAN-to-LAN, một cho WireGuard subnet sang FortiGate:

**P2 #1 — LAN ↔ FortiGate LAN:**

| Tham số | Giá trị |
|---------|---------|
| Mode | Tunnel IPv4 |
| Local Network | `10.10.201.0/24` |
| Remote Network | `10.10.203.0/24` |
| Protocol | ESP |
| Encryption Algorithms | **3DES** |
| Hash Algorithms | **SHA1** |
| PFS Key Group | **2 (1024 bit)** |
| Lifetime (Seconds) | 3600 |

**P2 #2 — WireGuard subnet ↔ FortiGate LAN:**

| Tham số | Giá trị |
|---------|---------|
| Mode | Tunnel IPv4 |
| Local Network | `192.168.10.0/24` |
| Remote Network | `10.10.203.0/24` |
| Protocol | ESP |
| Encryption Algorithms | **3DES** |
| Hash Algorithms | **SHA1** |
| PFS Key Group | **2 (1024 bit)** |
| Lifetime (Seconds) | 3600 |

Save, **Apply Changes**.

**Firewall > Rules > IPsec** — Add 2 rule cho traffic từ FortiGate vào:

| # | Action | Source | Destination | Mô tả |
|---|--------|--------|-------------|--------|
| 1 | Pass | `10.10.203.0/24` | `10.10.201.0/24` | FortiGate site → pfSense LAN |
| 2 | Pass | `10.10.203.0/24` | `192.168.10.0/24` | FortiGate site → WireGuard clients |

#### 9.2. Cấu hình IPsec trên FortiGate

Mình dùng CLI để config Phase 1, Phase 2, route và firewall policy. Lưu ý `ike-version 1` và proposal `3des-sha1` để khớp với giới hạn Permanent Trial.

Phase 1 và 2 Phase 2 (LAN + WireGuard subnet), route và firewall policy:

```
config vpn ipsec phase1-interface
    edit "TO-PFSENSE"
        set interface "port1"
        set ike-version 1
        set peertype any
        set proposal 3des-sha1
        set dhgrp 2
        set remote-gw 10.10.200.50
        set psksecret <ipsec_psk>
    next
end

config vpn ipsec phase2-interface
    edit "TO-PFSENSE-LAN"
        set phase1name "TO-PFSENSE"
        set proposal 3des-sha1
        set dhgrp 2
        set src-subnet 10.10.203.0 255.255.255.0
        set dst-subnet 10.10.201.0 255.255.255.0
    next
    edit "TO-PFSENSE-WG"
        set phase1name "TO-PFSENSE"
        set proposal 3des-sha1
        set dhgrp 2
        set src-subnet 10.10.203.0 255.255.255.0
        set dst-subnet 192.168.10.0 255.255.255.0
    next
end

config router static
    edit 2
        set device "TO-PFSENSE"
        set dst 10.10.201.0 255.255.255.0
    next
    edit 3
        set device "TO-PFSENSE"
        set dst 192.168.10.0 255.255.255.0
    next
end

config firewall policy
    edit 1
        set name "LAN-to-IPsec"
        set srcintf "port2"
        set dstintf "TO-PFSENSE"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
    next
    edit 2
        set name "IPsec-to-LAN"
        set srcintf "TO-PFSENSE"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
    next
end
```

#### 9.3. Bring up tunnel

Trên pfSense, vào **Status > IPsec** → click **Connect P1 and P2**. Sau vài giây tunnel negotiate và chuyển sang **Established**.

Kiểm tra từ FortiGate CLI:

```
diagnose vpn ike gateway list name TO-PFSENSE
diagnose vpn tunnel list name TO-PFSENSE-LAN
diagnose vpn tunnel list name TO-PFSENSE-WG
```

Kết quả mong đợi: Phase 1 `established`, cả 2 Phase 2 `selectors: 1/1`.

### 10. WireGuard Client-to-Site

WireGuard phục vụ kỹ thuật viên remote — kết nối từ Windows laptop vào mạng nội bộ VLAN 201 và qua IPsec tunnel sang FortiGate site. Mình tạo peer `anhlx` với IP tunnel cố định `192.168.10.2`.

#### 10.1. Cài WireGuard package

**System > Package Manager > Available Packages**, tìm `wireguard`, click **Install**. Chờ hoàn tất.

#### 10.2. Tạo WireGuard Tunnel

**VPN > WireGuard > Tunnels > Add Tunnel:**

| Tham số | Giá trị |
|---------|---------|
| Description | `WG-ClientVPN` |
| Listen Port | `51820` |
| Interface Keys | Click **Generate** |
| Interface Addresses | `192.168.10.1/24` |

Save — ghi lại **Public Key** của pfSense (dùng trong config Windows client).

#### 10.3. Tạo Peer anhlx

**VPN > WireGuard > Peers > Add Peer:**

| Tham số | Giá trị |
|---------|---------|
| Tunnel | `tun_wg0` |
| Description | `anhlx` |
| Dynamic Endpoint | checked *(client IP thay đổi theo wifi/4G)* |
| Peer WireGuard Keys | Click **Generate** — pfSense tạo key pair cho peer |
| Allowed IPs | `192.168.10.2/32` |

Save — ghi lại **Private Key** của peer (đưa vào config Windows client bên dưới).

#### 10.4. Assign Interface & Firewall Rules

**Interfaces > Assignments** — dropdown *Available network ports*, chọn `tun_wg0`, click **Add**.

**Interfaces > WG** (interface vừa tạo):
- Enable: checked
- Description: `WG`
- Save, Apply

**Firewall > Rules > WAN** — Add rule:
- Action: Pass, Protocol: UDP
- Destination: This Firewall (self), Port: `51820`
- Description: `Allow WireGuard`

**Firewall > Rules > WG** — Add 2 rule:

| # | Action | Source | Destination | Mô tả |
|---|--------|--------|-------------|--------|
| 1 | Pass | WG net | LAN net | WG clients → pfSense LAN |
| 2 | Pass | WG net | `10.10.203.0/24` | WG clients → FortiGate site qua IPsec |

#### 10.5. Cấu hình Windows Client

Trên Windows laptop, cài **WireGuard for Windows** từ `wireguard.com/install`. Mở app, click **Add Tunnel > Add empty tunnel**, đặt tên `pfsense-vpn`.

Dán config vào (thay 2 key bằng giá trị thực từ pfSense):

```ini
[Interface]
PrivateKey = <PEER_PRIVATE_KEY_TỪ_PFSENSE>
Address    = 192.168.10.2/32
DNS        = 8.8.8.8

[Peer]
PublicKey           = <PFSENSE_WG_PUBLIC_KEY>
Endpoint            = 10.10.200.50:51820
AllowedIPs          = 10.10.201.0/24, 10.10.203.0/24, 192.168.10.0/24
PersistentKeepalive = 25
```

Click **Save** → **Activate**.

### 11. Chuẩn bị VM test

Mình deploy 2 Ubuntu 22.04 minimal: vm01 nằm trong VLAN 201 (pfSense LAN), vm02 nằm trong VLAN 203 (FortiGate LAN). Cả 2 dùng làm endpoint để test traffic qua IPsec tunnel.

**Trên vm01** (VLAN 201, gateway là CARP VIP):

```bash
sudo tee /etc/netplan/01-netcfg.yaml << 'EOF'
network:
  version: 2
  ethernets:
    ens160:
      addresses: [10.10.201.11/24]
      routes:
        - to: default
          via: 10.10.201.50
      nameservers:
        addresses: [8.8.8.8]
EOF
sudo netplan apply
```

**Trên vm02** (VLAN 203, gateway là FortiGate port2):

```bash
sudo tee /etc/netplan/01-netcfg.yaml << 'EOF'
network:
  version: 2
  ethernets:
    ens160:
      addresses: [10.10.203.11/24]
      routes:
        - to: default
          via: 10.10.203.1
      nameservers:
        addresses: [8.8.8.8]
EOF
sudo netplan apply
```

Kiểm tra SSH service đang chạy trên cả 2:

```bash
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

Tạo user test trên vm02 để dùng khi SSH vào từ bên ngoài:

```bash
sudo adduser <testuser>
```

### 12. Kiểm tra & Test Failover

#### 12.1. Test SSH vm01 → vm02 qua IPsec tunnel

Traffic path: `vm01 (10.10.201.11)` → pfSense LAN → IPsec P2 (10.10.201.0/24 ↔ 10.10.203.0/24) → FortiGate → `vm02 (10.10.203.11)`.

Từ vm01, ping và SSH sang vm02:

```bash
ping 10.10.203.11
ssh <testuser>@10.10.203.11
```

Kiểm tra tunnel có traffic không, từ pfSense **Status > IPsec** — cột *Bytes In/Out* tăng lên khi SSH active.

Từ FortiGate CLI xác nhận traffic đi qua tunnel:

```
diagnose vpn tunnel list name TO-PFSENSE-LAN
```

#### 12.2. Test SSH WireGuard user → vm02 qua IPsec tunnel

Traffic path: `WG client (192.168.10.2)` → pfSense WG interface → IPsec P2 (192.168.10.0/24 ↔ 10.10.203.0/24) → FortiGate → `vm02 (10.10.203.11)`.

Kết nối WireGuard từ Windows trước, sau đó mở terminal và SSH:

```
ping 10.10.203.11
ssh <testuser>@10.10.203.11
```

Trên pfSense **Status > IPsec**, P2 `192.168.10.0/24 ↔ 10.10.203.0/24` phải hiện trạng thái **Established** và có bytes transferred.

Kiểm tra từ FortiGate:

```
diagnose vpn tunnel list name TO-PFSENSE-WG
```

#### 12.3. Test WireGuard connectivity

Kết nối WireGuard từ Windows, kiểm tra ping vào pfSense LAN:

```
ping 192.168.10.1
ping 10.10.201.1
ping 10.10.201.11
```

Trên pfSense, **VPN > WireGuard > Status** — peer `anhlx` hiện **Latest Handshake** timestamp và bytes transferred.

#### 12.4. Test CARP Failover

Với IPsec và WireGuard đang active (SSH session từ vm01 → vm02 đang chạy), mình tắt **pfsense01** (shutdown VM).

Quan sát trên **pfsense02:**
- **Status > CARP**: pfsense02 chuyển lên **MASTER** cho cả VHID 1 và 2
- VIP `10.10.200.50` và `10.10.201.50` chuyển sang pfsense02

IPsec tunnel sẽ re-negotiate sau vài giây vì CARP VIP chuyển node. Kiểm tra từ FortiGate:

```
execute ping 10.10.201.50
diagnose vpn ike gateway list name TO-PFSENSE
```

Tunnel re-establish với cùng endpoint `10.10.200.50` — FortiGate không nhận ra có failover vì IP không đổi. Session SSH từ vm01 → vm02 có thể bị ngắt tạm thời trong vài giây rồi recover nhờ pfsync đã sync state trước đó.

WireGuard cũng tự reconnect khi pfsense02 bắt đầu lắng nghe UDP 51820 trên VIP.

Bật lại pfsense01 — sau vài giây nó giành lại vai **MASTER** do Advertising Skew `0` thấp hơn `100` của pfsense02.

### 13. Lời kết

Mình đã hoàn thiện một bộ hạ tầng VPN enterprise-ready gồm pfSense 2.7 HA với CARP, firewall rules theo mô hình whitelist, IPsec IKEv2 tương thích FortiGate, và WireGuard client-to-site cho remote user. Điểm quan trọng nhất là bind IPsec tunnel vào CARP VIP và cấu hình đủ Phase 2 selector cho cả LAN subnet lẫn WireGuard subnet — giúp remote user qua WireGuard vẫn reach được site FortiGate xuyên suốt qua một đường tunnel duy nhất. Từ lab này có thể mở rộng thêm multiple Phase 2 selectors khi cần route nhiều subnet hơn, hoặc thay WireGuard bằng OpenVPN kết hợp Windows NPS nếu muốn xác thực VPN client qua tài khoản Active Directory.
