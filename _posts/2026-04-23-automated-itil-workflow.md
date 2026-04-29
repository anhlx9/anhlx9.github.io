---
title: "Automated ITIL Workflow: Zabbix + ServiceDesk Plus + Grafana + AD DC"
categories:
- Monitoring
- ITIL
- Active Directory

feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Lab Automated ITIL Workflow trên Ubuntu 22.04: Zabbix phát hiện sự cố → SDP ticket round-robin → Grafana, xác thực tập trung qua Active Directory
---

Bài viết xây dựng lab **Automated ITIL Workflow** trên **Ubuntu Server 22.04**, tích hợp **Active Directory Domain Controller** (Windows Server 2022) làm nguồn xác thực tập trung. Toàn bộ dịch vụ cài đặt **trực tiếp trên server**.

**Luồng tự động hóa:**

1. Zabbix 7.4 phát hiện alert (disk full, host down)
2. Webhook tạo ticket trên **ServiceDesk Plus** và gắn KTV theo **round-robin**
3. **Telegram Bot** gửi thông báo tức thì đến group/channel
4. **Grafana** hiển thị realtime metrics và trạng thái ticket

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Kiến trúc lab](#2-kiến-trúc-lab)
  - [2.1. Topology](#21-topology)
  - [2.2. Kế hoạch IP và phần mềm](#22-kế-hoạch-ip-và-phần-mềm)
  - [2.3. Luồng Automated ITIL Workflow](#23-luồng-automated-itil-workflow)
- [3. Chuẩn bị VM dc — AD Domain Controller](#3-chuẩn-bị-vm-dc--ad-domain-controller)
  - [3.1. Cấu hình IP tĩnh và hostname](#31-cấu-hình-ip-tĩnh-và-hostname)
  - [3.2. Cài đặt AD DS và promote Domain Controller](#32-cài-đặt-ad-ds-và-promote-domain-controller)
  - [3.3. Cấu hình DNS Forwarder trên DC](#33-cấu-hình-dns-forwarder-trên-dc)
  - [3.4. Tạo OU, Service Account và tài khoản KTV](#34-tạo-ou-service-account-và-tài-khoản-ktv)
  - [3.5. Cài Zabbix Agent 2 trên Windows Server 2022](#35-cài-zabbix-agent-2-trên-windows-server-2022)
- [4. Chuẩn bị VM itil-stack](#4-chuẩn-bị-vm-itil-stack)
- [5. Setup PostgreSQL 16](#5-setup-postgresql-16)
- [6. Setup Zabbix 7.4](#6-setup-zabbix-74)
  - [6.1. Cài đặt Zabbix Server, Frontend, Agent](#61-cài-đặt-zabbix-server-frontend-agent)
  - [6.2. Import schema và cấu hình database](#62-import-schema-và-cấu-hình-database)
  - [6.3. Cấu hình Zabbix Server](#63-cấu-hình-zabbix-server)
  - [6.4. Cấu hình Nginx và PHP cho Zabbix Frontend](#64-cấu-hình-nginx-và-php-cho-zabbix-frontend)
  - [6.5. Cấu hình Zabbix Web lần đầu](#65-cấu-hình-zabbix-web-lần-đầu)
  - [6.6. Add hosts vào Zabbix](#66-add-hosts-vào-zabbix)
  - [6.7. Cấu hình Zabbix LDAP / Active Directory](#67-cấu-hình-zabbix-ldap--active-directory)
- [7. Setup Grafana](#7-setup-grafana)
  - [7.1. Cài đặt Grafana](#71-cài-đặt-grafana)
  - [7.2. Cấu hình Grafana LDAP / Active Directory](#72-cấu-hình-grafana-ldap--active-directory)
  - [7.3. Kết nối Zabbix datasource và import dashboard](#73-kết-nối-zabbix-datasource-và-import-dashboard)
- [8. Setup ServiceDesk Plus 14](#8-setup-servicedesk-plus-14)
  - [8.1. Cài đặt SDP](#81-cài-đặt-sdp)
  - [8.2. Đăng nhập lần đầu](#82-đăng-nhập-lần-đầu)
  - [8.3. Tích hợp LDAP / Active Directory](#83-tích-hợp-ldap--active-directory)
  - [8.4. Tạo kỹ thuật viên từ AD, cấu hình Round-Robin và Escalation](#84-tạo-kỹ-thuật-viên-từ-ad-cấu-hình-round-robin-và-escalation)
  - [8.5. Lấy API Token — cập nhật macro Zabbix](#85-lấy-api-token--cập-nhật-macro-zabbix)
  - [8.6. Tạo Webhook Media Type gọi SDP API](#86-tạo-webhook-media-type-gọi-sdp-api)
  - [8.7. Tạo Action tự động tạo ticket](#87-tạo-action-tự-động-tạo-ticket)
  - [8.8. Tích hợp Telegram Alert](#88-tích-hợp-telegram-alert)
- [9. Test Automated Workflow](#9-test-automated-workflow)
  - [9.1. Test: Disk Full](#91-test-disk-full)
- [10. Kết quả](#10-kết-quả)

---

### 1. Giới thiệu

Lab triển khai trên **Ubuntu Server 22.04 LTS** với **Active Directory Domain Controller** (Windows Server 2022) làm nguồn xác thực tập trung — tất cả KTV đăng nhập vào SDP, Zabbix và Grafana bằng một tài khoản AD duy nhất. Tất cả dịch vụ cài đặt **trực tiếp trên server** (bare-metal), không sử dụng container.

| Thành phần | Vai trò |
|---|---|
| **Windows Server 2022** | AD DC: xác thực tập trung, DNS nội bộ |
| **Zabbix 7.4** | Giám sát hạ tầng, phát hiện sự cố, gửi alert |
| **ServiceDesk Plus 14** | ITSM platform, quản lý ticket SLA, xác thực qua AD |
| **Grafana** | Dashboard tổng hợp metrics Zabbix |
| **PostgreSQL 16** | Backend database dùng chung cho Zabbix và SDP |

**Phần mềm sử dụng:**

- Zabbix 7.4 (open source, cài từ repo chính thức)
- Grafana latest (cài từ repo chính thức)
- ManageEngine ServiceDesk Plus 14 (trial 30 ngày)
- PostgreSQL 16 (apt)

---

### 2. Kiến trúc lab

#### 2.1. Topology

```
┌──────────────────────────────────────────────────────────────────────┐
│                    VMware ESXi — Lab-vlan200 (10.10.200.0/24)        │
│                                                                      │
│  ┌────────────────────────────────────────────┐                      │
│  │  itil-stack  (10.10.200.11)                │                      │
│  │  Ubuntu Server 22.04 — 16 vCPU, 24GB RAM   │                      │
│  │  100GB Disk                                │                      │
│  │                                            │                      │
│  │  ┌────────────┐  ┌──────────────────────┐  │                      │
│  │  │ Zabbix 7.4 │  │ ServiceDesk Plus 14  │  │                      │
│  │  │ :8080      │  │ :8400                │  │                      │
│  │  └────────────┘  └──────────────────────┘  │                      │
│  │  ┌────────────┐  ┌──────────────────────┐  │                      │
│  │  │  Grafana   │  │   PostgreSQL 16      │  │                      │
│  │  │  :3000     │  │   :5432              │  │                      │
│  │  └────────────┘  └──────────────────────┘  │                      │
│  └────────────────────────────────────────────┘                      │
│              │ Zabbix Agent         │ LDAP Auth                      │
│    ┌─────────┴──────────┐   ┌───────┴────────────────────┐           │
│    ▼                    ▼   ▼                            │           │
│  ┌───────────────────────────────────────────────────────┴──┐        │
│  │  dc  (10.10.200.12)                                      │        │
│  │  Windows Server 2022 — AD DC, DNS                        │        │
│  │  Domain: anhlx.lab                                       │        │
│  │  Zabbix Agent 2 :10050                                   │        │
│  └──────────────────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────────────────┘
```

#### 2.2. Kế hoạch IP và phần mềm

| VM | Hostname | IP | OS | Vai trò |
|---|---|---|---|---|
| Stack | `itil-stack` | `10.10.200.11` | Ubuntu Server 22.04 | Zabbix + Grafana + SDP + PostgreSQL |
| DC | `dc` | `10.10.200.12` | Windows Server 2022 | AD DC + DNS + Zabbix Agent |

**Port mapping trên itil-stack:**

| Service | Port | URL |
|---|---|---|
| PostgreSQL | 5432 | TCP |
| Zabbix Web | 8080 | `http://10.10.200.11:8080` |
| Zabbix Server | 10051 | TCP |
| Zabbix Agent | 10050 | TCP |
| Grafana | 3000 | `http://10.10.200.11:3000` |
| ServiceDesk Plus | 8400 | `https://10.10.200.11:8400` |

**Thông tin AD Domain:**

| Tham số | Giá trị |
|---|---|
| Domain name | `anhlx.lab` |
| NetBIOS | `ANHLX` |
| Domain Controller | `dc.anhlx.lab` (10.10.200.12) |
| LDAP port | 389 |
| Service account | `CN=svc-ldap,OU=ServiceAccounts,OU=ITIL-Lab,DC=anhlx,DC=lab` |

#### 2.3. Luồng Automated ITIL Workflow

```
                     ┌─────────────────┐
                     │   Sự cố xảy ra  │
                     │ (disk full /    │
                     │  host down)     │
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │  Zabbix Agent   │
                     │  phát hiện      │
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │  Zabbix Server  │
                     │  Trigger kích   │
                     │  hoạt (PROBLEM) │
                     └────────┬────────┘
                              │
         ┌────────────────────┼──────────────────────┐
         ▼                    ▼                      ▼
┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐
│  Webhook → SDP  │  │  Telegram Bot   │  │  Grafana       │
│  Tạo ticket     │  │  Gửi alert      │  │  Dashboard     │
│  Round-robin    │  │  tức thì tới    │  │  realtime      │
│  Assign (AD)    │  │  group/channel  │  │  update        │
└────────┬────────┘  └─────────────────┘  └────────────────┘
         │
┌────────▼──────────────────────────────────────────────┐
│  Level 1 — Round-robin (xử lý ban đầu):               │
│  Ticket 1 → nguyenvana@anhlx.lab                      │
│  Ticket 2 → tranthib@anhlx.lab                        │
│  Ticket 3 → levanc@anhlx.lab                          │
│  Ticket 4 → nguyenvana@anhlx.lab ...                  │
└──────────────────────────┬────────────────────────────┘
                           │ Quá SLA / Không xử lý được
             ┌─────────────▼─────────────┐
             │  Level 2 — phamvand       │
             │  (Escalation từ L1)       │
             └─────────────┬─────────────┘
                           │ Quá SLA
             ┌─────────────▼─────────────┐
             │  Level 3 — hoangvane      │
             │  (Escalation từ L2)       │
             └───────────────────────────┘
```

---

### 3. Chuẩn bị VM dc — AD Domain Controller

#### 3.1. Cấu hình IP tĩnh và hostname

Trên Windows Server 2022 vừa cài xong:

```powershell
# Xem tên interface hiện tại
Get-NetAdapter

# Đặt IP tĩnh (thay "Ethernet0" bằng tên interface thực)
New-NetIPAddress -InterfaceAlias "Ethernet0" `
    -IPAddress "10.10.200.12" `
    -PrefixLength 24 `
    -DefaultGateway "10.10.200.1"

# Đặt DNS tự trỏ về chính mình (AD DNS)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" `
    -ServerAddresses "10.10.200.12","8.8.8.8"

# Đổi hostname
Rename-Computer -NewName "dc" -Restart
```

#### 3.2. Cài đặt AD DS và promote Domain Controller

Sau khi máy khởi động lại:

```powershell
# Cài role AD DS và DNS
Install-WindowsFeature -Name AD-Domain-Services, DNS `
    -IncludeManagementTools

# Promote lên Domain Controller — tạo forest mới
Import-Module ADDSDeployment
Install-ADDSForest `
    -DomainName "anhlx.lab" `
    -DomainNetbiosName "ANHLX" `
    -DomainMode "WinThreshold" `
    -ForestMode "WinThreshold" `
    -InstallDns `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "Admin@2026DC" -AsPlainText -Force) `
    -Force
```
<img src="/assets/img/2026-04-23-automated-itil-workflow/01.png"/>

> Máy sẽ tự động khởi động lại sau khi promote thành công. Sau khi reboot, đăng nhập bằng `ANHLX\Administrator`.

Xác nhận AD đã sẵn sàng:

```powershell
# Kiểm tra domain
Get-ADDomain

# Kiểm tra DNS
Resolve-DnsName anhlx.lab
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/02.png"/>

#### 3.3. Cấu hình DNS Forwarder trên DC

Sự kiện sau khi promote DC xong, AD DNS mặc định chỉ resolve nội bộ. Cần bật **forwarder** để DC tự chuyển tiếp các query không thuộc domain ra internet — giúp tất cả VM trong lab chỉ cần truyền một DNS duy nhất là DC.

```powershell
# Cấu hình DNS Forwarder — forward query external ra 8.8.8.8 và 8.8.4.4
Set-DnsServerForwarder -IPAddress "8.8.8.8","8.8.4.4" -PassThru

# Xác nhận
Get-DnsServerForwarder
```

**Kiểm tra forwarder hoạt động:**

```powershell
# Resolve domain nội bộ
Resolve-DnsName dc.anhlx.lab

# Resolve domain external qua forwarder
Resolve-DnsName google.com
```
<img src="/assets/img/2026-04-23-automated-itil-workflow/02b.png"/> 

> Từ đây, tất cả VM trong lab chỉ cần đặt DNS duy nhất là `10.10.200.12`. DC sẽ tự phân giải `anhlx.lab` nội bộ và forward query còn lại (apt, wget, ...) ra internet.

#### 3.4. Tạo OU, Service Account và tài khoản KTV

Tạo cấu trúc OU và tài khoản cần thiết:

```powershell
# Tạo OU
New-ADOrganizationalUnit -Name "ITIL-Lab" -Path "DC=anhlx,DC=lab"
New-ADOrganizationalUnit -Name "Technicians" -Path "OU=ITIL-Lab,DC=anhlx,DC=lab"
New-ADOrganizationalUnit -Name "ServiceAccounts" -Path "OU=ITIL-Lab,DC=anhlx,DC=lab"
New-ADOrganizationalUnit -Name "Groups" -Path "OU=ITIL-Lab,DC=anhlx,DC=lab"

# Tạo Service Account cho LDAP binding (dùng bởi SDP, Zabbix, Grafana)
New-ADUser `
    -Name "svc-ldap" `
    -SamAccountName "svc-ldap" `
    -UserPrincipalName "svc-ldap@anhlx.lab" `
    -Path "OU=ServiceAccounts,OU=ITIL-Lab,DC=anhlx,DC=lab" `
    -AccountPassword (ConvertTo-SecureString "Svc@Ldap2026" -AsPlainText -Force) `
    -PasswordNeverExpires $true `
    -CannotChangePassword $true `
    -Enabled $true

# Tạo 5 tài khoản KTV (3 Level 1 + 1 Level 2 + 1 Level 3)
$ktv = @(
    @{Name="Nguyen Van A"; Sam="nguyenvana"; Email="nguyenvana@anhlx.lab"},
    @{Name="Tran Thi B";   Sam="tranthib";   Email="tranthib@anhlx.lab"},
    @{Name="Le Van C";     Sam="levanc";     Email="levanc@anhlx.lab"},
    @{Name="Pham Van D";   Sam="phamvand";   Email="phamvand@anhlx.lab"},
    @{Name="Hoang Van E";  Sam="hoangvane";  Email="hoangvane@anhlx.lab"}
)

foreach ($k in $ktv) {
    New-ADUser `
        -Name $k.Name `
        -GivenName ($k.Name.Split(" ")[0]) `
        -Surname ($k.Name.Split(" ")[-1]) `
        -SamAccountName $k.Sam `
        -UserPrincipalName "$($k.Sam)@anhlx.lab" `
        -EmailAddress $k.Email `
        -DisplayName $k.Name `
        -Path "OU=Technicians,OU=ITIL-Lab,DC=anhlx,DC=lab" `
        -AccountPassword (ConvertTo-SecureString "Ktv@Abc2026" -AsPlainText -Force) `
        -PasswordNeverExpires $true `
        -Enabled $true
}

# Tạo group theo level — đặt trong OU Groups (tránh trùng tên với OU Technicians)
New-ADGroup -Name "L1" -GroupScope Global -Path "OU=Groups,OU=ITIL-Lab,DC=anhlx,DC=lab"
New-ADGroup -Name "L2" -GroupScope Global -Path "OU=Groups,OU=ITIL-Lab,DC=anhlx,DC=lab"
New-ADGroup -Name "L3" -GroupScope Global -Path "OU=Groups,OU=ITIL-Lab,DC=anhlx,DC=lab"
New-ADGroup -Name "Technicians" -GroupScope Global -Path "OU=Groups,OU=ITIL-Lab,DC=anhlx,DC=lab"

Add-ADGroupMember -Identity "L1" -Members "nguyenvana","tranthib","levanc"
Add-ADGroupMember -Identity "L2" -Members "phamvand"
Add-ADGroupMember -Identity "L3" -Members "hoangvane"
Add-ADGroupMember -Identity "Technicians" -Members "nguyenvana","tranthib","levanc","phamvand","hoangvane"
```

Xác nhận:

```powershell
Get-ADUser -Filter * -SearchBase "OU=Technicians,OU=ITIL-Lab,DC=anhlx,DC=lab" |
    Select-Object Name, SamAccountName, EmailAddress | Format-Table

Get-ADGroupMember "L1" | Select-Object Name
Get-ADGroupMember "L2" | Select-Object Name
Get-ADGroupMember "L3" | Select-Object Name
```
<img src="/assets/img/2026-04-23-automated-itil-workflow/03.png"/>

**Cấu trúc KTV theo level:**

| Level | Tài khoản | Vai trò |
|---|---|---|
| **L1** | `nguyenvana`, `tranthib`, `levanc` | Nhận ticket đầu tiên theo round-robin, xử lý sự cố thông thường |
| **L2** | `phamvand` | Nhận escalation từ L1 khi quá SLA hoặc không xử lý được |
| **L3** | `hoangvane` | Nhận escalation từ L2, xử lý sự cố phức tạp/nghiêm trọng |


#### 3.5. Cài Zabbix Agent 2 trên Windows Server 2022

Tải Zabbix Agent 2 cho Windows tại [zabbix.com/download](https://www.zabbix.com/download):

- OS: **Windows**
- Package: **Agent 2**

Chạy installer, điền:

| Field | Value |
|---|---|
| **Host name** | `DC` |
| **Zabbix server IP/DNS** | `10.10.200.11` |
| **Agent listen port** | `10050` |
| **Server or Proxy for active checks** | `10.10.200.11` |

<img src="/assets/img/2026-04-23-automated-itil-workflow/05.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/06.png"/>

```powershell
# Kiểm tra service
Get-Service -Name "Zabbix Agent 2"

# Mở firewall
New-NetFirewallRule -DisplayName "Zabbix Agent" -Direction Inbound `
    -Protocol TCP -LocalPort 10050 -Action Allow
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/07.png"/>

---

### 4. Chuẩn bị VM itil-stack

Trên VM `itil-stack` (Ubuntu Server 22.04):

```bash
# Cập nhật hệ thống
sudo apt update && sudo apt upgrade -y

# Cài các gói cơ bản
sudo apt install -y curl wget gnupg2 apt-transport-https \
    software-properties-common ca-certificates lsb-release

# Cấu hình DNS — chỉ cần truyền về DC, DC tự forward external query ra internet
sudo tee /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=10.10.200.12
FallbackDNS=8.8.8.8
Domains=anhlx.lab
EOF

sudo systemctl restart systemd-resolved

# Kiểm tra resolve domain
nslookup dc.anhlx.lab
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/08.png"/>

---

### 5. Setup PostgreSQL 16

```bash
# Thêm repo PostgreSQL 16
sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
    > /etc/apt/sources.list.d/pgdg.list'

wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | \
    sudo tee /etc/apt/trusted.gpg.d/pgdg.asc > /dev/null

sudo apt update
sudo apt install -y postgresql-16 postgresql-client-16

sudo systemctl enable postgresql
sudo systemctl start postgresql
sudo systemctl status postgresql
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/09.png"/>

**Tạo database cho Zabbix:**

```bash
sudo -u postgres psql << 'EOF'
CREATE USER zabbix WITH PASSWORD 'zabbix_pass_2026';
CREATE DATABASE zabbix
    OWNER zabbix
    ENCODING 'UTF8'
    LC_COLLATE='en_US.utf-8'
    LC_CTYPE='en_US.utf-8'
    TEMPLATE template0;
GRANT ALL PRIVILEGES ON DATABASE zabbix TO zabbix;
\l
EOF
```

**Tạo database cho SDP:**

```bash
sudo -u postgres psql << 'EOF'
CREATE USER sdpadmin WITH PASSWORD 'sdp_admin_2026';
CREATE DATABASE servicedesk OWNER sdpadmin ENCODING 'UTF8';
GRANT ALL PRIVILEGES ON DATABASE servicedesk TO sdpadmin;
\l
EOF
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/10.png"/>

---

### 6. Setup Zabbix 7.4

#### 6.1. Cài đặt Zabbix Server, Frontend, Agent

```bash
# Thêm Zabbix repository 7.4
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu22.04_all.deb

sudo dpkg -i zabbix-release_latest_7.4+ubuntu22.04_all.deb
sudo apt update

# Cài Zabbix server, frontend (nginx), agent2
# php8.1-pgsql bắt buộc — không có thì wizard chỉ hiện MySQL
sudo apt install -y zabbix-server-pgsql zabbix-frontend-php \
    zabbix-nginx-conf zabbix-sql-scripts zabbix-agent2 php8.1-pgsql

# Restart php-fpm để load extension pgsql mới cài
sudo systemctl restart php8.1-fpm
```

#### 6.2. Import schema và cấu hình database

```bash
# Import schema Zabbix vào PostgreSQL
# Dùng PGPASSWORD + -h localhost để tránh lỗi peer auth
zcat /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz | \
    PGPASSWORD=zabbix_pass_2026 psql -h localhost -U zabbix zabbix

# Xác nhận schema đã import (phải thấy bảng dbversion)
PGPASSWORD=zabbix_pass_2026 psql -h localhost -U zabbix zabbix \
    -c "SELECT mandatory FROM dbversion;"
```

#### 6.3. Cấu hình Zabbix Server

```bash
sudo sed -i 's/^# DBHost=.*/DBHost=localhost/' /etc/zabbix/zabbix_server.conf
sudo sed -i 's/^DBName=.*/DBName=zabbix/' /etc/zabbix/zabbix_server.conf
sudo sed -i 's/^# DBUser=.*/DBUser=zabbix/' /etc/zabbix/zabbix_server.conf
sudo sed -i 's/^# DBPassword=.*/DBPassword=zabbix_pass_2026/' /etc/zabbix/zabbix_server.conf

# Xác nhận
grep -E '^DB' /etc/zabbix/zabbix_server.conf
```

**Cấu hình Zabbix Agent2 trên itil-stack** — cho phép zabbix-server passive check chính mình:

```bash
# Thêm IP server vào Server= và ServerActive=
sudo sed -i 's/^Server=.*/Server=127.0.0.1,10.10.200.11/' /etc/zabbix/zabbix_agent2.conf
sudo sed -i 's/^ServerActive=.*/ServerActive=127.0.0.1,10.10.200.11/' /etc/zabbix/zabbix_agent2.conf

# Đặt Hostname khớp với tên host trong Zabbix web
# Mặc định là "Zabbix server" — phải đổi thành "itil-stack" để active checks register đúng host
sudo sed -i 's/^Hostname=.*/Hostname=itil-stack/' /etc/zabbix/zabbix_agent2.conf

# Xác nhận
grep -E '^Server|^Hostname' /etc/zabbix/zabbix_agent2.conf
```


```bash
sudo systemctl enable zabbix-server zabbix-agent2
sudo systemctl start zabbix-server zabbix-agent2
sudo systemctl status zabbix-server
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/11.png"/>

#### 6.4. Cấu hình Nginx và PHP cho Zabbix Frontend

```bash
# Bỏ comment và đặt listen port + server_name
sudo sed -i 's/#\s*listen\s*8080;/listen          8080;/' /etc/zabbix/nginx.conf
sudo sed -i 's/#\s*server_name\s*example.com;/server_name     10.10.200.11;/' /etc/zabbix/nginx.conf

# Đặt timezone PHP
sudo sed -i 's|.*php_value\[date.timezone\].*|php_value[date.timezone] = Asia/Ho_Chi_Minh|' /etc/zabbix/php-fpm.conf

# Xác nhận
grep -E 'listen|server_name' /etc/zabbix/nginx.conf
grep 'date.timezone' /etc/zabbix/php-fpm.conf
```

```bash
sudo systemctl enable nginx php8.1-fpm
sudo systemctl restart nginx php8.1-fpm zabbix-server
```

#### 6.5. Cấu hình Zabbix Web lần đầu

Truy cập `http://10.10.200.11:8080`, làm theo wizard:

<img src="/assets/img/2026-04-23-automated-itil-workflow/12.png"/>


| Step | Giá trị |
|---|---|
| Database type | **PostgreSQL** |
| Database host | `localhost` |
| Database port | `5432` |
| Database name | `zabbix` |
| Database schema | _(để trống — bảng nằm trong schema `public`)_ |
| User | `zabbix` |
| Password | `zabbix_pass_2026` |
| Database TLS encryption | ☐ _(không cần — kết nối localhost)_ |

<img src="/assets/img/2026-04-23-automated-itil-workflow/13.png"/>
<img src="/assets/img/2026-04-23-automated-itil-workflow/14.png"/>
<img src="/assets/img/2026-04-23-automated-itil-workflow/15.png"/>


Đăng nhập `Admin / zabbix`, đổi mật khẩu ngay: **User Settings → Change Password**.

<img src="/assets/img/2026-04-23-automated-itil-workflow/16.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/17.png"/>

**Sửa host "Zabbix server" mặc định** — **Data Collection → Hosts → Zabbix server → Interfaces → Agent:**
- Loại: **IP**
- IP: `10.10.200.11`
- Port: `10050`

<img src="/assets/img/2026-04-23-automated-itil-workflow/18.png"/>

#### 6.6. Add hosts vào Zabbix

**Host 1: dc** — **Data Collection → Hosts → Create host**

| Field | Value |
|---|---|
| Host name | `DC` |
| Groups | `Windows servers` |
| Interfaces > Agent | IP: `10.10.200.12`, Port: `10050` |
| Templates | `Windows by Zabbix agent active` |

> Hostname phải khớp chính xác với giá trị đã khai báo khi cài agent trên Windows (`DC`).

<img src="/assets/img/2026-04-23-automated-itil-workflow/19.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/20.png"/>

**Host 2: itil-stack (self-monitoring)**

> Mô phỏng add host linux cần monitor

| Field | Value |
|---|---|
| Host name | `itil-stack` |
| Groups | `Linux servers` |
| Interfaces > Agent | IP: `10.10.200.11`, Port: `10050` |
| Templates | `Linux by Zabbix agent active` |

<img src="/assets/img/2026-04-23-automated-itil-workflow/21.png"/>


Đợi vài phút để hosts chuyển sang màu xanh (Available).

#### 6.7. Cấu hình Zabbix LDAP / Active Directory

**Users → Authentication → LDAP settings**

| Field | Value |
|---|---|
| **Enable LDAP authentication** | ☑ |
| **LDAP host** | `ldap://10.10.200.12` |
| **Port** | `389` |
| **Base DN** | `OU=ITIL-Lab,DC=anhlx,DC=lab` |
| **Search attribute** | `sAMAccountName` |
| **Bind DN** | `CN=svc-ldap,OU=ServiceAccounts,OU=ITIL-Lab,DC=anhlx,DC=lab` |
| **Bind password** | `Svc@Ldap2026` |

Click **Test authentication** — nhập `nguyenvana / Ktv@Abc2026` → hiện **"Login name attribute value obtained": Nguyen Van A** là thành công.

Click **Update** để lưu.

<img src="/assets/img/2026-04-23-automated-itil-workflow/22.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/23.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/24.png"/>

> **Lưu ý:** Người dùng AD đăng nhập lần đầu vào Zabbix sẽ tự động được tạo account với role mặc định **User**. Để cấp quyền admin, vào **Users → Users → <tên user> → Role → chọn Zabbix Admin hoặc Super Admin**.

**Tạo sẵn user LDAP cho KTV (cấp quyền trước):**

> KTV chỉ cần **xem** monitoring — dùng role **Zabbix User** (roleid=1), không phải Admin.  
> User group `Zabbix administrators` cho phép truy cập toàn bộ host data (đủ dùng cho lab).

Dùng Zabbix API để tạo 5 user cùng lúc thay vì click từng cái:

```bash
ZBX_URL="http://10.10.200.11:8080/api_jsonrpc.php"
ZBX_PASS="<admin-password>"   # password của tài khoản Admin Zabbix

# Lấy auth token
# Zabbix 7.x: user.login trả về token, các request sau dùng "Authorization: Bearer"
TOKEN=$(curl -s -X POST "$ZBX_URL" \
  -H "Content-Type: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"method\":\"user.login\",\"params\":{\"username\":\"Admin\",\"password\":\"$ZBX_PASS\"},\"id\":1}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['result'])")

echo "Token: $TOKEN"

# Lấy usrgrpid của group "Zabbix administrators"
# Zabbix 7.x: dùng header "Authorization: Bearer <token>" thay vì field "auth"
GRPID=$(curl -s -X POST "$ZBX_URL" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"jsonrpc\":\"2.0\",\"method\":\"usergroup.get\",\"params\":{\"filter\":{\"name\":\"Zabbix administrators\"},\"output\":[\"usrgrpid\"]},\"id\":1}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['result'][0]['usrgrpid'])")

echo "Group ID: $GRPID"

# Tạo 5 KTV — role Zabbix User (roleid=1), passwd dummy (LDAP sẽ override)
for data in "nguyenvana:Nguyen Van A" "tranthib:Tran Thi B" "levanc:Le Van C" \
            "phamvand:Pham Van D" "hoangvane:Hoang Van E"; do
  sam="${data%%:*}"
  name="${data#*:}"
  curl -s -X POST "$ZBX_URL" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    -d "{\"jsonrpc\":\"2.0\",\"method\":\"user.create\",\"params\":{\"username\":\"$sam\",\"name\":\"$name\",\"passwd\":\"Zbx_Dummy@2026\",\"usrgrps\":[{\"usrgrpid\":\"$GRPID\"}],\"roleid\":\"1\"},\"id\":1}" \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print('$sam:', 'OK userid=' + str(r['result']['userids'][0]) if 'result' in r else r['error']['data'])"
done
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/25.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/26.png"/>

---

### 7. Setup Grafana

#### 7.1. Cài đặt Grafana

```bash
# Thêm Grafana repository
wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key

echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" | \
    sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt update
sudo apt install -y grafana

# Cài Zabbix plugin (Grafana 10+: dùng "grafana cli" thay vì "grafana-cli")
sudo grafana cli plugins install alexanderzobnin-zabbix-app

sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/27.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/28.png"/>

Truy cập `http://10.10.200.11:3000`, đăng nhập `admin / admin`, đổi mật khẩu.

<img src="/assets/img/2026-04-23-automated-itil-workflow/29.png"/>


**Kích hoạt plugin:** Administration → Plugins → tìm **Zabbix** → Enable.

<img src="/assets/img/2026-04-23-automated-itil-workflow/30.png"/>

#### 7.2. Cấu hình Grafana LDAP / Active Directory

```bash
sudo tee /etc/grafana/ldap.toml << 'EOF'
[[servers]]
host = "10.10.200.12"
port = 389
use_ssl = false
start_tls = false
bind_dn = "CN=svc-ldap,OU=ServiceAccounts,OU=ITIL-Lab,DC=anhlx,DC=lab"
bind_password = "Svc@Ldap2026"
search_base_dns = ["OU=ITIL-Lab,DC=anhlx,DC=lab"]
search_filter = "(sAMAccountName=%s)"

[servers.attributes]
name = "displayName"
username = "sAMAccountName"
member_of = "memberOf"
email = "mail"

[[servers.group_mappings]]
group_dn = "CN=Technicians,OU=Groups,OU=ITIL-Lab,DC=anhlx,DC=lab"
org_role = "Editor"

[[servers.group_mappings]]
group_dn = "*"
org_role = "Viewer"
EOF
```

Bật LDAP trong `grafana.ini`:

```bash
sudo sed -i 's|^;\?\[auth.ldap\]|[auth.ldap]|' /etc/grafana/grafana.ini
sudo sed -i '/^\[auth.ldap\]/,/^\[/ s|^;\?enabled\s*=.*|enabled = true|' /etc/grafana/grafana.ini
sudo sed -i '/^\[auth.ldap\]/,/^\[/ s|^;\?config_file\s*=.*|config_file = /etc/grafana/ldap.toml|' /etc/grafana/grafana.ini
sudo sed -i '/^\[auth.ldap\]/,/^\[/ s|^;\?allow_sign_up\s*=.*|allow_sign_up = true|' /etc/grafana/grafana.ini

# Xác nhận
grep -A3 '\[auth.ldap\]' /etc/grafana/grafana.ini
```

```bash
sudo systemctl restart grafana-server
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/31.png"/>

**Kiểm tra:** Đăng nhập bằng `nguyenvana / Ktv@Abc2026` → Grafana tự tạo account với role **Editor** (theo group mapping Technicians).

<img src="/assets/img/2026-04-23-automated-itil-workflow/32.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/33.png"/>

#### 7.3. Kết nối Zabbix datasource và import dashboard

**Configuration → Data Sources → Add data source → Zabbix**

| Field | Value |
|---|---|
| URL | `http://10.10.200.11:8080/api_jsonrpc.php` |
| Username | `Admin` |
| Password | `<zabbix-password>` |

Click **Save & Test** → hiển thị "Zabbix API version: 7.4.x".


<img src="/assets/img/2026-04-23-automated-itil-workflow/34.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/35.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/36.png"/>


**Import dashboard:** Dashboards → New → Import → nhập ID `5363` → Load → Import.

<img src="/assets/img/2026-04-23-automated-itil-workflow/37.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/38.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/39.png"/>

---

### 8. Setup ServiceDesk Plus 14

#### 8.1. Cài đặt SDP

```bash
# Copy file installer lên server 
# scp "D:\softs\ManageEngine_ServiceDesk_Plus.bin" ubuntu@10.10.200.11:/tmp/

# Cài dependencies
sudo apt install -y libc6 libstdc++6 fontconfig libxi6 libxrender1 libxtst6

chmod +x /tmp/ManageEngine_ServiceDesk_Plus.bin

# Chạy installer (interactive console)
# sudo /tmp/ManageEngine_ServiceDesk_Plus.bin -i console

sudo env -i \
  HOME=/root USER=root LOGNAME=root \
  PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 \
  /tmp/ManageEngine_ServiceDesk_Plus.bin -i console
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/40.png"/>

| Prompt | Giá trị |
|---|---|
| Installation directory | `/opt/ManageEngine/ServiceDesk` |
| Web server port | `8400` |
| Start as service | `Yes` |

<img src="/assets/img/2026-04-23-automated-itil-workflow/41.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/42.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/43.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/44.png"/>

**Chuyển SDP sang dùng PostgreSQL 16:**

```bash
# Backup config gốc
cp /opt/ManageEngine/ServiceDesk/conf/database_params.conf \
   /opt/ManageEngine/ServiceDesk/conf/database_params.conf.bak

cat > /opt/ManageEngine/ServiceDesk/conf/database_params.conf << 'DBEOF'
drivername=org.postgresql.Driver
url=jdbc:postgresql://localhost:5432/servicedesk?charSet=UTF-8&loginTimeout=30&socketTimeout=1200&connectTimeout=20&sslmode=disable
username=sdpadmin
password=sdp_admin_2026
rodatasource.drivername=org.postgresql.Driver
rodatasource.url=jdbc:postgresql://localhost:5432/servicedesk?charSet=UTF-8&loginTimeout=5&socketTimeout=1200&connectTimeout=5&sslmode=disable
rodatasource.username=sdpadmin
rodatasource.password=sdp_admin_2026
rodatasource.minsize=1
rodatasource.maxsize=20
minsize=5
maxsize=40
blockingtimeout=5
transaction_isolation=TRANSACTION_READ_COMMITTED
rodatasource.transaction_isolation=TRANSACTION_READ_COMMITTED
validconnectionsql=select 1 as test ;
exceptionsorterclassname=com.adventnet.db.adapter.postgres.PostgresExceptionSorter
rodatasource.exceptionsorterclassname=com.adventnet.db.adapter.postgres.PostgresExceptionSorter
trace.connections=true
DBEOF
```

```bash
# Khởi động SDP
cd /opt/ManageEngine/ServiceDesk/bin
sudo ./run.sh start

<img src="/assets/img/2026-04-23-automated-itil-workflow/45.png"/>

# Theo dõi log — chờ "Server startup in ... ms"
tail -f /opt/ManageEngine/ServiceDesk/logs/serverOut.txt

```


**Tạo systemd service:**

```bash
sudo tee /etc/systemd/system/sdp.service << 'EOF'
[Unit]
Description=ManageEngine ServiceDesk Plus
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=oneshot
RemainAfterExit=yes
User=root
WorkingDirectory=/opt/ManageEngine/ServiceDesk/bin
ExecStart=/bin/bash -c 'cd /opt/ManageEngine/ServiceDesk/bin && ./run.sh start &'
ExecStop=/opt/ManageEngine/ServiceDesk/bin/run.sh stop
TimeoutStartSec=10
KillMode=none

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable sdp
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/46.png"/>

#### 8.2. Đăng nhập lần đầu

Truy cập `https://10.10.200.11:8400`, đăng nhập `administrator / administrator`, đổi mật khẩu theo yêu cầu.

<img src="/assets/img/2026-04-23-automated-itil-workflow/47.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/48.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/49.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/50.png"/>

#### 8.3. Tích hợp LDAP / Active Directory

> **Lưu ý:** SDP có 2 tab riêng biệt:
> - **Active Directory**: dùng khi server SDP join domain (Windows). `itil-stack` là Ubuntu không join domain → **không dùng tab này**.
> - **LDAP**: kết nối AD qua LDAP protocol (port 389) — dùng cho lab này.

**Bước 1 — Bật LDAP Authentication:**

**Admin → Users & Permission → LDAP**

Tích **Enable LDAP Authentication** → **Save**

<img src="/assets/img/2026-04-23-automated-itil-workflow/51.png"/>

**Bước 2 — Add New Domain Controller:**

Cuộn xuống phần **Add New Domain Controller**, điền:

| Field | Value |
|---|---|
| **Domain Controller** | `ldap://10.10.200.12:389` _(điền URI đầy đủ, không chỉ IP)_ |
| **Username** | `svc-ldap@anhlx.lab` _(dùng UPN)_ |
| **Password** | `Svc@Ldap2026` |
| **Base DN** | `OU=ITIL-Lab,DC=anhlx,DC=lab` |
| **Search Filter** | `(&(objectClass=user)(objectCategory=person))` |
| **LDAP Server Type** | `Microsoft Active Directory` |
| **Login Attribute Label** | `sAMAccountName` |
| **Mail Attribute Label** | `mail` |
| **Distinguished Name Attribute Label** | `distinguishedName` |

Click **Save** → domain xuất hiện trong danh sách **Domain Controllers** phía dưới.

<img src="/assets/img/2026-04-23-automated-itil-workflow/52.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/53.png"/>

#### 8.4. Tạo kỹ thuật viên từ AD, cấu hình Round-Robin và Escalation

**Xóa 5 technician mẫu:**

**Admin → Users & Permission → Technicians** → chọn tất cả → **Actions → Delete**.

**Tạo 5 KTV — import qua CSV (SDP trên Linux không hỗ trợ "Import from AD"):**

> Nút **"Import from AD"** trong phần Users chỉ hoạt động khi SDP chạy trên Windows (dùng Windows API). Trên Ubuntu, dùng **Import Wizard → CSV**.

Tạo file `technicians.csv` trên máy tính:

```csv
Name,Login Name,Primary Email,Password,Description
Nguyen Van A,nguyenvana,nguyenvana@anhlx.lab,Ktv@Abc2026,L1 Technician
Tran Thi B,tranthib,tranthib@anhlx.lab,Ktv@Abc2026,L1 Technician
Le Van C,levanc,levanc@anhlx.lab,Ktv@Abc2026,L1 Technician
Pham Van D,phamvand,phamvand@anhlx.lab,Ktv@Abc2026,L2 Technician
Hoang Van E,hoangvane,hoangvane@anhlx.lab,Ktv@Abc2026,L3 Technician
```

> **Lưu ý header CSV:** field email trong SDP là `Primary Email` (không phải `Email`) — phải khớp chính xác để auto-map. Password đặt tạm `Ktv@Abc2026` — LDAP override khi đăng nhập bằng password AD.


1. **File format:** chọn `CSV` → **Next**
2. Upload file `technicians.csv`
3. **Map Fields** — kiểm tra đủ 5 field:

| SDP Field | CSV Column |
|---|---|
| Name * | `Name (col 1)` |
| Login Name * | `Login Name (col 2)` |
| Primary Email | `Primary Email (col 3)` |
| Password | `Password (col 4)` |
| Description | `Description (col 5)` |

4. Click **Import**

Sau khi import, user được tạo với role **User** — cần đổi thành **Technician**:

**Admin → Users & Permission → Users** → tick chọn các user cần đổi → **Actions → Change as Technician**

> Hoặc ngay trên trang Technicians, dùng dropdown **"Change as Technician"** để chọn lần lượt từng user.

<img src="/assets/img/2026-04-23-automated-itil-workflow/54.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/55.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/56.png"/>

**Cấu hình Round-Robin Auto Assign (chỉ L1):**

**Admin → Automation → Technician Auto Assign**

| Field | Giá trị |
|---|---|
| **Enabled** | ☑ |
| **Select Tech Auto Assign model** | ● Round Robin |
| **Execute when a request is** | ● Created |
| **Apply Tech Auto Assign while creating requests to** | ● All Requests |
| **Include or Exclude Technicians** | ☑ Include: `Nguyen Van A`, `Tran Thi B`, `Le Van C` |

> Chỉ include 3 KTV L1. `Pham Van D` (L2) và `Hoang Van E` (L3) không nằm trong round-robin ban đầu — chỉ nhận ticket khi được escalate.

Click **Save**.

<img src="/assets/img/2026-04-23-automated-itil-workflow/57.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/58.png"/>

**Cấu hình Escalation Rules:**

SDP có sẵn 4 SLA mặc định. Escalation được cấu hình **ngay trong form edit từng SLA** (không phải tab riêng).

**Admin → Automation → Service Level Agreements** → click icon bút chì (✏) hoặc tên **High SLA**


<img src="/assets/img/2026-04-23-automated-itil-workflow/59.png"/>

Trong form **Edit SLA - High SLA**, có 2 section escalation riêng biệt:

<img src="/assets/img/2026-04-23-automated-itil-workflow/60.png"/>

**Section "If response time is elapsed then escalate:"**

- Tick ☑ **Enable Level 1 Escalation**
- **Escalate to:** `Pham Van D` _(L2)_
- **Escalate After:** `0` Days `4` Hours `0` Minutes

> ⚠️ Escalation target phải là L2/L3 — **không** chọn L1 technician (nguyenvana, tranthib, levanc) vì đó là nhóm ban đầu nhận ticket.

<img src="/assets/img/2026-04-23-automated-itil-workflow/61.png"/>

Click **Save**. Lặp lại tương tự cho **Medium SLA**, **Normal SLA**, **Low SLA**.

> **Lưu ý:** KTV L1 có thể chủ động chuyển ticket cho L2/L3 bằng cách vào ticket → **Assign** → chọn `Pham Van D` hoặc `Hoang Van E`.

#### 8.5. Lấy API Token — cập nhật macro Zabbix

> Trong môi trường lab tôi dùng account Administrator , thực tế cần phân quyền hợp lý .

Click vào **avatar** (góc trên phải) → chọn **Generate Authtoken**.

<img src="/assets/img/2026-04-23-automated-itil-workflow/62.png"/>

Trên popup, chọn tab **Generate Authtoken** → chọn **Never expires** → click **Generate** → copy Authtoken hiển thị trong ô vàng.

<img src="/assets/img/2026-04-23-automated-itil-workflow/63.png"/>

Copy authtoken, sau đó vào Zabbix:

**Administration → Macros**

| Macro | Value |
|---|---|
| `{$SDP_API_TOKEN}` | `<authtoken-vừa-copy>` |
| `{$SDP_URL}` | `https://10.10.200.11:8400` |
| `{$TG_BOT_TOKEN}` | `<bot-token-từ-BotFather>` |
| `{$TG_CHAT_ID}` | `<chat-id-group-hoặc-channel>` |

<img src="/assets/img/2026-04-23-automated-itil-workflow/64.png"/>

#### 8.6. Tạo Webhook Media Type gọi SDP API

**Alerts → Media types → Create media type**

- **Name:** `ServiceDesk Plus`
- **Type:** `Webhook`

Xóa các parameter mặc định, thêm mới:

| Name | Value |
|---|---|
| `sdp_url` | `{$SDP_URL}` |
| `sdp_token` | `{$SDP_API_TOKEN}` |
| `event_id` | `{EVENT.ID}` |
| `host` | `{HOST.NAME}` |
| `trigger_name` | `{TRIGGER.NAME}` |
| `trigger_severity` | `{TRIGGER.SEVERITY}` |
| `event_status` | `{EVENT.STATUS}` |
| `event_time` | `{EVENT.TIME} {EVENT.DATE}` |

**Script:**

```javascript
var params = JSON.parse(value);

// Round-robin chỉ trong nhóm L1 (3 KTV)
var l1Technicians = ['Nguyen Van A', 'Tran Thi B', 'Le Van C'];
var techIndex = parseInt(params.event_id) % 3;
var assignedTech = l1Technicians[techIndex];
// L2: Pham Van D — nhận escalation từ SDP SLA sau 4h
// L3: Hoang Van E — nhận escalation từ SDP SLA sau 8h

var priorityMap = {
    'Not classified': 'Low',
    'Information': 'Low',
    'Warning': 'Medium',
    'Average': 'Medium',
    'High': 'High',
    'Disaster': 'Urgent'
};
var priority = priorityMap[params.trigger_severity] || 'Medium';

var subject = '[' + params.trigger_severity + '] ' + params.trigger_name + ' - ' + params.host;
var description = 'Host: ' + params.host + '\n'
    + 'Trigger: ' + params.trigger_name + '\n'
    + 'Severity: ' + params.trigger_severity + '\n'
    + 'Status: ' + params.event_status + '\n'
    + 'Time: ' + params.event_time + '\n'
    + 'Event ID: ' + params.event_id;

var inputData = JSON.stringify({
    "request": {
        "subject": subject,
        "description": description,
        "priority": {"name": priority},
        "requester": {"name": "administrator"},
        "technician": {"name": assignedTech}
    }
});

var req = new HttpRequest();
req.addHeader('Content-Type: application/x-www-form-urlencoded');
req.addHeader('authtoken: ' + params.sdp_token);

var url = params.sdp_url + '/api/v3/requests';
var postData = 'input_data=' + encodeURIComponent(inputData);

var response = req.post(url, postData);
Zabbix.log(4, 'SDP response: ' + response);

var respJson = JSON.parse(response);
if (respJson.response_status && respJson.response_status.status_code === 2000) {
    return 'Ticket created: ' + respJson.request.id + ' -> Assigned: ' + assignedTech;
} else {
    throw 'SDP API error: ' + response;
}
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/65.png"/>

**Thêm Message templates cho ServiceDesk Plus:**

> ⚠️ Zabbix yêu cầu mỗi media type phải có ít nhất 1 **Message template** — dù webhook script xử lý qua `value`/params, không dùng nội dung template. Thiếu template → lỗi **"No message defined for media type"** → action Failed.

Trong form media type **ServiceDesk Plus** → tab **Message templates** → **Add**:

| Message type | Subject | Message |
|---|---|---|
| `Problem` | `{TRIGGER.NAME}` | `{TRIGGER.NAME}` |
| `Problem recovery` | `{TRIGGER.NAME}` | `{TRIGGER.NAME}` |

Click **Add** sau mỗi dòng → **Update** để lưu.

#### 8.7. Tạo Action tự động tạo ticket

**Alerts → Actions → Trigger actions → Create action**

**Action tab:**

| Field | Value |
|---|---|
| Name | `Auto Create SDP Ticket` |
| Conditions | Trigger severity >= Warning |

<img src="/assets/img/2026-04-23-automated-itil-workflow/66.png"/>


**Operations tab → Add operation:**

| Field | Value |
|---|---|
| Operation type | Send message |
| Send to users | `Admin (Zabbix Administrator)` |
| Send to media type | `ServiceDesk Plus` |

<img src="/assets/img/2026-04-23-automated-itil-workflow/67.png"/>

**Recovery operations:** thêm Send message để thông báo RESOLVED.

**Gán media "ServiceDesk Plus" cho user Admin:**

**Users → Users → Admin → tab Media → Add**

| Field | Value |
|---|---|
| Type | `ServiceDesk Plus` |
| Send to | `1` |
| When active | `1-7,00:00-24:00` |
| Use if severity | Tích hết |

<img src="/assets/img/2026-04-23-automated-itil-workflow/68.png"/>

**Test webhook thủ công:**

**Alerts → Media types → ServiceDesk Plus → Test**

Popup hiện ra các field — điền giá trị thật:


| Field | Giá trị điền vào |
|---|---|
| `event_id` | `1` |
| `event_status` | `PROBLEM` |
| `event_time` | `12:00:00 2026-04-23` |
| `host` | `itil-stack` |
| `sdp_token` | `<authtoken-thật-vừa-copy-từ-SDP>` |
| `sdp_url` | `https://10.10.200.11:8400` |
| `trigger_name` | `Disk space is critically low` |
| `trigger_severity` | `High` |

Click **Test**.

Kết quả hiện trong ô **Response**: `Ticket created: <ID> -> Assigned: Nguyen Van A`

<img src="/assets/img/2026-04-23-automated-itil-workflow/71.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/72.png"/>

---

#### 8.8. Tích hợp Telegram Alert

**Tạo Telegram Bot:**

<img src="/assets/img/2026-04-23-automated-itil-workflow/73.png"/>

**Cập nhật macro Zabbix** tại **Administration → General → Macros** (đã thêm `{$TG_BOT_TOKEN}` và `{$TG_CHAT_ID}` ở mục 8.5).

**Tạo Media Type Telegram:**

**Alerts → Media types → Create media type**

- **Name:** `Telegram`
- **Type:** `Webhook`

Thêm parameter:

| Name | Value |
|---|---|
| `bot_token` | `{$TG_BOT_TOKEN}` |
| `chat_id` | `{$TG_CHAT_ID}` |
| `host` | `{HOST.NAME}` |
| `trigger_name` | `{TRIGGER.NAME}` |
| `trigger_severity` | `{TRIGGER.SEVERITY}` |
| `event_status` | `{EVENT.STATUS}` |
| `event_time` | `{EVENT.TIME} {EVENT.DATE}` |
| `event_id` | `{EVENT.ID}` |

**Script:**

```javascript
var params = JSON.parse(value);

var severityEmoji = {
    'Not classified': '⚪',
    'Information':    '🔵',
    'Warning':        '🟡',
    'Average':        '🟠',
    'High':           '🔴',
    'Disaster':       '💀'
};

var statusEmoji = params.event_status === 'RESOLVED' ? '✅ RESOLVED' : '🔥 PROBLEM';
var emoji = severityEmoji[params.trigger_severity] || '⚪';

var text = emoji + ' *' + statusEmoji + '*\n'
    + '*Host:* ' + params.host + '\n'
    + '*Trigger:* ' + params.trigger_name + '\n'
    + '*Severity:* ' + params.trigger_severity + '\n'
    + '*Time:* ' + params.event_time + '\n'
    + '*Event ID:* ' + params.event_id;

var req = new HttpRequest();
req.addHeader('Content-Type: application/json');

var url = 'https://api.telegram.org/bot' + params.bot_token + '/sendMessage';
var body = JSON.stringify({
    chat_id: params.chat_id,
    text: text,
    parse_mode: 'Markdown'
});

var response = req.post(url, body);
Zabbix.log(4, 'Telegram response: ' + response);

var resp = JSON.parse(response);
if (resp.ok) {
    return 'Telegram sent: message_id=' + resp.result.message_id;
} else {
    throw 'Telegram API error: ' + response;
}
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/74.png"/>

**Thêm Message templates cho Telegram:**

Trong form media type **Telegram** → tab **Message templates** → **Add**:

| Message type | Subject | Message |
|---|---|---|
| `Problem` | `{TRIGGER.NAME}` | `{TRIGGER.NAME}` |
| `Problem recovery` | `{TRIGGER.NAME}` | `{TRIGGER.NAME}` |

Click **Add** sau mỗi dòng → **Update** để lưu.

**Gán media "Telegram" cho user Admin:**

**Users → Users → Admin → tab Media → Add**

| Field | Value |
|---|---|
| Type | `Telegram` |
| Send to | `1` |
| When active | `1-7,00:00-24:00` |
| Use if severity | Tích hết |

<img src="/assets/img/2026-04-23-automated-itil-workflow/75.png"/>

**Thêm Telegram vào Action đã tạo:**

**Alerts → Actions → Auto Create SDP Ticket → Operations → Add operation:**

| Field | Value |
|---|---|
| Operation type | Send message |
| Send to users | Admin |
| Send only to | `Telegram` |

<img src="/assets/img/2026-04-23-automated-itil-workflow/76.png"/>

**Recovery operations:** thêm operation Telegram tương tự để nhận thông báo RESOLVED.

**Test thủ công:**

**Alerts → Media types → Telegram → Test**

> ⚠️ Popup test **không tự resolve global macro** — phải nhập giá trị thật vào `bot_token` và `chat_id`.

| Field | Giá trị điền vào |
|---|---|
| `bot_token` | `<bot-token-thật-từ-BotFather>` |
| `chat_id` | `<chat_id-thật-của-group>` _(số âm, ví dụ `-1001234567890`)_ |
| `event_id` | `1` |
| `event_status` | `PROBLEM` |
| `event_time` | `12:00:00 2026-04-23` |
| `host` | `itil-stack` |
| `trigger_name` | `Disk space is critically low` |
| `trigger_severity` | `High` |

Click **Test** → kiểm tra Telegram group nhận được tin nhắn.

<img src="/assets/img/2026-04-23-automated-itil-workflow/77.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/78.png"/>

---

### 9. Test Automated Workflow

#### 9.1. Test: Disk Full

**Trên itil-stack (Ubuntu):**

```bash
# Kiểm tra dung lượng trống trước
df -hT /

# Tính count cần thiết để đạt >90% disk:
# count = (total_GB * 0.90 - used_GB) * 1024
# Ví dụ disk 194G, used 14G: count = (194*0.9 - 14)*1024 = 160,768 → dùng 163000 cho chắc
dd if=/dev/zero of=/tmp/fill_disk_test bs=1M count=163000 status=progress

# Kiểm tra — Use% phải >90%
df -hT /
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/79.png"/>

**Trên dc (Windows):**

```powershell
# Tạo file lớn chiếm dung lượng C:
# Tính: disk 199GB, used ~15GB → cần fill ~164GB để vượt 90%
# (total_GB * 0.90 - used_GB) = (199*0.9 - 15) ≈ 164GB → dùng 170GB cho chắc
$path = "C:\fill_disk_test.tmp"
$fs = [System.IO.File]::Create($path)
$fs.SetLength(170GB)
$fs.Close()

# Kiểm tra — Used% phải >90%
Get-PSDrive C
```

<img src="/assets/img/2026-04-23-automated-itil-workflow/80.png"/>

**Kết quả mong đợi (1–2 phút):**

1. ✅ Zabbix trigger `Disk space is critically low` kích hoạt
2. ✅ Webhook gọi SDP API → ticket mới, assign round-robin cho KTV AD
3. ✅ Telegram gửi tin nhắn 🔥 `PROBLEM | itil-stack | Disk space is critically low`
4. ✅ Grafana panel cập nhật active problems

<img src="/assets/img/2026-04-23-automated-itil-workflow/81.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/82.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/83.png"/>

<img src="/assets/img/2026-04-23-automated-itil-workflow/84.png"/>

**Dọn dẹp:**

```bash
rm /tmp/fill_disk_test
```

```powershell
Remove-Item "C:\fill_disk_test.tmp"
```

---

### 10. Kết quả


| Sự kiện | Zabbix | SDP | Telegram | Grafana |
|---|---|---|---|---|
| Disk > 90% | Alert PROBLEM | Ticket tạo tự động | 🔥 PROBLEM | Panel cập nhật |

**Escalation logic:**

```
[Ticket tạo]  →  Round-robin L1:
                  event_id mod 3 = 0  →  Le Van C
                  event_id mod 3 = 1  →  Nguyen Van A
                  event_id mod 3 = 2  →  Tran Thi B

[Sau 4 giờ chưa xử lý]  →  Escalation L2: Pham Van D

[Sau 8 giờ chưa xử lý]  →  Escalation L3: Hoang Van E
```

**Xác thực tập trung qua AD:**

| Hệ thống | Phương thức | Tài khoản AD |
|---|---|---|
| ServiceDesk Plus | LDAP Import + Auth | L1: `nguyenvana`, `tranthib`, `levanc` / L2: `phamvand` / L3: `hoangvane` |
| Zabbix | LDAP Auth (`sAMAccountName`) | Tất cả 5 KTV |
| Grafana | LDAP Auth (`ldap.toml`) | Tất cả 5 KTV |

**Tóm tắt các endpoint:**

| Service | URL | Credentials |
|---|---|---|
| Zabbix | `http://10.10.200.11:8080` | `Admin / <password>` |
| Grafana | `http://10.10.200.11:3000` | `admin / <password>` hoặc AD user |
| ServiceDesk Plus | `https://10.10.200.11:8400` | `administrator / <password>` hoặc AD user |

**Thứ tự khởi động toàn bộ stack:**

```bash
# 1. PostgreSQL
sudo systemctl start postgresql

# 2. Zabbix
sudo systemctl start zabbix-server zabbix-agent2

# 3. Nginx + PHP (Zabbix frontend)
sudo systemctl start nginx php8.1-fpm

# 4. Grafana
sudo systemctl start grafana-server

# 5. SDP
sudo systemctl start sdp
```

**Hướng phát triển tiếp theo:**

- **Auto-resolve ticket SDP** khi Zabbix báo RECOVERY
- **Email notification** qua SMTP relay của AD (Exchange/Postfix)
- **Weekly incident report** tổng hợp tự động mỗi thứ Hai
