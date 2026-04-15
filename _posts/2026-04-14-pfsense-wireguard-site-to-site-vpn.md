---
title: "pfSense WireGuard Site-to-Site VPN"
categories:
- Network
- Security

feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Lab VPN Site-to-Site giữa 2 pfSense với WireGuard — On-Premise to Cloud
---

### Mục lục

- [Mục lục](#mục-lục)
- [1. Giới thiệu](#1-giới-thiệu)
- [2. Topology \& Quy hoạch IP](#2-topology--quy-hoạch-ip)
- [3. Cài đặt WireGuard package trên pfSense](#3-cài-đặt-wireguard-package-trên-pfsense)
- [4. Cấu hình WireGuard trên FW01 — Site A (On-Premise, Initiator)](#4-cấu-hình-wireguard-trên-fw01--site-a-on-premise-initiator)
  - [4.1. Tạo Tunnel](#41-tạo-tunnel)
  - [4.2. Thêm Peer (FW02 — Cloud)](#42-thêm-peer-fw02--cloud)
- [5. Cấu hình WireGuard trên FW02 — Site B (Cloud, Listener)](#5-cấu-hình-wireguard-trên-fw02--site-b-cloud-listener)
  - [5.1. Tạo Tunnel](#51-tạo-tunnel)
  - [5.2. Thêm Peer (FW01 — On-Premise)](#52-thêm-peer-fw01--on-premise)
- [6. Assign Interface WireGuard](#6-assign-interface-wireguard)
- [7. Cấu hình Firewall Rules](#7-cấu-hình-firewall-rules)
  - [7.1. WAN Rules - Cho phép WireGuard](#71-wan-rules---cho-phép-wireguard)
  - [7.2. WireGuard Interface Rules](#72-wireguard-interface-rules)
  - [7.3. LAN Rules](#73-lan-rules)
- [8. Kiểm tra kết nối](#8-kiểm-tra-kết-nối)
  - [8.1. Kiểm tra trạng thái WireGuard](#81-kiểm-tra-trạng-thái-wireguard)
  - [8.2. Ping test từ pfSense](#82-ping-test-từ-pfsense)
  - [8.3. Ping test từ VM](#83-ping-test-từ-vm)
- [9. Lời kết](#9-lời-kết)

---

### 1. Giới thiệu

**WireGuard** là một giao thức VPN hiện đại, nhẹ, nhanh và bảo mật cao hơn so với IPsec hay OpenVPN truyền thống. WireGuard sử dụng mã hoá tiên tiến (ChaCha20, Curve25519, BLAKE2s) và có codebase nhỏ gọn (~4000 dòng code), giúp dễ audit và ít lỗi bảo mật.

Trong bài lab này, mình sẽ triển khai **VPN Site-to-Site** giữa 2 pfSense firewall sử dụng WireGuard theo mô hình thực tế:
- **Site A (On-Premise):** Không có IP public, nằm sau NAT/ISP router
- **Site B (Cloud):** Có IP public gắn trực tiếp vào WAN interface

Đây là kịch bản rất phổ biến khi doanh nghiệp cần kết nối văn phòng (on-prem) với hạ tầng cloud. WireGuard hỗ trợ tốt trường hợp này nhờ cơ chế **persistent keepalive** — FW01 (on-prem) chủ động khởi tạo kết nối đến FW02 (cloud) và duy trì tunnel qua NAT.

**Mục tiêu:**
- VLAN 200 (10.10.200.0/24) giả lập đường Internet giữa 2 site
- VLAN 201 (10.10.201.0/24) là mạng nội bộ Site A — **On-Premise**
- VLAN 202 (10.10.202.0/24) là mạng nội bộ Site B — **Cloud**
- FW01 (on-prem) **không có IP public**, kết nối outbound đến FW02
- FW02 (cloud) **có IP public** (`10.10.200.253` trong lab, tương đương Public IP trên cloud)
- Các VM trong VLAN 201 và 202 thông nhau qua WireGuard tunnel

**Thông tin lab:**

| Thành phần | Giá trị |
|-----------|--------|
| pfSense Version | 2.7.2-RELEASE (FreeBSD 14.0) |
| Hypervisor | VMware vSphere (ESXi) |

> **Lưu ý mô hình lab:** Trong lab, VLAN 200 giả lập Internet nên cả 2 FW đều có IP trên cùng subnet. Trong thực tế, FW01 sẽ có private IP từ ISP (NAT) và FW02 sẽ có Public IP thật trên cloud. Cấu hình WireGuard giữ nguyên — chỉ thay IP endpoint của FW02 thành Public IP thật.

---

### 2. Topology & Quy hoạch IP

```
  Site A - ON-PREMISE (VLAN 201)        Internet (VLAN 200)           Site B - CLOUD (VLAN 202)
  10.10.201.0/24                        10.10.200.0/24                10.10.202.0/24
  *** Không có IP Public ***                                         *** Có IP Public ***

 ┌──────────────┐      ┌─────────────────────────────────────┐      ┌──────────────┐
 │  VM-A        │      │                                     │      │  VM-B        │
 │ 10.10.201.x  │      │         "Simulated Internet"        │      │ 10.10.202.x  │
 └──────┬───────┘      │          VLAN 200                   │      └──────┬───────┘
        │              │                                     │             │
 ┌──────▼───────┐      │                                     │      ┌──────▼───────┐
 │   FW01       │      │       FW01 initiates ──────►        │      │   FW02       │
 │ On-Premise   │      │       (outbound UDP 51820)          │      │ Cloud        │
 │ LAN: .201.254├──────┤ WAN: 10.10.200.254 (private/NAT)    │      │ LAN: .202.253│
 │              ├──────┤          10.10.200.253 (Public) :WAN─┤─────┤              │
 │ WG: 10.0.0.1 │      │                                     │      │ WG: 10.0.0.2 │
 └──────────────┘      └─────────────────────────────────────┘      └──────────────┘

                  WireGuard Tunnel: 10.0.0.0/30 | Port: 51820/UDP
                  FW01 → Initiator (no public IP, keepalive=25s)
                  FW02 → Listener  (public IP, dynamic endpoint for FW01)
```

**Bảng quy hoạch IP:**

| Thiết bị | Interface | IP Address | Subnet | Vai trò |
|----------|-----------|-----------|--------|---------|
| FW01 | WAN | 10.10.200.254 | /24 | Private IP / NAT (On-Premise) |
| FW01 | LAN | 10.10.201.254 | /24 | Gateway Site A (VLAN 201) |
| FW01 | WG (tun_wg0) | 10.0.0.1 | /30 | WireGuard Tunnel — Initiator |
| FW02 | WAN | 10.10.200.253 | /24 | **Public IP** (Cloud) |
| FW02 | LAN | 10.10.202.253 | /24 | Gateway Site B (VLAN 202) |
| FW02 | WG (tun_wg0) | 10.0.0.2 | /30 | WireGuard Tunnel — Listener |

**WireGuard Tunnel — Vai trò:**

| | FW01 — On-Premise (Initiator) | FW02 — Cloud (Listener) |
|---|---|---|
| Vai trò | Client — chủ động kết nối | Server — lắng nghe kết nối |
| Public IP | ❌ Không có | ✅ 10.10.200.253 (lab) |
| Listen Port | 51820 | 51820 |
| Tunnel Address | 10.0.0.1/30 | 10.0.0.2/30 |
| Peer Endpoint | `10.10.200.253:51820` (IP public FW02) | **Không cần** (Dynamic Endpoint ✅) |
| Keep Alive | `25` (bắt buộc — duy trì NAT) | `0` hoặc không set |
| Allowed IPs | 10.0.0.2/32, 10.10.202.0/24 | 10.0.0.1/32, 10.10.201.0/24 |

> **Giải thích:** FW01 không có IP public nên FW02 không thể chủ động kết nối đến FW01. Thay vào đó:
> - FW01 set endpoint = IP public của FW02 → chủ động gửi handshake
> - FW02 set **Dynamic Endpoint** cho FW01 → tự động học IP:port của FW01 sau khi nhận handshake
> - FW01 set **Keep Alive = 25s** → gửi packet định kỳ để giữ NAT mapping trên ISP router, đảm bảo tunnel không bị drop

---

### 3. Cài đặt WireGuard package trên pfSense

Thực hiện trên **cả 2 pfSense** (FW01 và FW02):

1. Truy cập **System → Package Manager → Available Packages**
2. Tìm kiếm **"WireGuard"**
3. Nhấn **Install** → **Confirm**

Sau khi cài xong, menu **VPN → WireGuard** sẽ xuất hiện.


![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/01.png)

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/02.png)

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/03.png)

---

### 4. Cấu hình WireGuard trên FW01 — Site A (On-Premise, Initiator)

> FW01 không có IP public → đóng vai trò **Initiator** — chủ động kết nối đến FW02 cloud.

#### 4.1. Tạo Tunnel

Truy cập **VPN → WireGuard → Tunnels → Add Tunnel**

| Tham số | Giá trị |
|---------|---------|
| Enable | ✅ |
| Description | `WG-S2S-OnPrem` |
| Listen Port | `51820` |
| Interface Keys | Nhấn **Generate** để tạo cặp key |
| Interface Addresses | `10.0.0.1/30` |

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/04.png)

Nhấn **Save Tunnel** → **Copy Public Key** của FW01 ra notepad để dùng cho bước cấu hình peer trên FW02.

#### 4.2. Thêm Peer (FW02 — Cloud)

Truy cập **VPN → WireGuard → Peers → Add Peer**

| Tham số | Giá trị |
|---------|---------|
| Enable | ✅ |
| Tunnel | `tun_wg0 (WG-S2S-OnPrem)` |
| Description | `FW02-Cloud` |
| Dynamic Endpoint | ❌ **Bỏ tick** (vì FW02 có IP public cố định) |
| Endpoint | `10.10.200.253` *(thay bằng Public IP thật của FW02 cloud)* |
| Endpoint Port | `51820` |
| Keep Alive | **`25`** *(bắt buộc — duy trì NAT mapping)* |
| Public Key | *(Public Key của FW02 - lấy từ bước 5.1)* |
| Allowed IPs | `10.0.0.2/32` |
| Allowed IPs | `10.10.202.0/24` |

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/05.png)

> **Quan trọng:**
> - **Dynamic Endpoint = OFF** vì FW02 có IP public cố định, FW01 cần biết chính xác endpoint để khởi tạo kết nối
> - **Keep Alive = 25** là **bắt buộc** cho site on-prem nằm sau NAT. Nếu không set, NAT table trên ISP router sẽ timeout và tunnel bị mất kết nối
> - **Allowed IPs** phải bao gồm cả tunnel IP của peer (`10.0.0.2/32`) VÀ subnet LAN phía bên kia (`10.10.202.0/24`). Đây là cơ chế crypto-routing của WireGuard

Nhấn **Save Peer**.

---

### 5. Cấu hình WireGuard trên FW02 — Site B (Cloud, Listener)

> FW02 có IP public → đóng vai trò **Listener** — lắng nghe kết nối từ FW01 on-prem.

#### 5.1. Tạo Tunnel

Truy cập **VPN → WireGuard → Tunnels → Add Tunnel**

| Tham số | Giá trị |
|---------|---------|
| Enable | ✅ |
| Description | `WG-S2S-Cloud` |
| Listen Port | `51820` |
| Interface Keys | Nhấn **Generate** để tạo cặp key |
| Interface Addresses | `10.0.0.2/30` |

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/06.png)

Nhấn **Save Tunnel** → **Copy Public Key** của FW02 để paste vào cấu hình peer trên FW01 (bước 4.2).

#### 5.2. Thêm Peer (FW01 — On-Premise)

Truy cập **VPN → WireGuard → Peers → Add Peer**

| Tham số | Giá trị |
|---------|---------|
| Enable | ✅ |
| Tunnel | `tun_wg0 (WG-S2S-Cloud)` |
| Description | `FW01-OnPrem` |
| Dynamic Endpoint | ✅ **Tick chọn** *(FW01 không có IP public → FW02 không biết trước IP)* |
| Endpoint | *(để trống — WireGuard tự học sau handshake)* |
| Endpoint Port | *(để trống)* |
| Keep Alive | `0` hoặc để trống |
| Public Key | *(Public Key của FW01 - lấy từ bước 4.1)* |
| Allowed IPs | `10.0.0.1/32` |
| Allowed IPs | `10.10.201.0/24` |

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/07.png)

> **Điểm khác biệt quan trọng so với cấu hình 2 đầu public IP:**
> - **Dynamic Endpoint = ON** — FW02 không biết trước IP của FW01 (vì FW01 nằm sau NAT). Sau khi FW01 gửi handshake, WireGuard tự ghi nhận source IP:port và dùng nó để gửi traffic ngược lại
> - **Không cần set Endpoint IP/Port** — để WireGuard tự học
> - **Keep Alive = 0** — Phía cloud (listener) không cần keepalive. FW01 (initiator) mới cần keepalive để giữ NAT

Nhấn **Save Peer**.

---

### 6. Assign Interface WireGuard

Thực hiện trên **cả 2 pfSense**:

1. Truy cập **Interfaces → Assignments**
2. Ở **Available network ports**, chọn **tun_wg0** → Nhấn **Add**
3. Click vào interface vừa assign (thường là **OPT1**)
4. Cấu hình:

| Tham số | Giá trị |
|---------|---------|
| Enable | ✅ |
| Description | `WG` |
| IPv4 Configuration Type | **Static IPv4** |
| IPv4 Address | FW01: `10.0.0.1/30` — FW02: `10.0.0.2/30` |
| IPv4 Upstream Gateway | **None** |

> **Lưu ý:** Trên pfSense 2.7.x, bước assign interface là **không bắt buộc** nếu bạn chỉ dùng firewall rules trên tab WireGuard. Tuy nhiên, assign interface giúp bạn quản lý rules rõ ràng hơn và có thể sử dụng gateway cho routing nâng cao.

Nhấn **Save** → **Apply Changes**.

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/08.png)

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/09.png)

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/10.png)

---

### 7. Cấu hình Firewall Rules

#### 7.1. WAN Rules - Cho phép WireGuard

> **Chỉ cần tạo trên FW02 (Cloud)** — vì FW02 là listener, nhận kết nối inbound trên port 51820.
> FW01 (on-prem) **không cần** WAN rule cho WireGuard vì traffic là outbound (FW01 chủ động kết nối ra ngoài).

**Trên FW02 (Cloud)**, truy cập **Firewall → Rules → WAN** → **Add** rule:

| Tham số | Giá trị |
|---------|--------|
| Action | **Pass** |
| Interface | WAN |
| Protocol | **UDP** |
| Source | **any** |
| Destination | **WAN address** |
| Destination Port | **51820** |
| Description | `Allow WireGuard Inbound` |

Nhấn **Save** → **Apply Changes**.

> **Trên cloud provider** (AWS, GCP, Azure...) ngoài firewall rule trên pfSense, bạn còn cần mở **Security Group / Firewall Rule** cho phép **UDP 51820 inbound** trên cloud platform.
>
> **FW01 (On-Prem):** Không cần mở port WAN vì FW01 gửi packet outbound. Response traffic từ FW02 sẽ đi qua NAT mapping đã được tạo bởi outbound connection. Tuy nhiên nếu ISP/router phía trước FW01 có firewall chặn outbound UDP, cần đảm bảo cho phép UDP 51820 outbound.

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/11.png)

#### 7.2. WireGuard Interface Rules

Trên **cả 2 pfSense**, truy cập **Firewall → Rules → WireGuard** (hoặc **OPT1** nếu đã assign) → **Add**:

| Tham số | Giá trị |
|---------|---------|
| Action | **Pass** |
| Interface | WireGuard |
| Protocol | **any** |
| Source | **any** |
| Destination | **any** |
| Description | `Allow all WireGuard traffic` |

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/12.png)

> **Trong môi trường production**, bạn nên giới hạn source/destination cụ thể thay vì allow any-any. Ví dụ chỉ cho phép từ `10.10.201.0/24` đến `10.10.202.0/24` và ngược lại.

Nhấn **Save** → **Apply Changes**.

#### 7.3. LAN Rules

LAN rules mặc định trên cả 2 pfSense đang có:
- **Anti-Lockout Rule** (cho phép truy cập web GUI)
- **Default allow LAN to LAN** (cho phép LAN subnet ra mọi nơi)
- **Block all** (chặn còn lại)

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/13.png)


Rule **"Default allow LAN to LAN"** đã đủ để cho phép traffic từ LAN đi qua WireGuard tunnel. Nếu bạn muốn kiểm soát chặt hơn, có thể tạo rule cụ thể:

**Trên FW01 - LAN:**

| Tham số | Giá trị |
|---------|---------|
| Action | Pass |
| Source | LAN net (10.10.201.0/24) |
| Destination | Network `10.10.202.0/24` |
| Description | `Allow LAN to Site B via VPN` |

**Trên FW02 - LAN:**

| Tham số | Giá trị |
|---------|---------|
| Action | Pass |
| Source | LAN net (10.10.202.0/24) |
| Destination | Network `10.10.201.0/24` |
| Description | `Allow LAN to Site A via VPN` |

---

### 8. Kiểm tra kết nối

#### 8.1. Kiểm tra trạng thái WireGuard

Truy cập **Status → WireGuard** trên cả 2 pfSense:

Kiểm tra:
- **Handshake**: Phải hiển thị thời gian handshake gần nhất (vd: `X seconds ago`)
- **Transfer**: Có dữ liệu sent/received
- **Endpoint**: Hiển thị IP:port của peer

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/14.png)

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/15.png)

#### 8.2. Ping test từ pfSense

**Trên FW01**, truy cập **Diagnostics → Ping**:

```
Ping 10.0.0.2 (tunnel IP FW02)     → phải thành công
Ping 10.10.202.253 (LAN FW02)      → phải thành công
```

**Trên FW02**, truy cập **Diagnostics → Ping**:

```
Ping 10.0.0.1 (tunnel IP FW01)     → phải thành công
Ping 10.10.201.254 (LAN FW01)      → phải thành công
```

> **Quan trọng:** Khi ping đến subnet LAN phía bên kia, phải chọn **Source Address** là **LAN address** hoặc **WireGuard address**, không dùng default (WAN).

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/16.png)


#### 8.3. Ping test từ VM

- Các VM trong 2 site ping thông qua lại 

![](/assets/img/2026-04-14-pfsense-wireguard-site-to-site-vpn/17.png)

---

### 9. Lời kết

WireGuard Site-to-Site VPN trên pfSense là giải pháp đơn giản, hiệu quả để kết nối On-Premise với Cloud. So với IPsec:

| Tiêu chí | WireGuard | IPsec |
|----------|-----------|------|
| Cấu hình | Đơn giản | Phức tạp (Phase 1, Phase 2) |
| Hiệu năng | Cao (kernel module) | Trung bình |
| Codebase | ~4000 dòng | Hàng trăm nghìn dòng |
| Mã hoá | ChaCha20, Curve25519 | Tuỳ chọn (AES, 3DES...) |
| NAT Traversal | Native (keepalive) | Cần NAT-T (UDP 4500) |
| 1 đầu không có Public IP | ✅ Hỗ trợ tốt | ⚠️ Phức tạp hơn |

**Tổng kết cấu hình On-Premise ↔ Cloud:**

| Bước | FW01 (On-Prem) | FW02 (Cloud) |
|------|---------------|-------------|
| 1. Cài WireGuard | ✅ | ✅ |
| 2. Tạo tunnel + keypair | ✅ | ✅ |
| 3. Peer endpoint | Set Public IP của FW02 | **Dynamic Endpoint** |
| 4. Keep Alive | **25s** (bắt buộc) | 0 (không cần) |
| 5. WAN rule UDP 51820 | Không cần (outbound) | **Bắt buộc** (inbound) |
| 6. Cloud Security Group | N/A | Mở UDP 51820 inbound |
| 7. WG interface rule | Allow tunnel traffic | Allow tunnel traffic |

> **Key takeaway:** Chỉ cần 1 đầu có IP public (cloud), đầu on-prem chủ động kết nối outbound + keepalive giữ tunnel — WireGuard xử lý NAT traversal hoàn toàn tự động.

Chúc bạn lab thành công!
