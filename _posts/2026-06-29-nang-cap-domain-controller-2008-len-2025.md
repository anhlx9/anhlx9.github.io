---
title: "Nâng cấp Domain Controller Windows Server 2008 lên 2025 (qua trung gian 2019)"
categories:
- Windows
tags:
- active-directory
- windows-server
- domain-controller
- migration
- adcs
- dns
- dhcp
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Migrate DC 2008 → 2019 → 2025, giữ nguyên DNS, DHCP, ADCS và FSMO
---

Lab nâng cấp một forest Active Directory đang chạy **Windows Server 2008 SP2 32 bit** lên **Windows Server 2025 64 bit**, kèm migrate toàn bộ role hạ tầng (DNS, DHCP, ADCS, Print Server). Vì WS 2025 yêu cầu Forest Functional Level ≥ 2016 nên không lên thẳng từ FFL 2008 được — lab đi qua một DC trung gian WS 2019 để nâng FFL trước. Flow gồm 9 bước:

1. Chuẩn bị `win2008` làm DC gốc + sinh dữ liệu test (OU, user, DNS zone, DHCP, ADCS).
2. Backup toàn diện trên `win2008` (System State, ADCS, DHCP, Printer, FSMO).
3. Dựng `win2019` làm DC trung gian (join + promote).
4. Migrate toàn bộ role (FSMO, DNS, DHCP, ADCS, Print) `win2008` → `win2019`.
5. Soak vài hôm rồi demote `win2008`.
6. Nâng DFL/FFL lên Windows Server 2016 (lúc này forest chỉ còn `win2019`).
7. Cài `win2025`, join domain, promote lên DC (adprep tự chạy, nâng schema).
8. Migrate role `win2019` → `win2025`, soak rồi demote `win2019`, nâng DFL/FFL lên Windows Server 2025.
9. Kiểm tra cuối.

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/50.png)

## Mục lục

- [Mục tiêu](#mục-tiêu)
- [Môi trường](#môi-trường)
- [Chuẩn bị](#chuẩn-bị)
- [Tại sao cần DC trung gian 2019?](#tại-sao-cần-dc-trung-gian-2019)
- [Các bước thực hiện](#các-bước-thực-hiện)
  - [PHẦN 1 – Chuẩn bị VM1: win2008](#phần-1--chuẩn-bị-vm1-win2008)
  - [PHẦN 2 – Backup toàn diện trên win2008](#phần-2--backup-toàn-diện-trên-win2008)
  - [PHẦN 3 – Cài win2019 làm DC trung gian](#phần-3--cài-win2019-làm-dc-trung-gian)
  - [PHẦN 4 – Migrate toàn bộ role 2008 → 2019](#phần-4--migrate-toàn-bộ-role-2008--2019)
  - [PHẦN 5 – Soak rồi demote win2008](#phần-5--soak-rồi-demote-win2008)
  - [PHẦN 6 – Nâng DFL/FFL lên Windows Server 2016](#phần-6--nâng-dflffl-lên-windows-server-2016)
  - [PHẦN 7 – Cài và promote win2025](#phần-7--cài-và-promote-win2025)
  - [PHẦN 8 – Migrate role sang win2025, demote win2019 và nâng FFL 2025](#phần-8--migrate-role-sang-win2025-demote-win2019-và-nâng-ffl-2025)
- [Kiểm tra kết quả](#kiểm-tra-kết-quả)
  - [Bảng kết quả mong đợi](#bảng-kết-quả-mong-đợi)
- [Kết luận](#kết-luận)

## Mục tiêu

- Hiểu lộ trình nâng cấp DC nhiều bậc khi chênh lệch functional level quá lớn.
- Thực hành backup/restore ADCS, DHCP, Print Server giữa các DC.
- Migrate FSMO an toàn và dọn metadata DC cũ.

## Môi trường

3 VM trên VMware ESXi, cùng spec, cùng VLAN 200.

| VM | Tên máy | OS | IP | vCPU | RAM | Disk C | Disk D |
|----|---------|-----|-----|------|-----|--------|--------|
| VM1 | win2008 | WS 2008 SP2 x86 (32-bit) | 10.10.200.41 | 8 | 8GB | 200GB | 500GB |
| VM2 | win2019 | WS 2019 STD x64 (64-bit) | 10.10.200.42 | 8 | 8GB | 200GB | 500GB |
| VM3 | win2025 | WS 2025 STD x64 (24H2) | 10.10.200.43 | 8 | 8GB | 200GB | 500GB |

| Thành phần | Thông số |
|-----------|---------|
| Domain | `anhlx.lab` |
| Network | VLAN 200 — 10.10.200.0/24, gateway 10.10.200.1 |
| DNS forwarder | 8.8.8.8 |
| CA | ANHLX-CA (Enterprise Root CA) |
| Backup target | ổ `D:` (500GB) |

> **Lý do cần DC trung gian win2019:** WS 2025 yêu cầu Forest Functional Level ≥ **Windows Server 2016**. FFL gốc là 2008 → promote WS 2025 thẳng sẽ báo lỗi `The forest functional level must be Windows Server 2016 or higher`. Phải nâng FFL qua `win2019` trước.


## Chuẩn bị

- 3 VM đã cài OS, đặt đúng tên máy và IP như bảng trên.
- ISO Windows Server 2008 SP2 32-bit, 2019 Eval x64, 2025 Eval x64.
- Mỗi VM có ổ `D:` 500GB để chứa backup và file migrate.
- Toàn bộ password trong bài (DSRM, local admin, CA backup, user) dùng `Zxcasd123!@#` cho lab. ⚠️ **Môi trường production phải đổi sang mật khẩu mạnh riêng từng dịch vụ.**

## Tại sao cần DC trung gian 2019?

> Lộ trình "lên thẳng 2025" **không khả thi** từ FFL 2008.

- WS 2025 yêu cầu **Forest Functional Level ≥ Windows Server 2016**.
- FFL hiện tại là **Windows Server 2008** → promote WS 2025 DC thẳng sẽ fail.
- Thêm 1 DC trung gian (`win2019`) để gỡ được `win2008` mà domain vẫn còn DC, **sau đó mới nâng FFL lên 2016** (FFL 2016 yêu cầu mọi DC ≥ Windows Server 2016).
- **Lộ trình thực tế:** thêm `win2019` → migrate toàn bộ role sang `win2019` → soak → demote `win2008` → nâng FFL 2016 → thêm `win2025` → migrate role sang `win2025` → demote `win2019` → nâng FFL 2025.

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/01.png)

## Các bước thực hiện

### PHẦN 1 – Chuẩn bị VM1: win2008

> Nếu đã có sẵn VM win2008 từ lab khác, dùng lại — bỏ qua phần này.

#### 1.1. Cài WS2008 SP2 32-bit và đặt tên `win2008`

```
IP:       10.10.200.41
Subnet:   255.255.255.0
Gateway:  10.10.200.1
DNS:      127.0.0.1
```

#### 1.2. Promote lên DC – tạo domain `anhlx.lab`

```cmd
dcpromo
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/02.png)

- Create new domain in new forest
- FQDN: `anhlx.lab`
- Forest/Domain Functional Level: **Windows Server 2008**
- Tích DNS Server
- DSRM password: `Zxcasd123!@#`

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/03.png)

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/04.png)

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/05.png)

**Cấu hình DNS Forwarder → 8.8.8.8** (sau khi reboot và login lại):

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/06.png)

```cmd
dnscmd /resetforwarders 8.8.8.8

:: Kiểm tra
dnscmd /info /forwarders
nslookup google.com
```

> Bước này bắt buộc trước khi cài DNS zones phụ và các role khác, đảm bảo win2008 resolve được internet.

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/07.png)

#### 1.3. Tạo dữ liệu test mô phỏng môi trường

```cmd
:: ===== OUs =====
dsadd ou "OU=Sample,DC=anhlx,DC=lab"
dsadd ou "OU=Computers,OU=Sample,DC=anhlx,DC=lab"
dsadd ou "OU=Groups,OU=Sample,DC=anhlx,DC=lab"
dsadd ou "OU=Users,OU=Sample,DC=anhlx,DC=lab"

:: Phòng ban mẫu dưới Users
dsadd ou "OU=IT,OU=Users,OU=Sample,DC=anhlx,DC=lab"
dsadd ou "OU=Finance,OU=Users,OU=Sample,DC=anhlx,DC=lab"
dsadd ou "OU=Sales,OU=Users,OU=Sample,DC=anhlx,DC=lab"

:: ===== Groups (Global Security) trong OU=Groups =====
dsadd group "CN=IT,OU=Groups,OU=Sample,DC=anhlx,DC=lab"      -secgrp yes -scope g
dsadd group "CN=Finance,OU=Groups,OU=Sample,DC=anhlx,DC=lab" -secgrp yes -scope g
dsadd group "CN=Sales,OU=Groups,OU=Sample,DC=anhlx,DC=lab"   -secgrp yes -scope g

:: ===== Users mẫu (1-2 user/phòng để test) =====
set GRP=OU=Groups,OU=Sample,DC=anhlx,DC=lab

dsadd user "CN=Sample User01,OU=IT,OU=Users,OU=Sample,DC=anhlx,DC=lab" -samid user01 -fn "Sample" -ln "User01" -display "Sample User01" -pwd Zxcasd123!@# -disabled no -mustchpwd no
dsmod group "CN=IT,%GRP%" -addmbr "CN=Sample User01,OU=IT,OU=Users,OU=Sample,DC=anhlx,DC=lab"

dsadd user "CN=Sample User02,OU=Finance,OU=Users,OU=Sample,DC=anhlx,DC=lab" -samid user02 -fn "Sample" -ln "User02" -display "Sample User02" -pwd Zxcasd123!@# -disabled no -mustchpwd no
dsmod group "CN=Finance,%GRP%" -addmbr "CN=Sample User02,OU=Finance,OU=Users,OU=Sample,DC=anhlx,DC=lab"

dsadd user "CN=Sample User03,OU=Sales,OU=Users,OU=Sample,DC=anhlx,DC=lab" -samid user03 -fn "Sample" -ln "User03" -display "Sample User03" -pwd Zxcasd123!@# -disabled no -mustchpwd no
dsmod group "CN=Sales,%GRP%" -addmbr "CN=Sample User03,OU=Sales,OU=Users,OU=Sample,DC=anhlx,DC=lab"
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/08.png)

#### 1.4. Cài DNS zones phụ

Tạo nhiều zone để minh hoạ migrate DNS đa zone về sau.

> ⚠️ **Bắt buộc dùng `/dsprimary` (AD-integrated), KHÔNG dùng `/primary`.** Zone tạo bằng `/primary` là standard primary — lưu **file** trên chính `win2008`, **không replicate qua AD** → khi `win2008` bị gỡ thì zone biến mất. Chỉ zone **AD-integrated** (`/dsprimary`) mới nằm trong partition `DomainDnsZones`/`ForestDnsZones` và tự theo sang `win2019`/`win2025`.

```cmd
:: Forward Lookup Zones (8 zones mẫu) — AD-integrated
dnscmd /zoneadd sample1.local /dsprimary
dnscmd /zoneadd sample2.local /dsprimary
dnscmd /zoneadd sample3.local /dsprimary
dnscmd /zoneadd sample4.local /dsprimary
dnscmd /zoneadd sample5.local /dsprimary
dnscmd /zoneadd sample6.local /dsprimary
dnscmd /zoneadd sample7.local /dsprimary
dnscmd /zoneadd sample8.local /dsprimary

:: Reverse Lookup Zones (4 zones) — AD-integrated
dnscmd /zoneadd 0.16.172.in-addr.arpa   /dsprimary
dnscmd /zoneadd 1.16.172.in-addr.arpa   /dsprimary
dnscmd /zoneadd 3.16.172.in-addr.arpa   /dsprimary
dnscmd /zoneadd 30.168.192.in-addr.arpa /dsprimary
```

> Nếu lỡ tạo bằng `/primary` rồi, chuyển sang AD-integrated bằng: `dnscmd /zoneresettype <zone> /dsprimary` (chạy trên DC còn giữ zone, trước khi demote nó).

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/09.png)

#### 1.5. Cài DHCP – 2 scopes

```cmd
:: Server Manager → Add Roles → DHCP Server
:: Tạo scope 172.16.1.100–200 và 172.16.3.100–200
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/10.png)

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/11.png)

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/12.png)

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/13.png)

#### 1.6. Cài ADCS – ANHLX-CA

```cmd
:: Server Manager → Add Roles → Active Directory Certificate Services
:: CA Type: Enterprise Root CA
:: CA Name: ANHLX-CA
```

#### 1.7. Kiểm tra và chụp Snapshot

> Mục đích: xác nhận DC healthy trước khi nâng cấp. Có lỗi ở đây → fix trước, không làm tiếp.

```cmd
dcdiag /test:replications
dcdiag /test:services
netdom query fsmo
```

**Kết quả mong đợi:**

- `dcdiag`: các test đều **passed**.
- `netdom query fsmo`: cả 5 roles đều hiện `win2008.anhlx.lab`.

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/14.png)

Chụp Snapshot **`win2008-Clean`** trong VMware ← rollback về đây nếu nâng cấp lỗi.

### PHẦN 2 – Backup toàn diện trên win2008

> Thực hiện trước khi đụng vào bất cứ thứ gì.

```cmd
:: Tạo thư mục backup tập trung trên ổ D:
mkdir D:\backup_DC
```

#### 2.1. Backup System State

> `wbadmin` yêu cầu Windows Server Backup feature được cài trước.

```cmd
:: Bước 1 – Cài Windows Server Backup feature (nếu chưa có)
ServerManagerCmd -install Backup-Features

:: Bước 2 – Backup System State ra D: (wbadmin không hỗ trợ subfolder)
wbadmin start systemstatebackup -backuptarget:D: -quiet
```

> Backup System State lưu tự động vào `D:\WindowsImageBackup\`.

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/15.png)

#### 2.2. Backup ADCS (ANHLX-CA) – quan trọng nhất

> `certutil -backup` yêu cầu certsvc **đang chạy** để backup database qua RPC. Không dừng service trước.

```cmd
:: Đảm bảo certsvc đang chạy
net start certsvc

:: Backup CA key (.p12) + database (live backup)
certutil -backup -p "Zxcasd123!@#" D:\backup_DC\CABackup

:: Backup registry CA config
reg export HKLM\SYSTEM\CurrentControlSet\Services\CertSvc D:\backup_DC\CABackup\CertSvc_Registry.reg
```

Kiểm tra: `dir D:\backup_DC\CABackup` — phải thấy thư mục `DataBase` và file `ANHLX-CA.p12`.

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/16.png)

#### 2.3. Backup DHCP

```cmd
netsh dhcp server export D:\backup_DC\dhcp_export.xml all
```

#### 2.4. Backup Printers

> `printbrm.exe` không nằm trong PATH — phải dùng full path. Cần role **Print and Document Services** trước.

```cmd
:: Bước 1 – Cài Print Server role (nếu chưa cài)
ServerManagerCmd -install Print-Services

:: Bước 2 – Backup (dùng full path)
%windir%\system32\spool\tools\printbrm.exe -B -F D:\backup_DC\PrinterBackup.printerExport
```

Kết quả: `Successfully finished operation` → file `PrinterBackup.printerExport` trong `D:\backup_DC\`.

#### 2.5. Ghi lại FSMO roles

```cmd
netdom query fsmo > D:\backup_DC\fsmo_before.txt
type D:\backup_DC\fsmo_before.txt
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/17.png)

### PHẦN 3 – Cài win2019 làm DC trung gian

#### 3.1. Cài WS2019 và đặt tên `win2019`

1. Tạo VM2 → mount ISO WS2019 Eval x64.
2. Cài **Windows Server 2019 Standard (Desktop Experience)**.
3. Đổi tên: `win2019`.
4. Cấu hình IP:

```
IP:       10.10.200.42
Subnet:   255.255.255.0
Gateway:  10.10.200.1
DNS:      10.10.200.41    ← trỏ về win2008
```

#### 3.2. Test kết nối và join domain

```powershell
ping 10.10.200.41
nslookup anhlx.lab 10.10.200.41

Add-Computer -DomainName "anhlx.lab" -Credential (Get-Credential) -Restart
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/18.png)

#### 3.3. Promote win2019 lên DC

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Import-Module ADDSDeployment
Install-ADDSDomainController `
    -DomainName "anhlx.lab" `
    -InstallDns:$true `
    -CreateDnsDelegation:$false `
    -DatabasePath "C:\Windows\NTDS" `
    -LogPath "C:\Windows\NTDS" `
    -SysvolPath "C:\Windows\SYSVOL" `
    -NoGlobalCatalog:$false `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "Zxcasd123!@#" -AsPlainText -Force) `
    -Force:$true
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/19.png)

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/20.png)

### PHẦN 4 – Migrate toàn bộ role 2008 → 2019

> **Chiến lược không-downtime:** dồn **tất cả role sống** (FSMO, DNS, DHCP, ADCS, Print) sang `win2019` trong khi `win2008` vẫn chạy. Sau khi services chạy ổn trên `win2019` (qua giai đoạn soak ở PHẦN 5), mới demote `win2008`. Mỗi lần đổi máy chỉ là cutover ngắn vài phút, không có khoảng trống dài như khi để role về tận `win2025`.

#### 4.1. Chuyển toàn bộ FSMO sang win2019

```powershell
# Trên win2019 — kiểm tra hiện trạng (cả 5 role đang ở win2008)
netdom query fsmo

# Chuyển 5 role sang win2019
Move-ADDirectoryServerOperationMasterRole -Identity "win2019" `
    -OperationMasterRole PDCEmulator, RIDMaster, InfrastructureMaster, SchemaMaster, DomainNamingMaster

# Xác nhận — tất cả phải hiện win2019.anhlx.lab
netdom query fsmo
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/21.png)

#### 4.2. Trỏ DNS của win2019 về chính nó

`win2019` đang trỏ DNS về `win2008`; chuẩn bị gỡ `win2008` nên đổi về chính nó:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses "127.0.0.1"
```

Cấu hình DNS forwarder cho `win2019` (forwarder là per-server, **không** replicate giữa các DC nên phải set lại — `win2008` sắp tắt):

```powershell
# Trỏ forwarder ra 8.8.8.8
Set-DnsServerForwarder -IPAddress 8.8.8.8

# Kiểm tra
Get-DnsServerForwarder
nslookup google.com
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/22.png)

#### 4.3. Đưa file backup từ win2008 sang win2019

DHCP/ADCS/Print sẽ được restore lên `win2019` từ backup đã tạo ở PHẦN 2. Copy thư mục backup sang `win2019` (cả hai máy đang chạy):

**Trên win2008 — share thư mục backup:**
```cmd
net share backup_DC=D:\backup_DC /grant:Everyone,READ
```

**Trên win2019 — kéo file về:**
```powershell
robocopy \\win2008\backup_DC D:\backup_DC /E
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/23.png)

Xác nhận đủ file (`CABackup\`, `dhcp_export.xml`, `PrinterBackup.printerExport`) rồi gỡ share:
```cmd
:: Trên win2008
net share backup_DC /delete
```

#### 4.4. Migrate DHCP → win2019

**Trên win2019:**
```powershell
Install-WindowsFeature DHCP -IncludeManagementTools

# Authorize win2019
Add-DhcpServerInDC -DnsName "win2019.anhlx.lab" -IPAddress 10.10.200.42

# Import scope từ backup
netsh dhcp server import D:\backup_DC\dhcp_export.xml all

# Trỏ option DNS của scope về win2019
Set-DhcpServerv4OptionValue -ScopeId 172.16.1.0 -OptionId 6 -Value 10.10.200.42
Set-DhcpServerv4OptionValue -ScopeId 172.16.3.0 -OptionId 6 -Value 10.10.200.42

Get-DhcpServerv4Scope
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/24.png)

**Trên win2008 — gỡ authorization & dừng DHCP cũ:**

```cmd
:: Gỡ authorization DHCP khỏi AD
netsh dhcp delete server win2008.anhlx.lab 10.10.200.41

:: Dừng & disable service
net stop DHCPServer
sc config DHCPServer start= disabled
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/25.png)

#### 4.5. Migrate ADCS (ANHLX-CA) → win2019

> **Cutover CA:** CA = 1 danh tính (cùng key + tên), **không chạy đồng thời 2 máy** được. Quy trình: gỡ CA trên `win2008` → restore lên `win2019` (hai bước liền nhau, gap vài phút). Cert đã cấp + CRL trong AD vẫn validate xuyên suốt nên không gián đoạn dịch vụ. Việc gỡ CA ở đây **đồng thời thỏa điều kiện demote `win2008`** ở PHẦN 5 — Enterprise CA phụ thuộc AD DS, còn CA thì `dcpromo` sẽ báo lỗi.

**Trên win2008 — gỡ role CA (đã backup ở PHẦN 2.2):**
```cmd
ServerManagerCmd -remove ADCS-Cert-Authority
```
Hoặc GUI: Server Manager → Roles → Remove Roles → bỏ tích **Active Directory Certificate Services**.

**Trên win2019 — restore CA cùng tên `ANHLX-CA`:**

> Thứ tự đúng: cấu hình CA dùng lại **key cũ** từ file `.p12` trước (`-CertFile`), **rồi mới** restore database. `certutil -restore` chỉ chạy được khi CA đã cấu hình — gọi nó trước `Install-AdcsCertificationAuthority` sẽ lỗi *"The system cannot find the file specified"*.

```powershell
# 1. Cài role (binaries)
Install-WindowsFeature ADCS-Cert-Authority -IncludeManagementTools

# 2. Cấu hình CA, nạp lại key + cert cũ từ .p12 (giữ nguyên danh tính ANHLX-CA)
$pw = ConvertTo-SecureString "Zxcasd123!@#" -AsPlainText -Force
Install-AdcsCertificationAuthority `
    -CAType EnterpriseRootCA `
    -CertFile "D:\backup_DC\CABackup\ANHLX-CA.p12" `
    -CertFilePassword $pw `
    -OverwriteExistingKey `
    -Force

# 3. Khôi phục registry config (CDP/AIA, validity, template settings...)
reg import D:\backup_DC\CABackup\CertSvc_Registry.reg

# 4. Restore database (CA đã cấu hình nên restoredb mới chạy được)
Stop-Service certsvc
certutil -f -restoredb "D:\backup_DC\CABackup"
Start-Service certsvc
```

**Kiểm tra sau restore:**

```powershell
# CA sống — output của -ping in luôn tên CA (ANHLX-CA) và thời gian phản hồi
certutil -ping

# Xem CDP (lưu ý dưới)
Get-CACrlDistributionPoint

# Lịch sử cert đã cấp có restore không (Disposition=20 = Issued)
certutil -view -restrict "Disposition=20" -out "RequestID,CommonName" | more

# Publish lại CRL để chắc chắn CA ký được
certutil -CRL
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/26.png)

Mở thêm `certsrv.msc` (thấy `ANHLX-CA` Running) và `pkiview.msc` (Enterprise PKI — tất cả phải xanh).

> **`0 Rows` ở bước xem lịch sử là bình thường** nếu CA chưa từng cấp cert cho client nào (CA lab mới tạo) — database vẫn được restore đúng (`-restoredb` báo 100%). Khi có cert đã cấp thì các request sẽ hiện ở đây.

> **Lưu ý CDP/AIA:** mặc định Enterprise CA dùng URL dạng **biến** (`http://<ServerDNSName>/...`, `ldap:///...`) nên tự trỏ về server đang chạy CA — không hardcode tên máy cũ. Nếu môi trường production có cấu hình **CDP/AIA cố định theo tên `win2008`** thì sau khi tắt máy đó client không tải được CRL/AIA; sửa bằng `Get-CACrlDistributionPoint` / `Get-CAAuthorityInformationAccess` → trỏ lại DC mới → `certutil -CRL`.

##### pkiview báo `Unable To Download` ở các location HTTP

Sau khi restore, `pkiview.msc` thường hiện CA màu **đỏ (Error)** với 3 location HTTP báo `Unable To Download`:

```
AIA #2      http://win2019.anhlx.lab/CertEnroll/...crt   Unable To Download
CDP #2      http://win2019.anhlx.lab/CertEnroll/ANHLX-CA.crl   Unable To Download
DeltaCRL #2 http://win2019.anhlx.lab/CertEnroll/ANHLX-CA+.crl  Unable To Download
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/27.png)

Các location **LDAP** và **CA Certificate** đều OK → CA hoàn toàn khỏe, chỉ là DC mới **chưa có IIS virtual directory `/CertEnroll`** để phục vụ file qua HTTP. Cách production chuẩn: tạo vdir `CertEnroll` trỏ tới thư mục CA publish file — **không cần** AD CS Web Enrollment (trang /certsrv legacy, và `Install-AdcsWebEnrollment` hay lỗi `0x80070057` trên Server mới).

```powershell
# Cài IIS web role nếu chưa có
Install-WindowsFeature Web-WebServer,Web-Mgmt-Console -IncludeManagementTools

Import-Module WebAdministration

# vdir CertEnroll → nơi CA ghi .crt/.crl
New-WebVirtualDirectory -Site "Default Web Site" -Name CertEnroll `
  -PhysicalPath "$env:windir\System32\CertSrv\CertEnroll"

# BẮT BUỘC: cho phép double escaping — delta CRL có dấu '+' trong tên (ANHLX-CA+.crl),
# IIS mặc định chặn => DeltaCRL HTTP sẽ 404 nếu không bật
Set-WebConfigurationProperty -PSPath "IIS:\Sites\Default Web Site\CertEnroll" `
  -Filter "system.webServer/security/requestFiltering" -Name allowDoubleEscaping -Value $true

certutil -CRL
iisreset
```

Test: `http://win2019.anhlx.lab/CertEnroll/ANHLX-CA.crl` tải được. Refresh `pkiview.msc` (F5) → các location HTTP chuyển **OK**, CA xanh hết.

> Điểm production hay quên khi migrate CA: vdir HTTP `CertEnroll` nằm ở IIS của **từng DC chạy CA**, không replicate theo AD — mỗi lần CA sang máy mới đều phải dựng lại vdir này (kèm bật `allowDoubleEscaping`).

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/28.png)

#### 4.6. Migrate Print Server → win2019

**Trên win2019:**
```powershell
Install-WindowsFeature Print-Server -IncludeManagementTools

# printbrm.exe không nằm trong PATH — gọi full path
& "$env:windir\System32\spool\tools\printbrm.exe" -R -F D:\backup_DC\PrinterBackup.printerExport
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/29.png)

Đến đây toàn bộ role (FSMO, DNS, DHCP, ADCS, Print) đã chạy trên `win2019`.

### PHẦN 5 – Soak rồi demote win2008

Toàn bộ role đã nằm trên `win2019`. Trước khi gỡ hẳn `win2008`, **tắt nó và chạy một mình `win2019` vài hôm** để chắc chắn không sót dịch vụ/cấu hình nào còn trỏ về `win2008`.

#### 5.1. Power off win2008 và soak (burn-in) vài hôm

```powershell
# Trên win2008
Stop-Computer
```

Trong thời gian soak, kiểm tra trên `win2019` (và client):

- `dcdiag` không FAIL; `netdom query fsmo` — cả 5 role = `win2019.anhlx.lab`.
- DNS resolve nội bộ + internet, client nhận IP từ DHCP, `certutil -ping` OK, in thử qua print server.
- Theo dõi Event Log vài hôm xem có lỗi tham chiếu `win2008` không.

> **Vì sao soak an toàn:** đây là **power off/on bình thường**, KHÔNG phải revert snapshot → nếu cần, bật `win2008` lại nó sẽ replicate inbound từ `win2019` như thường, **không gây USN rollback**. Chỉ cần đừng để `win2008` off lâu hơn **tombstone lifetime** (mặc định 60/180 ngày) — quá hạn đó thì không được bật lại mà phải metadata cleanup.

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/30.png)

#### 5.2. Demote win2008 (sau khi soak ổn)

> ⚠️ `win2008` đã sạch CA (gỡ ở 4.5) và không còn FSMO nên `dcpromo` không báo lỗi AD CS nữa. Đảm bảo snapshot `win2008-Clean` còn nguyên (xem Fallback bên dưới).

Bật `win2008` lên rồi demote:

```cmd
:: Trên win2008
dcpromo
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/31.png)

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/32.png)

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/33.png)

Bỏ chọn vai trò Domain Controller. Nếu wizard lỗi không gỡ được, dùng `dcpromo /forceremoval`. Nhập password local admin, restart → `win2008` thành member server.

**Xóa metadata (trên win2019):**

Demote kiểu wizard (clean) đã tự dọn metadata. Chạy lệnh sau cho chắc; thường nó cũng xong luôn:

```powershell
Get-ADDomainController -Filter {Name -eq "win2008"} | Remove-ADObject -Recursive -Confirm:$false

# Kiểm tra — chỉ còn win2019
Get-ADDomainController -Filter * | Select-Object Name, OperatingSystem
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/34.png)


Sau đó **power off `win2008` vĩnh viễn** (xóa VM sau khi hoàn tất).

> **🔁 Fallback / Rollback**
>
> Snapshot `win2008-Clean` (PHẦN 1.7) là bản gốc của forest — chụp trước khi có `win2019`/`win2025`, FSMO còn nguyên trên `win2008`. Nếu sau khi demote mà phát hiện sự cố nghiêm trọng không khắc phục được, rollback về hiện trạng ban đầu:
>
> 1. **Power off `win2019` (và `win2025` nếu đã có)** — tuyệt đối không để chúng chạy/replicate với `win2008` sau khi restore.
> 2. Restore `win2008` về snapshot **`win2008-Clean`** rồi power on. `win2008` trở lại DC duy nhất, giữ đủ 5 FSMO, không biết gì về `win2019`/`win2025`.
> 3. Verify: `dcdiag` không FAIL, `netdom query fsmo` cả 5 role = `win2008.anhlx.lab`.
> 4. **Xóa hẳn (hoặc dựng lại từ đầu) VM `win2019` + `win2025`.** ⚠️ KHÔNG bật lại để join vào `win2008` đã restore — sẽ gây **USN rollback / lingering objects** làm hỏng replication.
>
> Lưu ý phân biệt: rollback này dùng **revert snapshot** (mới gây USN rollback) — khác với power off/on khi soak ở 5.1.

### PHẦN 6 – Nâng DFL/FFL lên Windows Server 2016

> Giờ forest chỉ còn `win2019` (Server 2019 ≥ 2016) và đang giữ toàn bộ FSMO nên nâng được. Đây là điều kiện để promote `win2025`.

```powershell
# Trên win2019
Get-ADForest | Select-Object ForestMode
Get-ADDomain | Select-Object DomainMode

# Nâng Domain + Forest Functional Level lên 2016
Set-ADDomainMode -Identity "anhlx.lab" -DomainMode Windows2016Domain
Set-ADForestMode -Identity "anhlx.lab" -ForestMode Windows2016Forest

# Xác nhận
Get-ADDomain | Select-Object DomainMode
Get-ADForest | Select-Object ForestMode
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/35.png)

**Kết quả mong đợi:** `Windows2016Domain` / `Windows2016Forest` → `win2025` mới promote được.

> Vì FSMO đã nằm trên `win2019` và `win2008` đã rời forest, lệnh không còn báo `A referral was returned from the server`.

### PHẦN 7 – Cài và promote win2025

#### 7.1. Cài WS2025 và đặt tên `win2025`

1. Tạo VM3 → mount ISO WS2025 Eval x64.
2. Cài **Windows Server 2025 Standard (Desktop Experience)** — bản 24H2, build 26100.
3. Đổi tên: `win2025`.
4. Cấu hình IP:

```
IP:       10.10.200.43
Subnet:   255.255.255.0
Gateway:  10.10.200.1
DNS:      10.10.200.42    ← trỏ về win2019 (win2008 đã gỡ)
```

#### 7.2. Test kết nối trước khi join

```powershell
ping 10.10.200.42
nslookup anhlx.lab 10.10.200.42
nslookup win2025 10.10.200.42
```

Phải resolve được domain. Nếu không — kiểm tra lại DNS và VMnet.

#### 7.3. Join domain anhlx.lab

```powershell
Add-Computer -DomainName "anhlx.lab" -Credential (Get-Credential) -Restart
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/36.png)

#### 7.4. Cài AD DS role

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

#### 7.5. Promote (adprep chạy tự động)

```powershell
Import-Module ADDSDeployment

Install-ADDSDomainController `
    -DomainName "anhlx.lab" `
    -InstallDns:$true `
    -CreateDnsDelegation:$false `
    -DatabasePath "C:\Windows\NTDS" `
    -LogPath "C:\Windows\NTDS" `
    -SysvolPath "C:\Windows\SYSVOL" `
    -NoGlobalCatalog:$false `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "Zxcasd123!@#" -AsPlainText -Force) `
    -Force:$true
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/37.png)

> WS2025 sẽ tự động chạy `adprep /forestprep` và `adprep /domainprep` để nâng schema lên 2025 (có thể mất 5–10 phút).

Sau khi reboot, đăng nhập `anhlx\administrator`.

#### 7.6. Kiểm tra replication

Chờ 10–15 phút sau reboot rồi chạy trên `win2025`.

**a. Tổng quan replication — phải 0 fail:**
```powershell
repadmin /replsummary
```
Kết quả mong đợi: cột `fails/total` = `0 / N`, cả `WIN2019` và `WIN2025` đều 0 lỗi.

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/38.png)

**b. Chi tiết partner — mọi partition "successful":**
```powershell
repadmin /showrepl
```
Phải thấy `win2025` có INBOUND từ `win2019` cho đủ 5 partition (`DC=anhlx,DC=lab`, `Configuration`, `Schema`, `ForestDnsZones`, `DomainDnsZones`), mỗi dòng `Last attempt @ ... was successful`.

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/39.png)

**c. Test replication:**
```powershell
dcdiag /test:replications
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/40.png)

**d. DNS zones đã replicate chưa:**
```powershell
Get-DnsServerZone | Select-Object ZoneName, IsADIntegrated, ZoneType
```
Phải thấy đủ `anhlx.lab`, `_msdcs.anhlx.lab`, **8 zone `sample*.local` + 4 reverse zone**, cột `IsADIntegrated = True`.

> Nếu **thiếu** các zone `sample*` → chúng được tạo dạng `/primary` (file, không replicate) ở PHẦN 1.4 chứ không phải `/dsprimary`. Tạo lại dạng AD-integrated trên `win2025`:
> ```powershell
> "sample1.local","sample2.local","sample3.local","sample4.local",
> "sample5.local","sample6.local","sample7.local","sample8.local",
> "0.16.172.in-addr.arpa","1.16.172.in-addr.arpa","3.16.172.in-addr.arpa","30.168.192.in-addr.arpa" |
>   ForEach-Object { dnscmd /zoneadd $_ /dsprimary }
> ```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/41.png)

#### 7.7. Kiểm tra schema đã nâng

```powershell
Get-ADObject (Get-ADRootDSE).schemaNamingContext -Properties objectVersion |
    Select-Object objectVersion
```

- WS2008 = 44
- WS2025 = 91

Giá trị phải là **91** → schema đã nâng thành công.

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/42.png)

### PHẦN 8 – Migrate role sang win2025, demote win2019 và nâng FFL 2025

`win2025` đã là DC (PHẦN 7). Giờ chuyển nốt role từ `win2019` → `win2025` rồi gỡ `win2019`. Quy trình lặp lại như PHẦN 4 nhưng **nguồn là `win2019`, đích là `win2025`** — lúc này không gấp, làm khi rảnh.

#### 8.1. Chuyển FSMO sang win2025

```powershell
netdom query fsmo

Move-ADDirectoryServerOperationMasterRole -Identity "win2025" `
    -OperationMasterRole PDCEmulator, RIDMaster, InfrastructureMaster, SchemaMaster, DomainNamingMaster

netdom query fsmo
```

Tất cả phải hiện `win2025.anhlx.lab`.

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/43.png)

#### 8.2. DNS — trỏ win2025 về chính nó + forwarder

DNS AD-Integrated zones tự replicate sang `win2025` (đã `-InstallDns` lúc promote). Cấu hình client DNS và forwarder:

```powershell
# Trên win2025
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses "127.0.0.1", "10.10.200.42"
Set-DnsServerForwarder -IPAddress 8.8.8.8
Get-DnsServerZone | Where-Object {$_.IsADIntegrated -eq $true}
```

#### 8.3. Migrate DHCP → win2025

**Trên win2019 — export :**
```powershell
netsh dhcp server export D:\dhcp_2019.xml all
```

Copy `D:\dhcp_2019.xml` sang `win2025`, rồi **trên win2025:**
```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
Add-DhcpServerInDC -DnsName "win2025.anhlx.lab" -IPAddress 10.10.200.43
netsh dhcp server import D:\dhcp_2019.xml all

# Trỏ option DNS của scope về win2025
Set-DhcpServerv4OptionValue -ScopeId 172.16.1.0 -OptionId 6 -Value 10.10.200.43
Set-DhcpServerv4OptionValue -ScopeId 172.16.3.0 -OptionId 6 -Value 10.10.200.43
Get-DhcpServerv4Scope
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/44.png)

**Trên win2019 — gỡ DHCP cũ:**
```powershell
Remove-DhcpServerInDC -DnsName "win2019.anhlx.lab" -IPAddress 10.10.200.42
Stop-Service DHCPServer
Set-Service DHCPServer -StartupType Disabled
```

#### 8.4. Migrate ADCS (ANHLX-CA) → win2025

> Cutover CA lần 2 (`win2019` → `win2025`), tương tự 4.5. Gỡ CA trên `win2019` đồng thời là điều kiện để demote `win2019` ở 8.6.

**Trên win2019 — backup (key + database + registry) rồi gỡ CA:**
```powershell
certutil -backup -p "Zxcasd123!@#" D:\CABackup_2019
reg export HKLM\SYSTEM\CurrentControlSet\Services\CertSvc D:\CABackup_2019\CertSvc_Registry.reg

# Gỡ Web Enrollment TRƯỚC (đã cài ở 4.5) — nếu không, gỡ CA sẽ báo
# "must uninstall Web enrollment role services before uninstalling the CA"
Uninstall-WindowsFeature ADCS-Web-Enrollment
Uninstall-WindowsFeature ADCS-Cert-Authority
```

Copy `D:\CABackup_2019` sang `win2025`, rồi **trên win2025** (thứ tự như 4.5: cấu hình bằng key cũ trước, restore database sau):
```powershell
Install-WindowsFeature ADCS-Cert-Authority -IncludeManagementTools

$pw = ConvertTo-SecureString "Zxcasd123!@#" -AsPlainText -Force
Install-AdcsCertificationAuthority `
    -CAType EnterpriseRootCA `
    -CertFile "D:\CABackup_2019\ANHLX-CA.p12" `
    -CertFilePassword $pw `
    -OverwriteExistingKey `
    -Force

reg import D:\CABackup_2019\CertSvc_Registry.reg

Stop-Service certsvc
certutil -f -restoredb "D:\CABackup_2019"
Start-Service certsvc

# win2025 là DC cuối — dựng lại vdir HTTP CertEnroll cho CDP/AIA (như mục 4.5)
Install-WindowsFeature Web-WebServer,Web-Mgmt-Console -IncludeManagementTools
Import-Module WebAdministration
New-WebVirtualDirectory -Site "Default Web Site" -Name CertEnroll `
  -PhysicalPath "$env:windir\System32\CertSrv\CertEnroll"
Set-WebConfigurationProperty -PSPath "IIS:\Sites\Default Web Site\CertEnroll" `
  -Filter "system.webServer/security/requestFiltering" -Name allowDoubleEscaping -Value $true
certutil -CRL
iisreset
```

Kiểm tra: `certutil -ping` thành công, `certsrv.msc` thấy `ANHLX-CA` Running, `pkiview.msc` (F5) tất cả location **xanh** (gồm cả HTTP nhờ vdir CertEnroll).

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/45.png)

#### 8.5. Migrate Print Server → win2025

**Trên win2019 — export tươi** (`printbrm.exe` gọi full path):
```powershell
& "$env:windir\System32\spool\tools\printbrm.exe" -B -F D:\Printers_2019.printerExport
```

Copy sang `win2025`, rồi **trên win2025:**
```powershell
Install-WindowsFeature Print-Server -IncludeManagementTools
& "$env:windir\System32\spool\tools\printbrm.exe" -R -F D:\Printers_2019.printerExport
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/46.png)

#### 8.6. Soak rồi demote win2019

Toàn bộ role đã sang `win2025`. Trước khi gỡ hẳn `win2019`, **tắt nó chạy mình `win2025` vài hôm** để chắc chắn không sót gì còn trỏ về `win2019` — giống cách soak `win2008` ở PHẦN 5.

**Soak:** trên `win2019` chạy `Stop-Computer`, rồi trong vài hôm kiểm tra trên `win2025` (và client):
- `dcdiag` không FAIL; `netdom query fsmo` cả 5 role = `win2025.anhlx.lab`.
- DNS resolve nội bộ + internet, client nhận IP từ DHCP, `certutil -ping` OK, in thử qua print server.
- Event Log không có lỗi tham chiếu `win2019`.

> Như PHẦN 5: đây là **power off/on bình thường** (không revert snapshot) → cần thì bật `win2019` lại nó replicate bình thường từ `win2025`, **không USN rollback**. Đừng để off lâu hơn **tombstone lifetime**.

**Demote (sau khi soak ổn):**

> ⚠️ Chụp Snapshot `win2019` trước bước này. CA đã gỡ ở 8.4 nên demote không lỗi. Trang **Remove DNS Delegation** xuất hiện thì **bỏ chọn** (forest root, không có zone cha).

Bật `win2019` lên rồi:
```powershell
Uninstall-ADDSDomainController `
    -LocalAdministratorPassword (ConvertTo-SecureString "Zxcasd123!@#" -AsPlainText -Force) `
    -LastDomainControllerInDomain:$false `
    -RemoveDnsDelegation:$false `
    -Force:$true
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/47.png)

Máy tự restart → `win2019` thành member server. **Xóa metadata (trên win2025):**
```powershell
Get-ADDomainController -Filter {Name -eq "win2019"} | Remove-ADObject -Recursive -Confirm:$false
Get-ADDomainController -Filter * | Select-Object Name, OperatingSystem
```
![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/48.png)


#### 8.7. Nâng DFL/FFL lên Windows Server 2025

> Giờ DC duy nhất là `win2025` nên nâng được lên mức 2025.

```powershell
Set-ADDomainMode -Identity "anhlx.lab" -DomainMode Windows2025Domain
Set-ADForestMode -Identity "anhlx.lab" -ForestMode Windows2025Forest

Get-ADDomain | Select-Object Name, DomainMode
Get-ADForest | Select-Object Name, ForestMode
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/49.png)

## Kiểm tra kết quả

> `dcdiag` chạy bộ test mặc định (nhanh). Cần kiểm tra sâu thì dùng `dcdiag /c` (comprehensive) — chậm hơn nhiều vì có test DNS/CutoffServers; muốn nhanh thì bỏ qua test DNS: `dcdiag /c /skip:DNS`.

```powershell
# Health check tổng thể
dcdiag
repadmin /replsummary

# FSMO
netdom query fsmo

# DFL / FFL
Get-ADDomain | Select-Object DomainMode
Get-ADForest | Select-Object ForestMode

# Schema version (phải = 91)
Get-ADObject (Get-ADRootDSE).schemaNamingContext -Properties objectVersion | Select-Object objectVersion

# DHCP
Get-DhcpServerv4Scope
Get-DhcpServerInDC

# DNS
Get-DnsServerZone | Select-Object ZoneName, IsADIntegrated

# ADCS (phải: -ping command completed successfully)
certutil -ping

# Test user login
runas /user:anhlx\user01 cmd
```

![](/assets/img/2026-06-29-nang-cap-domain-controller-2008-len-2025/50.png)

### Bảng kết quả mong đợi

| Kiểm tra | Kết quả mong đợi |
|---|---|
| DomainMode | Windows2025Domain |
| ForestMode | Windows2025Forest |
| Schema objectVersion | 91 |
| FSMO (cả 5) | win2025.anhlx.lab |
| DHCP scopes | 172.16.1.0 + 172.16.3.0 active |
| DNS zones | Đủ 8 forward + 4 reverse |
| ADCS | ANHLX-CA Running trên win2025 |
| dcdiag | Không có FAIL |
| User login | Thành công với user01 |

## Kết luận

Forest đã nâng từ Windows Server 2008 lên 2025 qua DC trung gian 2019, giữ nguyên domain `anhlx.lab` cùng toàn bộ DNS zone, DHCP scope, ADCS (ANHLX-CA) và FSMO. Hai điểm mấu chốt: (1) **dồn hết role sống sang `win2019` trước**, soak vài hôm rồi mới demote `win2008` — mỗi lần đổi máy chỉ là cutover ngắn, không có khoảng trống dài; (2) FFL 2016 yêu cầu mọi DC ≥ Windows Server 2016 nên `win2008` (và FSMO của nó) phải rời forest trước khi nâng FFL — đảo thứ tự sẽ dính lỗi `A referral was returned from the server`. Cutover CA luôn có gap vài phút (CA không chạy 2 máy cùng lúc), nhưng cert đã cấp + CRL trong AD vẫn validate xuyên suốt. Backup đầy đủ trước mọi thao tác, soak là bài test fallback sống, và rollback bằng snapshot `win2008-Clean` luôn sẵn sàng nếu cần.
