---
title: "IPsec S2S VPN: Check Point R81.20 + pfSense CARP và Zabbix Monitor qua SNMP"
categories:
- Network
- Monitoring
tags:
- checkpoint
- pfsense
- ipsec
- vpn
- zabbix
- snmp
- docker
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### IPsec S2S VPN dual-tunnel giữa pfSense CARP HA và Check Point R81.20, Zabbix monitor qua SNMP
---

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/topo.png)

IPsec S2S VPN giữa pfSense và Check Point là bài toán phổ biến khi 2 site dùng thiết bị khác vendor. Lab này dựng thêm một lớp redundancy bằng CARP HA: 2 pfSense node, mỗi node tạo 1 IPsec tunnel riêng về Check Point gateway — tổng cộng 2 tunnel song song, đảm bảo kết nối không bị gián đoạn khi 1 node gặp sự cố. Trên đường tunnel này, mình triển khai Zabbix 7.0 LTS bằng Docker Compose để monitor Ubuntu VM phía Check Point qua SNMPv2c.

| Component | Version |
|-----------|---------|
| Check Point | R81.20 (Gaia) |
| pfSense | 2.8.1 CE |
| Zabbix | 7.0 LTS |
| Ubuntu | 24.04 LTS |

- [1. Mô hình lab](#1-mô-hình-lab)
- [2. Chuẩn bị Objects trên SmartConsole](#2-chuẩn-bị-objects-trên-smartconsole)
- [3. Cấu hình VPN Domain](#3-cấu-hình-vpn-domain)
- [4. Tạo Star VPN Community](#4-tạo-star-vpn-community)
- [5. Security Policy](#5-security-policy)
- [6. Cấu hình pfSense-1](#6-cấu-hình-pfsense-1)
  - [Phase 1](#phase-1)
  - [Phase 2](#phase-2)
  - [Firewall Rules](#firewall-rules)
  - [Kiểm tra trạng thái tunnel](#kiểm-tra-trạng-thái-tunnel)
- [7. Cấu hình pfSense-2](#7-cấu-hình-pfsense-2)
  - [Kiểm tra 2 tunnel trên Check Point](#kiểm-tra-2-tunnel-trên-check-point)
- [8. Cấu hình CARP VIP LAN trên pfSense](#8-cấu-hình-carp-vip-lan-trên-pfsense)
- [9. Triển khai Zabbix 7.0 LTS bằng Docker Compose](#9-triển-khai-zabbix-70-lts-bằng-docker-compose)
  - [Cài Docker](#cài-docker)
  - [Docker Compose file](#docker-compose-file)
  - [Khởi động](#khởi-động)
  - [Truy cập Zabbix từ ngoài qua pfSense Port Forward](#truy-cập-zabbix-từ-ngoài-qua-pfsense-port-forward)
- [10. Cấu hình SNMP trên ubuntu-202-11](#10-cấu-hình-snmp-trên-ubuntu-202-11)
- [11. Verify VPN tunnel](#11-verify-vpn-tunnel)
- [12. Thêm host vào Zabbix và kiểm tra monitoring](#12-thêm-host-vào-zabbix-và-kiểm-tra-monitoring)
  - [Bật SNMP trên Check Point Gaia](#bật-snmp-trên-check-point-gaia)
  - [Bật SNMP trên pfSense](#bật-snmp-trên-pfsense)
  - [Thêm Check Point và pfSense vào Zabbix](#thêm-check-point-và-pfsense-vào-zabbix)
- [13. Test Failover: Node pfSense-1 down](#13-test-failover-node-pfsense-1-down)
  - [Chuẩn bị: xác nhận baseline](#chuẩn-bị-xác-nhận-baseline)
  - [Test — Tắt pfSense-1 hoàn toàn](#test--tắt-pfsense-1-hoàn-toàn)
- [Lời kết](#lời-kết)

### 1. Mô hình lab

Mình chia lab thành 2 site:

**Site A — pfSense + Zabbix (VLAN 201):**

| Node | WAN (VLAN 200) | LAN | Vai trò |
|------|----------------|-----|---------|
| pfsense-1 | 10.10.200.11 | 10.10.201.254/24 | CARP Master |
| pfsense-2 | 10.10.200.12 | 10.10.201.253/24 | CARP Backup |
| — | — | 10.10.201.50 | CARP LAN VIP |
| zabbix-201-11 | — | 10.10.201.11/24 | Zabbix 7.0 (Docker) |

**Site B — Check Point (VLAN 202):**

| Node | WAN (VLAN 200) | LAN | Vai trò |
|------|----------------|-----|---------|
| checkpoint | 10.10.200.21 | 10.10.202.254/24 | Gateway R81.20 |
| ubuntu-202-11 | — | 10.10.202.11/24 | Target SNMP |

2 IPsec tunnel bảo vệ traffic giữa `10.10.201.0/24` và `10.10.202.0/24`. Zabbix tại `10.10.201.11` kết nối SNMP tới `10.10.202.11` qua tunnel này.

### 2. Chuẩn bị Objects trên SmartConsole

Mình cấu hình phía Check Point trước — tạo các object đại diện cho 2 pfSense node của Site A.

**Tạo Interoperable Device cho pfSense-1:** Vào **Objects → New → Gateways and Servers → Interoperable Device...**:

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/01.png)

Điền thông tin pfsense-1 và click **OK**:

| Field | Value |
|-------|-------|
| Name | pfsense-1 |
| IPv4 Address | 10.10.200.11 |

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/02.png)

Tương tự tạo **pfsense-2** với IP `10.10.200.12`.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/03.png)

**Tạo Network object cho LAN Site A:** Vào **Objects → New → Network...**:

| Field | Value |
|-------|-------|
| Name | net-monitoring-lan |
| Network address | 10.10.201.0 |
| Net mask | 255.255.255.0 |

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/04.png)

**Tạo Network object cho LAN Site B:**

| Field | Value |
|-------|-------|
| Name | net-customer-lan |
| Network address | 10.10.202.0 |
| Net mask | 255.255.255.0 |

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/05.png)

### 3. Cấu hình VPN Domain

Mình khai báo VPN Domain cho từng object để Check Point biết subnet nào nằm sau mỗi peer.

**Gateway Check Point** — double-click gateway → **Network Management** → **VPN Domain**:
- Chọn **User defined** → chọn `net-customer-lan`

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/06.png)

**Object pfsense-1** — double-click → **IPSec VPN** → **VPN Domain**:
- Chọn **User defined** → chọn `net-monitoring-lan`

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/07.png)

**Object pfsense-2** — tương tự, chọn `net-monitoring-lan` (cùng subnet vì CARP pair).

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/08.png)

### 4. Tạo Star VPN Community

Mình tạo Star Community với Check Point làm center, 2 pfSense là satellite.

Vào menu **New... → More → VPN Community → Star Community...**, đặt tên `VPN-Monitoring`.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/09.png)

**Tab Gateways:**
- **Center Gateways:** thêm gateway `Checkpoint`
- **Satellite Gateways:** thêm `pfsense-1` và `pfsense-2`

Failover giữa 2 tunnel hoạt động tự nhiên: pfsense-1 và pfsense-2 cùng là satellite với VPN domain `net-monitoring-lan`. Check Point duy trì Permanent Tunnel đến cả 2 — khi tunnel pfsense-1 down, traffic tự động chuyển qua pfsense-2 mà không cần cấu hình thêm.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/10.png)

**Tab Encryption** — chọn **Custom encryption suite**, IKEv2 only:

IKE Security Association (Phase 1):

| Parameter | Value |
|-----------|-------|
| Encryption Algorithm | AES-256 |
| Data Integrity | SHA256 |
| Diffie-Hellman Group | Group 14 (2048 bit) |

IKE Security Association (Phase 2):

| Parameter | Value |
|-----------|-------|
| Encryption Algorithm | AES-256 |
| Data Integrity | SHA256 |
| Use Perfect Forward Secrecy | ✓ |
| Diffie-Hellman Group | Group 14 (2048 bit) |

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/11.png)

**Tab Tunnel Management:**
- **Set Permanent Tunnels:** ✓ → chọn **On specific tunnels in the community** → click **Select Gateways...** chọn `pfsense-1` và `pfsense-2`
- **Tunnel down track:** Log
- **VPN Tunnel Sharing:** One VPN tunnel per subnet pair

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/12.png)

**Tab Shared Secret:**

| Peer Name | Shared Secret |
|-----------|---------------|
| pfsense-1 | Zxc123!@# |
| pfsense-2 | Zxc123!@# |

> Password trong bài dùng `Zxc123!@#` — đổi lại trong môi trường thực tế.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/13.png)

Click **OK** để lưu community.

### 5. Security Policy

Mình thêm rule cho phép traffic 2 chiều qua VPN. Vào **Security Policies → Policy → Add Rule**:

| No. | Name | Source | Destination | VPN | Services & Applications | Action | Track |
|-----|------|--------|-------------|-----|------------------------|--------|-------|
| 2 | SiteA-to-SiteB | net-monitoring-lan | net-customer-lan | VPN-Monitoring | Any | Accept | Log |
| 3 | SiteB-to-SiteA | net-customer-lan | net-monitoring-lan | VPN-Monitoring | Any | Accept | Log |

**Publish** → **Install Policy** lên gateway Check Point.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/14.png)

### 6. Cấu hình pfSense-1

Mình cấu hình IPsec tunnel trên pfSense-1 kết nối về Check Point.

#### Phase 1

Vào **VPN → IPsec → Tunnels → Add P1**:

| Parameter | Value |
|-----------|-------|
| Description | VPN-To-Checkpoint |
| Key Exchange version | IKEv2 |
| Internet Protocol | IPv4 |
| Interface | WAN |
| Remote Gateway | 10.10.200.21 |
| Authentication Method | Mutual PSK |
| My Identifier | My IP address |
| Peer Identifier | Peer IP address |
| Pre-Shared Key | Zxc123!@# |
| Encryption Algorithm | AES |
| Key length | 256 bits |
| Hash | SHA256 |
| DH Group | 14 (2048 bit) |
| Life Time | 86400 |
| NAT Traversal | Auto |
| Dead Peer Detection | ✓ Enable DPD |
| Delay | 10 |
| Max failures | 5 |

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/15.png)

#### Phase 2

Click **Show Phase 2 Entries → Add P2**:

| Parameter | Value |
|-----------|-------|
| Description | VPN-To-Checkpoint-P2 |
| Mode | Tunnel IPv4 |
| Local Network | LAN subnet |
| NAT/BINAT translation | None |
| Remote Network | Network — `10.10.202.0/24` |
| Protocol | ESP |
| Encryption Algorithms | ✓ AES 256 bits |
| Hash Algorithms | ✓ SHA256 |
| PFS key group | 14 (2048 bit) |
| Life Time | 3600 |
| Automatically ping host | (trống) |
| Keep Alive | ✓ Enable periodic keep alive check |

**Save → Apply Changes.**

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/16.png)

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/17.png)

#### Firewall Rules

Vào **Firewall → Rules → IPsec → Add**:

| Field | Value |
|-------|-------|
| Action | Pass |
| Interface | IPsec |
| Address Family | IPv4 |
| Protocol | Any |
| Source | Network — `10.10.202.0/24` |
| Destination | Network — `10.10.201.0/24` |
| Description | Allow Site-B inbound |

**Save → Apply Changes.**

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/18.png)

#### Kiểm tra trạng thái tunnel

Vào **Status → IPsec → Overview** để xác nhận tunnel pfSense-1 lên thành công. Phase 1 hiện **Established** và Phase 2 hiện **Installed** là đã kết nối xong.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/19.png)

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/20.png)

### 7. Cấu hình pfSense-2

pfSense-2 cấu hình hoàn toàn tương tự pfSense-1. WAN interface khác (`10.10.200.12`) nên Check Point nhận diện đây là tunnel riêng biệt.

**Phase 1** — giống pfSense-1, Remote Gateway vẫn là `10.10.200.21`, Pre-Shared Key `Zxc123!@#`.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/21.png)

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/22.png)

**Phase 2** — giống pfSense-1: Local `10.10.201.0/24`, Remote `10.10.202.0/24`.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/23.png)


**Firewall Rules → IPsec** — thêm rule giống pfSense-1.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/25.png)

#### Kiểm tra 2 tunnel trên Check Point

Mở **Check Point SmartView Monitor** → **Tunnels → Tunnels on Gateway → Checkpoint**. Cả 2 tunnel về pfsense-1 (`10.10.200.11`) và pfsense-2 (`10.10.200.12`) đều hiển thị **State: Up** là thành công.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/24.png)

### 8. Cấu hình CARP VIP LAN trên pfSense

Mình tạo CARP VIP `10.10.201.50` trên LAN interface của cả 2 pfSense để Zabbix dùng làm default gateway — khi pfSense-1 down, VIP tự động chuyển sang pfSense-2 mà không cần đổi cấu hình gì trên Zabbix.

**Trên pfSense-1** — vào **Firewall → Virtual IPs → Add**:

| Field | Value |
|-------|-------|
| Type | CARP |
| Interface | LAN |
| IP Address | 10.10.201.50 / 24 |
| Virtual IP Password | Zxc123!@# |
| VHID Group | 1 |
| Advertising Frequency | Base: 1 / Skew: 0 |
| Description | CARP-LAN-VIP |

**Save → Apply Changes.**

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/26.png)

**Trên pfSense-2** — cấu hình giống hệt pfSense-1, chỉ khác **Skew: 100** để pfSense-2 là backup.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/27.png)

Sau khi apply trên cả 2 node, vào **Status → Dashboard → widget CARP Status** để xác nhận:

- pfSense-1: `10.10.201.50` → **MASTER**
- pfSense-2: `10.10.201.50` → **BACKUP**

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/28.png)

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/29.png)

Trên VM `zabbix-201-11`, đặt default gateway là `10.10.201.50`:

```bash
# Kiểm tra gateway hiện tại
ip route show default

# Đổi gateway qua netplan (Ubuntu 24.04)
sed -i 's/via: .*/via: 10.10.201.50/' /etc/netplan/50-cloud-init.yaml
netplan apply

# Xác nhận sau khi apply
ip route show default
```

### 9. Triển khai Zabbix 7.0 LTS bằng Docker Compose

Mình cài Docker và dựng Zabbix trên `zabbix-201-11` (`10.10.201.11`).

#### Cài Docker

```bash
apt update && apt install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  > /etc/apt/sources.list.d/docker.list
apt update && apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

#### Docker Compose file

```bash
mkdir -p /opt/zabbix && cat > /opt/zabbix/docker-compose.yml << 'EOF'
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: Zxc123!@#
    volumes:
      - postgres_data:/var/lib/postgresql/data

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:ubuntu-7.0-latest
    restart: unless-stopped
    ports:
      - "10051:10051"
    environment:
      DB_SERVER_HOST: postgres
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: Zxc123!@#
    depends_on:
      - postgres

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:ubuntu-7.0-latest
    restart: unless-stopped
    ports:
      - "80:8080"
    environment:
      DB_SERVER_HOST: postgres
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: Zxc123!@#
      ZBX_SERVER_HOST: zabbix-server
      PHP_TZ: Asia/Ho_Chi_Minh
    depends_on:
      - zabbix-server
      - postgres

volumes:
  postgres_data:
EOF
```

#### Khởi động

```bash
cd /opt/zabbix
docker compose up -d
docker compose ps
```

Truy cập Zabbix web tại `http://10.10.201.11` — login mặc định `Admin / zabbix`.

#### Truy cập Zabbix từ ngoài qua pfSense Port Forward

Nếu không truy cập được trực tiếp vào VLAN 201, mình cấu hình NAT Port Forward trên pfSense-1 để forward `http://10.10.200.11:8080` vào `10.10.201.11:80`.

Vào **Firewall → NAT → Port Forward → Add**:

| Field | Value |
|-------|-------|
| Interface | WAN |
| Address Family | IPv4 |
| Protocol | **TCP** ← phải chọn TCP để hiện port fields |
| Destination / Type | WAN address |
| Destination port range | 8080 → 8080 |
| Redirect target IP / Type | Address or Alias |
| Redirect target IP / Address | 10.10.201.11 |
| Redirect target port | 80 |
| Description | Zabbix-Web-Forward |
| Filter rule association | Add associated filter rule |

**Save → Apply Changes.**

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/30.png)

Truy cập Zabbix tại `http://10.10.200.11:8080`.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/31.png)

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/32.png)

### 10. Cấu hình SNMP trên ubuntu-202-11

Mình cài và cấu hình `snmpd` trên `ubuntu-202-11` (`10.10.202.11`) để Zabbix poll SNMP qua tunnel.

```bash
apt update && apt install -y snmpd snmp
```

Mình ghi đè config mặc định để cho phép Zabbix đọc qua community string:

```bash
cat > /etc/snmp/snmpd.conf << 'EOF'
agentaddress udp:161
rocommunity public 127.0.0.1
rocommunity public 10.10.201.0/24
sysLocation "Customer Site"
sysContact admin@customer.local
EOF
```

Ubuntu mặc định disable MIB names trong `/etc/snmp/snmp.conf` — bỏ comment dòng đó để `snmpwalk` dùng được tên OID:

```bash
apt install -y snmp-mibs-downloader
download-mibs

sed -i 's/^mibs :/# mibs :/' /etc/snmp/snmp.conf
```

```bash
systemctl enable snmpd && systemctl restart snmpd
```

Kiểm tra SNMP từ chính máy:

```bash
snmpwalk -v2c -c public 127.0.0.1 system
```

### 11. Verify VPN tunnel

Trước khi thêm host vào Zabbix, mình xác nhận tunnel đã up.

**Ping test từ zabbix-201-11:**

```bash
ping -c 4 10.10.202.11
```

**SNMP test qua tunnel:**

```bash
# Hostname
snmpget -v2c -c public 10.10.202.11 SNMPv2-MIB::sysName.0

# Danh sách IP interfaces
snmpwalk -v2c -c public 10.10.202.11 1.3.6.1.2.1.4.20
```

Nhận được sysName và danh sách IP interface trả về là tunnel hoạt động và SNMP reachable qua đường VPN.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/33.png)

### 12. Thêm host vào Zabbix và kiểm tra monitoring

Mình vào Zabbix web (`http://10.10.201.11`) → **Configuration → Hosts → Create host**.

**Tab Host:**

| Field | Value |
|-------|-------|
| Host name | ubuntu-202-11 |
| Visible name | Ubuntu Site B |
| Groups | Linux servers |

**Tab Interfaces — Add → SNMP:**

| Field | Value |
|-------|-------|
| IP address | 10.10.202.11 |
| Port | 161 |
| SNMP version | SNMPv2 |
| SNMP community | public |

**Tab Templates:**
- Thêm template **Linux by SNMP**

Click **Add** để lưu host.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/34.png)

Sau 1–2 phút, mình vào **Monitoring → Latest data** → filter host `ubuntu-202-11`. Các item như CPU utilization, memory, network interfaces sẽ bắt đầu có data.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/35.png)

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/36.png)

#### Bật SNMP trên Check Point Gaia

Mình vào Gaia Portal (`https://10.10.200.21`) → **System Management → SNMP**.

**SNMP General Settings** — tick **Enable SNMP Agent**, Version giữ nguyên **v1 / v2 / v3 (any)**, click **Apply**.

**Agent Interfaces** — giữ tất cả interface được chọn (eth0, eth1, lo).

**V1 / V2 Settings** — điền `public` vào **Read Only Community String**, click **Apply**.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/36a.png)

#### Bật SNMP trên pfSense

Thực hiện trên **cả 2 node** pfSense-1 và pfSense-2. Vào **Services → SNMP**:

**SNMP Daemon** — tick **Enable the SNMP Daemon and its controls**.

**SNMP Daemon Settings:**

| Field | Value |
|-------|-------|
| Polling Port | 161 |
| Read Community String | public |

**SNMP Modules** — giữ nguyên mặc định (MibII, Netgraph, PF, Host Resources, UCD, Regex đều được chọn).

**Interface Binding:**

| Field | Value |
|-------|-------|
| Internet Protocol | IPv4 |
| Bind Interfaces | LAN |

Click **Save**.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/36b.png)

#### Thêm Check Point và pfSense vào Zabbix

Mình dùng toàn bộ **LAN IP** để tránh phụ thuộc vào WAN routing: pfSense-1/2 reach trực tiếp qua LAN 201, Check Point reach qua IPsec tunnel (LAN 202). Thêm lần lượt 3 host vào **Configuration → Hosts → Create host**.

**Host: checkpoint** — dùng IP LAN `10.10.202.254` (qua IPsec tunnel):

| Tab | Field | Value |
|-----|-------|-------|
| Host | Host name | checkpoint |
| Host | Groups | Applications |
| Interfaces | IP address | 10.10.202.254 |
| Interfaces | Port | 161 |
| Interfaces | SNMP version | SNMPv2 |
| Interfaces | SNMP community | public |
| Templates | — | Check Point Next Generation Firewall by SNMP |

**Host: pfsense-1** — dùng IP LAN `10.10.201.254`:

| Tab | Field | Value |
|-----|-------|-------|
| Host | Host name | pfsense-1 |
| Host | Groups | Applications |
| Interfaces | IP address | 10.10.201.254 |
| Interfaces | Port | 161 |
| Interfaces | SNMP version | SNMPv2 |
| Interfaces | SNMP community | public |
| Templates | — | PFSense by SNMP |

**Host: pfsense-2** — tương tự pfsense-1, IP `10.10.201.253`.

Click **Add** cho từng host. Sau 1–2 phút, mình vào **Monitoring → Latest data** → filter từng host để xác nhận data đang về.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/36c.png)

### 13. Test Failover: Node pfSense-1 down

Thiết kế này failover theo **cấp node**: CARP VIP gắn với pfSense node đang sống, không tự chuyển khi chỉ tunnel drop mà node vẫn chạy. Nếu chỉ tắt tunnel trên pfSense-1 (node vẫn up), VIP `10.10.201.50` giữ nguyên ở pfSense-1 — `zabbix-201-11` vẫn route qua pfSense-1, không có tunnel nên ping/SNMP sẽ fail. Kịch bản failover thực tế là pfSense-1 down hoàn toàn: VIP và tunnel cùng chuyển sang pfSense-2.

#### Chuẩn bị: xác nhận baseline

Mình xác nhận trạng thái ban đầu trước khi test:

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/37.png)

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/38.png)

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/39.png)

```bash
# Từ zabbix-201-11 — xác nhận SNMP hoạt động trước khi test
snmpget -v2c -c public 10.10.202.11 SNMPv2-MIB::sysName.0
ping -c 4 10.10.202.11
```

#### Test — Tắt pfSense-1 hoàn toàn

**Kịch bản:** pfSense-1 shutdown → CARP VIP chuyển sang pfSense-2 đồng thời Tunnel-1 drop → `zabbix-201-11` tự động đi qua pfSense-2 và Tunnel-2 — Zabbix không mất data.

1. Tắt pfSense-1 VM hoàn toàn.

2. **CARP failover** — trên **pfSense-2** → **Status → Dashboard → widget CARP Status**: sau ~5 giây, entry `10.10.201.50` chuyển từ **BACKUP** sang **MASTER**.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/40.png)

3. **Tunnel failover** — trên **Check Point SmartView Monitor** → Tunnel-2 (`10.10.200.12`) vẫn **Up**.

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/41.png)


4. Kiểm tra SNMP vẫn hoạt động — `zabbix-201-11` lúc này đi qua VIP `10.10.201.50` (pfSense-2 MASTER) → Tunnel-2 → `ubuntu-202-11`:

```bash
# Từ zabbix-201-11
ping -c 4 10.10.202.11
snmpget -v2c -c public 10.10.202.11 SNMPv2-MIB::sysName.0
```

5. Trên **Zabbix** → **Monitoring → Hosts**: host `ubuntu-202-11` vẫn hiển thị data liên tục — không có gap.

> **Lưu ý Port Forward**: Khi pfSense-1 down, URL `http://10.10.200.11:8080` không còn hoạt động. Nếu cần admin access vào Zabbix web khi pfSense-1 down, thêm cùng rule Port Forward trên pfSense-2 (**Firewall → NAT → Port Forward**, WAN `10.10.200.12:8080` → `10.10.201.11:80`).

![](/assets/img/2026-06-23-checkpoint-pfsense-ipsec-dual-tunnel/42.png)

6. **Khôi phục**: bật lại pfSense-1 → pfSense-1 giành lại **MASTER** (Skew 0), Tunnel-1 renegotiate tự động.

### Lời kết

Trọng tâm của lab là cặp pfSense CARP: mỗi node dựng 1 tunnel riêng về Check Point, nên khi pfSense-1 down hoàn toàn, CARP VIP tự chuyển sang pfSense-2 và Tunnel-2 vẫn đang sẵn sàng — Zabbix không mất kết nối. Check Point chỉ đóng vai remote end, không cần can thiệp gì. Mô hình dễ mở rộng: thêm site mới chỉ cần tạo thêm 2 tunnel (1 từ mỗi pfSense), Zabbix server không thay đổi. 
