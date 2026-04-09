---
title: "Lab Active Directory - Windows Server 2022"
categories:
- System
- Windows Server
- Active Directory
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Lab dựng Active Directory Domain Services trên Windows Server 2022
---

Bài viết hướng dẫn triển khai lab Active Directory Domain Services (AD DS) với 2 Domain Controller chạy Windows Server 2022 và 1 máy Windows client join domain.

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Thiết kế lab](#2-thiết-kế-lab)
  - [2.1. Topology](#21-topology)
  - [2.2. Kế hoạch IP](#22-kế-hoạch-ip)
- [3. Cài đặt Domain Controller 1 (DC1)](#3-cài-đặt-domain-controller-1-dc1)
  - [3.1. Cấu hình IP tĩnh](#31-cấu-hình-ip-tĩnh)
  - [3.2. Đổi hostname](#32-đổi-hostname)
  - [3.3. Cài đặt AD DS và DNS](#33-cài-đặt-ad-ds-và-dns)
  - [3.4. Promote lên Domain Controller](#34-promote-lên-domain-controller)
- [4. Cài đặt Domain Controller 2 (DC2)](#4-cài-đặt-domain-controller-2-dc2)
  - [4.1. Cấu hình IP tĩnh](#41-cấu-hình-ip-tĩnh)
  - [4.2. Đổi hostname](#42-đổi-hostname)
  - [4.3. Cài đặt AD DS và DNS](#43-cài-đặt-ad-ds-và-dns)
  - [4.4. Promote lên Domain Controller (Additional DC)](#44-promote-lên-domain-controller-additional-dc)
- [5. Join domain từ Windows Client](#5-join-domain-từ-windows-client)
  - [5.1. Cấu hình IP tĩnh](#51-cấu-hình-ip-tĩnh)
  - [5.2. Join domain](#52-join-domain)
- [6. Quản lý Active Directory](#6-quản-lý-active-directory)
  - [6.1. Tạo Organizational Unit (OU)](#61-tạo-organizational-unit-ou)
  - [6.2. Tạo User](#62-tạo-user)
  - [6.3. Kiểm tra Replication giữa 2 DC](#63-kiểm-tra-replication-giữa-2-dc)

---

### 1. Giới thiệu

**Active Directory Domain Services (AD DS)** là dịch vụ thư mục của Microsoft, cho phép quản lý tập trung người dùng, máy tính, group policy và các tài nguyên mạng trong môi trường doanh nghiệp.

Trong lab này, ta sẽ triển khai:
- **2 Domain Controller** chạy Windows Server 2022 để đảm bảo tính sẵn sàng cao (High Availability) cho AD DS
- **1 Windows Client** join domain để kiểm tra hoạt động

### 2. Thiết kế lab

#### 2.1. Topology

![Topology](/assets/img/2026-04-09-active-directory-windows-server-2022/topology.png)

#### 2.2. Kế hoạch IP

| Máy | Hostname | IP | Subnet Mask | Gateway | DNS |
|-----|----------|----|-------------|---------|-----|
| Domain Controller 1 | DC1 | 10.10.200.31 | 255.255.255.0 | 10.10.200.1 | 127.0.0.1, 10.10.200.32 |
| Domain Controller 2 | DC2 | 10.10.200.32 | 255.255.255.0 | 10.10.200.1 | 10.10.200.31, 127.0.0.1 |
| Windows Client | win-2022 | 10.10.200.33 | 255.255.255.0 | 10.10.200.1 | 10.10.200.31, 10.10.200.32 |

**Thông tin domain:**
- Domain name: `anhlx.lab`
- NetBIOS name: `ANHLX`
- Forest/Domain Functional Level: Windows Server 2016

### 3. Cài đặt Domain Controller 1 (DC1)

#### 3.1. Cấu hình IP tĩnh

Mở **Server Manager** → **Local Server** → Click vào **Ethernet** → **Properties** → **Internet Protocol Version 4 (TCP/IPv4)**:

- IP Address: `10.10.200.31`
- Subnet Mask: `255.255.255.0`
- Default Gateway: `10.10.200.1`
- Preferred DNS: `127.0.0.1`
- Alternate DNS: `10.10.200.32`

![](/assets/img/2026-04-09-active-directory-windows-server-2022/01.png)

Hoặc cấu hình bằng PowerShell:

```powershell
# Đặt IP tĩnh
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 10.10.200.31 -PrefixLength 24 -DefaultGateway 10.10.200.1

# Đặt DNS
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 127.0.0.1, 10.10.200.32
```

#### 3.2. Đổi hostname

```powershell
Rename-Computer -NewName "DC1" -Restart
```

#### 3.3. Cài đặt AD DS và DNS

Mở **Server Manager** → **Add Roles and Features**:
1. Chọn **Role-based or feature-based installation**
2. Chọn server hiện tại
3. Tick chọn **Active Directory Domain Services** → Add Features
4. Tick chọn **DNS Server** → Add Features
5. Next → Install

![](/assets/img/2026-04-09-active-directory-windows-server-2022/02.png)

![](/assets/img/2026-04-09-active-directory-windows-server-2022/03.png)

![](/assets/img/2026-04-09-active-directory-windows-server-2022/04.png)
Hoặc cài bằng PowerShell:

```powershell
Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
```

#### 3.4. Promote lên Domain Controller

Sau khi cài xong role, click **Promote this server to a domain controller** trong Server Manager:

![](/assets/img/2026-04-09-active-directory-windows-server-2022/05.png)

- Chọn **Add a new forest**
- Root domain name: `anhlx.lab`

![](/assets/img/2026-04-09-active-directory-windows-server-2022/06.png)

- Forest/Domain functional level: **Windows Server 2016**
- Tick **Domain Name System (DNS) server** và **Global Catalog (GC)**
- Đặt **DSRM password**

![](/assets/img/2026-04-09-active-directory-windows-server-2022/07.png)

![](/assets/img/2026-04-09-active-directory-windows-server-2022/08.png)

- NetBIOS domain name: `ANHLX`

![](/assets/img/2026-04-09-active-directory-windows-server-2022/09.png)

- Giữ mặc định đường dẫn Database, Log, SYSVOL

![](/assets/img/2026-04-09-active-directory-windows-server-2022/10.png)

- Next → Install → Server sẽ tự restart

![](/assets/img/2026-04-09-active-directory-windows-server-2022/11.png)

![](/assets/img/2026-04-09-active-directory-windows-server-2022/12.png)

![](/assets/img/2026-04-09-active-directory-windows-server-2022/13.png)

Hoặc dùng PowerShell:

```powershell
Install-ADDSForest `
    -DomainName "anhlx.lab" `
    -DomainNetbiosName "ANHLX" `
    -ForestMode "WinThreshold" `
    -DomainMode "WinThreshold" `
    -InstallDNS:$true `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) `
    -Force:$true
```

Sau khi restart, đăng nhập bằng `ANHLX\Administrator`.

![](/assets/img/2026-04-09-active-directory-windows-server-2022/14.png)

![](/assets/img/2026-04-09-active-directory-windows-server-2022/15.png)

### 4. Cài đặt Domain Controller 2 (DC2)

#### 4.1. Cấu hình IP tĩnh

- IP Address: `10.10.200.32`
- Subnet Mask: `255.255.255.0`
- Default Gateway: `10.10.200.1`
- Preferred DNS: `10.10.200.31`
- Alternate DNS: `127.0.0.1`

![](/assets/img/2026-04-09-active-directory-windows-server-2022/16.png)

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 10.10.200.32 -PrefixLength 24 -DefaultGateway 10.10.200.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 10.10.200.31, 127.0.0.1
```

#### 4.2. Đổi hostname

```powershell
Rename-Computer -NewName "DC2" -Restart
```

#### 4.3. Cài đặt AD DS và DNS

```powershell
Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
```

![](/assets/img/2026-04-09-active-directory-windows-server-2022/17.png)

#### 4.4. Promote lên Domain Controller (Additional DC)

Click **Promote this server to a domain controller**:

- Chọn **Add a domain controller to an existing domain**
- Domain: `anhlx.lab`
- Credentials: `ANHLX\Administrator`

![](/assets/img/2026-04-09-active-directory-windows-server-2022/18.png)

- Tick **DNS server** và **Global Catalog**
- Đặt **DSRM password**

![](/assets/img/2026-04-09-active-directory-windows-server-2022/19.png)

![](/assets/img/2026-04-09-active-directory-windows-server-2022/20.png)

- Chọn **Replicate from**: `DC1.anhlx.lab`

![](/assets/img/2026-04-09-active-directory-windows-server-2022/21.png)

- Giữ mặc định đường dẫn → Install → Restart

![](/assets/img/2026-04-09-active-directory-windows-server-2022/22.png)

PowerShell:

```powershell
Install-ADDSDomainController `
    -DomainName "anhlx.lab" `
    -Credential (Get-Credential ANHLX\Administrator) `
    -InstallDNS:$true `
    -ReplicationSourceDC "DC1.anhlx.lab" `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) `
    -Force:$true
```

![](/assets/img/2026-04-09-active-directory-windows-server-2022/23.png)

### 5. Join domain từ Windows Client

#### 5.1. Cấu hình IP tĩnh

- IP Address: `10.10.200.33`
- Subnet Mask: `255.255.255.0`
- Default Gateway: `10.10.200.1`
- Preferred DNS: `10.10.200.31`
- Alternate DNS: `10.10.200.32`

![](/assets/img/2026-04-09-active-directory-windows-server-2022/24.png)

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 10.10.200.33 -PrefixLength 24 -DefaultGateway 10.10.200.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 10.10.200.31, 10.10.200.32
```

#### 5.2. Join domain

**Cách 1: GUI**

1. Mở **System Properties** → **Computer Name** → **Change**
2. Chọn **Domain**, nhập `anhlx.lab`
3. Nhập credentials `ANHLX\Administrator`
4. Restart máy

![](/assets/img/2026-04-09-active-directory-windows-server-2022/25.png)

![](/assets/img/2026-04-09-active-directory-windows-server-2022/26.png)

**Cách 2: PowerShell**

```powershell
Add-Computer -DomainName "anhlx.lab" -Credential (Get-Credential ANHLX\Administrator) -Restart
```

Sau khi restart, có thể đăng nhập bằng domain account: `ANHLX\Administrator` hoặc `administrator@anhlx.lab`.

![](/assets/img/2026-04-09-active-directory-windows-server-2022/27.png)

![](/assets/img/2026-04-09-active-directory-windows-server-2022/28.png)

![](/assets/img/2026-04-09-active-directory-windows-server-2022/29.png)

### 6. Quản lý Active Directory

#### 6.1. Tạo Organizational Unit (OU)

Trên DC1, mở **Active Directory Users and Computers** (dsa.msc):

1. Right-click domain `anhlx.lab` → **New** → **Organizational Unit**
2. Tạo các OU theo cấu trúc:
   - `ANHLX-lab-Users`
   - `ANHLX-lab-Computers`
   - `ANHLX-lab-Groups`

```powershell
New-ADOrganizationalUnit -Name "ANHLX-lab-Users" -Path "DC=anhlx,DC=lab"
New-ADOrganizationalUnit -Name "ANHLX-lab-Computers" -Path "DC=anhlx,DC=lab"
New-ADOrganizationalUnit -Name "ANHLX-lab-Groups" -Path "DC=anhlx,DC=lab"
```

#### 6.2. Tạo User

```powershell
New-ADUser `
    -Name "Nguyen Van A" `
    -SamAccountName "nguyenvana" `
    -UserPrincipalName "nguyenvana@anhlx.lab" `
    -Path "OU=ANHLX-lab-Users,DC=anhlx,DC=lab" `
    -AccountPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) `
    -Enabled $true
```

![](/assets/img/2026-04-09-active-directory-windows-server-2022/30.png)

#### 6.3. Kiểm tra Replication giữa 2 DC

Kiểm tra replication hoạt động bình thường giữa DC1 và DC2:

```powershell
# Kiểm tra trạng thái replication
repadmin /replsummary

# Kiểm tra replication partners
repadmin /showrepl

# Force sync replication
repadmin /syncall /AdeP
```

![](/assets/img/2026-04-09-active-directory-windows-server-2022/31.png)

![](/assets/img/2026-04-09-active-directory-windows-server-2022/32.png)

Kiểm tra DNS:

```powershell
# Kiểm tra DNS records
nslookup anhlx.lab 10.10.200.31
nslookup anhlx.lab 10.10.200.32

# Kiểm tra SRV records
nslookup -type=srv _ldap._tcp.anhlx.lab
```

![](/assets/img/2026-04-09-active-directory-windows-server-2022/33.png)
