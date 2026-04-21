---
title: "Lab Windows Server 2022 — Active Directory 2 DC + Exchange 2019 DAG cho doanh nghiệp"
categories:
- System
- Windows
- Infrastructure

feature_image: "../assets/postbanner.jpg"
feature_text: |
  ### Xây dựng hạ tầng email doanh nghiệp on-premise hoàn chỉnh: 2 Domain Controller Windows Server 2022, Exchange 2019 DAG (Database Availability Group) với HA mailbox trên VMware ESXi
---

### Mục lục

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Thiết kế Lab](#2-thiết-kế-lab)
  - [2.1. Topology](#21-topology)
  - [2.2. VM Plan](#22-vm-plan)
- [3. Cài đặt Domain Controller đầu tiên (DC01)](#3-cài-đặt-domain-controller-đầu-tiên-dc01)
  - [3.1. Chuẩn bị VM](#31-chuẩn-bị-vm)
  - [3.2. Tạo Active Directory Forest](#32-tạo-active-directory-forest)
  - [3.3. Cấu hình DNS](#33-cấu-hình-dns)
- [4. Cài đặt Domain Controller thứ hai (DC02)](#4-cài-đặt-domain-controller-thứ-hai-dc02)
  - [4.1. Join domain và promote thành DC](#41-join-domain-và-promote-thành-dc)
  - [4.2. Kiểm tra replication AD](#42-kiểm-tra-replication-ad)
  - [4.3. Chuyển FSMO roles](#43-chuyển-fsmo-roles)
- [5. Chuẩn bị cho Exchange 2019](#5-chuẩn-bị-cho-exchange-2019)
  - [5.1. Tạo OU và Service Account](#51-tạo-ou-và-service-account)
  - [5.2. Cài prerequisites trên EX01 và EX02](#52-cài-prerequisites-trên-ex01-và-ex02)
  - [5.3. Extend AD Schema cho Exchange](#53-extend-ad-schema-cho-exchange)
- [6. Cài đặt Exchange 2019 trên EX01](#6-cài-đặt-exchange-2019-trên-ex01)
- [7. Cài đặt Exchange 2019 trên EX02](#7-cài-đặt-exchange-2019-trên-ex02)
- [8. Cấu hình DAG (Database Availability Group)](#8-cấu-hình-dag-database-availability-group)
  - [8.1. Tạo DAG](#81-tạo-dag)
  - [8.2. Thêm node vào DAG](#82-thêm-node-vào-dag)
  - [8.3. Tạo Mailbox Database và nhân bản](#83-tạo-mailbox-database-và-nhân-bản)
- [9. Cấu hình Email Flow](#9-cấu-hình-email-flow)
  - [9.1. Send Connector](#91-send-connector)
  - [9.2. Receive Connector](#92-receive-connector)
  - [9.3. DNS MX Record](#93-dns-mx-record)
- [10. Tạo Mailbox và kiểm tra](#10-tạo-mailbox-và-kiểm-tra)
- [11. Kiểm tra DAG Failover](#11-kiểm-tra-dag-failover)

---

## 1. Giới thiệu

Hạ tầng email doanh nghiệp on-premise chuẩn thường gồm:

- **Active Directory** với tối thiểu 2 DC — đảm bảo authentication không bị gián đoạn khi 1 DC lỗi
- **Exchange Server** với **DAG (Database Availability Group)** — nhân bản Mailbox Database real-time giữa các node, failover tự động khi 1 EX server bị lỗi

Bài lab này xây dựng đầy đủ stack trên 1 ESXi host:

| | Production thực tế | Lab này |
|---|---|---|
| Domain Controller | ≥2 DC, multi-site | 2 DC (DC01 primary, DC02 standby) |
| Exchange HA | DAG ≥3 node + FSW | DAG 2 node + File Share Witness trên DC01 |
| OS | Windows Server 2019/2022 | Windows Server 2022 |
| Exchange | Exchange 2019 CU latest | Exchange 2019 CU15 |

**DAG với 2 node** — File Share Witness (FSW) đặt trên DC01 đóng vai trò tie-breaker khi 2 EX node không đồng thuận được quorum. Đây là cấu hình tối thiểu được Microsoft hỗ trợ cho production.

---

## 2. Thiết kế Lab

### 2.1. Topology

```
                    Internet
                       │
                  pfSense (GW)
                  10.10.200.1
                       │
          ─────────────┼─────────────────────
                    VLAN 200 (10.10.200.0/24)
          ─────────────┼─────────────────────
          │            │            │           │
       DC01          DC02         EX01         EX02
  10.10.200.31  10.10.200.32  10.10.200.33  10.10.200.34
  Primary DC    Secondary DC  Exchange MBX  Exchange MBX
  DNS Primary   DNS Secondary  DAG Node 1    DAG Node 2
  FSW for DAG
                                    │            │
                              DAG Replication Network
                         ─────────────────────────────
                              (dùng chung VLAN 200
                               hoặc thêm NIC riêng)

  VIP (Load Balancer / DNS Round-Robin):
  mail.company.local → EX01 + EX02 (Round-Robin DNS)
  autodiscover.company.local → EX01 + EX02
```

### 2.2. VM Plan

| VM | OS | Role | IP | vCPU | RAM | Disk |
|---|---|---|---|---|---|---|
| DC01 | Windows Server 2022 | Primary DC, DNS, FSW | 10.10.200.31 | 2 | 4 GB | 60 GB |
| DC02 | Windows Server 2022 | Secondary DC, DNS | 10.10.200.32 | 2 | 4 GB | 60 GB |
| EX01 | Windows Server 2022 | Exchange 2019 Mailbox, DAG node 1 | 10.10.200.33 | 4 | 16 GB | 150 GB |
| EX02 | Windows Server 2022 | Exchange 2019 Mailbox, DAG node 2 | 10.10.200.34 | 4 | 16 GB | 150 GB |

> **Disk layout cho Exchange VM:**
> - `C:` 80 GB — OS + Exchange binaries
> - `D:` 70 GB — Mailbox Database + Transaction Logs
>
> Exchange 2019 yêu cầu **tối thiểu 8 GB RAM**, 16 GB để lab thoải mái.

**Domain info:**
- **Forest/Domain:** `company.local`
- **NetBIOS:** `COMPANY`
- **Exchange Organization:** `Company-Mail`
- **Admin account:** `COMPANY\Administrator`

---

## 3. Cài đặt Domain Controller đầu tiên (DC01)

### 3.1. Chuẩn bị VM

Sau khi cài Windows Server 2022 (Desktop Experience), thực hiện trên **DC01**:

```powershell
# Đặt hostname
Rename-Computer -NewName "DC01" -Restart

# Sau khi restart — cấu hình IP tĩnh (thay "Ethernet0" bằng tên NIC thực tế)
$NIC = Get-NetAdapter | Where-Object {$_.Status -eq "Up"} | Select-Object -First 1
New-NetIPAddress -InterfaceAlias $NIC.Name `
  -IPAddress 10.10.200.31 `
  -PrefixLength 24 `
  -DefaultGateway 10.10.200.1

Set-DnsClientServerAddress -InterfaceAlias $NIC.Name `
  -ServerAddresses 127.0.0.1, 10.10.200.32

# Tắt Windows Firewall (lab only)
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

# Tắt IE Enhanced Security (để dễ download)
$AdminKey = "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}"
Set-ItemProperty -Path $AdminKey -Name "IsInstalled" -Value 0
```

### 3.2. Tạo Active Directory Forest

```powershell
# Cài AD DS role
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Tạo Forest mới — sẽ tự restart sau khi xong
Install-ADDSForest `
  -DomainName "company.local" `
  -DomainNetBiosName "COMPANY" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns `
  -DatabasePath "C:\Windows\NTDS" `
  -LogPath "C:\Windows\NTDS" `
  -SysvolPath "C:\Windows\SYSVOL" `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "Admin@2026!" -AsPlainText -Force) `
  -Force
# VM tự restart
```

Sau khi restart, login bằng `COMPANY\Administrator`:

```powershell
# Kiểm tra AD đã lên
Get-ADDomain
Get-ADForest

# Kiểm tra DNS
Resolve-DnsName company.local
```

### 3.3. Cấu hình DNS

Exchange yêu cầu DNS internal hoạt động chính xác:

```powershell
# Tạo Reverse Lookup Zone cho 10.10.200.0/24
Add-DnsServerPrimaryZone `
  -NetworkID "10.10.200.0/24" `
  -ReplicationScope "Forest"

# Tạo DNS records cho Exchange
Add-DnsServerResourceRecordA -ZoneName "company.local" `
  -Name "mail" -IPv4Address "10.10.200.33"
Add-DnsServerResourceRecordA -ZoneName "company.local" `
  -Name "autodiscover" -IPv4Address "10.10.200.33"
# Round-robin — thêm EX02
Add-DnsServerResourceRecordA -ZoneName "company.local" `
  -Name "mail" -IPv4Address "10.10.200.34"
Add-DnsServerResourceRecordA -ZoneName "company.local" `
  -Name "autodiscover" -IPv4Address "10.10.200.34"

# MX record cho internal mail routing
Add-DnsServerResourceRecordMX -ZoneName "company.local" `
  -Name "@" -MailExchange "mail.company.local" -Preference 10

# PTR records
Add-DnsServerResourceRecordPtr -ZoneName "200.10.10.in-addr.arpa" `
  -Name "33" -PtrDomainName "ex01.company.local."
Add-DnsServerResourceRecordPtr -ZoneName "200.10.10.in-addr.arpa" `
  -Name "34" -PtrDomainName "ex02.company.local."
```

<img src="../assets/img/2026-04-22-windows-ad-exchange-lab/01-ad-dns.png"/>

---

## 4. Cài đặt Domain Controller thứ hai (DC02)

### 4.1. Join domain và promote thành DC

**Thực hiện trên DC02:**

```powershell
# Đặt hostname và IP
Rename-Computer -NewName "DC02" -Restart

# Sau restart — cấu hình IP, DNS trỏ về DC01
$NIC = Get-NetAdapter | Where-Object {$_.Status -eq "Up"} | Select-Object -First 1
New-NetIPAddress -InterfaceAlias $NIC.Name `
  -IPAddress 10.10.200.32 -PrefixLength 24 -DefaultGateway 10.10.200.1
Set-DnsClientServerAddress -InterfaceAlias $NIC.Name `
  -ServerAddresses 10.10.200.31, 127.0.0.1

Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

# Join domain trước
Add-Computer -DomainName "company.local" `
  -Credential (Get-Credential) `
  -Restart
# Nhập: COMPANY\Administrator / Admin@2026!
```

Sau khi restart, login bằng `COMPANY\Administrator`:

```powershell
# Promote DC02 thành Additional Domain Controller
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

Install-ADDSDomainController `
  -DomainName "company.local" `
  -InstallDns `
  -Credential (Get-Credential) `
  -DatabasePath "C:\Windows\NTDS" `
  -LogPath "C:\Windows\NTDS" `
  -SysvolPath "C:\Windows\SYSVOL" `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "Admin@2026!" -AsPlainText -Force) `
  -Force
# VM tự restart
```

### 4.2. Kiểm tra replication AD

```powershell
# Kiểm tra replication partners
Get-ADReplicationPartnerMetadata -Target DC01 -Scope Domain

# Force sync từ DC01 sang DC02
repadmin /syncall DC02 /AdeP

# Kiểm tra replication status (không có lỗi là OK)
repadmin /replsummary
repadmin /showrepl
```

<img src="../assets/img/2026-04-22-windows-ad-exchange-lab/02-ad-replication.png"/>

### 4.3. Chuyển FSMO roles

Để cân bằng tải, phân chia FSMO roles giữa 2 DC:

```powershell
# Xem DC nào đang giữ các FSMO roles
netdom query fsmo

# DC01 giữ: Schema Master, Domain Naming Master, RID Master, PDC Emulator
# Chuyển Infrastructure Master sang DC02
Move-ADDirectoryServerOperationMasterRole `
  -Identity "DC02" `
  -OperationMasterRole InfrastructureMaster `
  -Force

# Kiểm tra lại
netdom query fsmo
```

> **Quy tắc FSMO trong lab 2 DC:**
> - DC01: Schema Master, Domain Naming Master, RID Master, PDC Emulator (4 roles)
> - DC02: Infrastructure Master (1 role)
>
> Infrastructure Master **không được** đặt trên DC cũng là Global Catalog Server (GC) trừ khi tất cả DC đều là GC — trong lab 2 DC cả 2 đều là GC nên đặt đâu cũng được.

---

## 5. Chuẩn bị cho Exchange 2019

### 5.1. Tạo OU và Service Account

```powershell
# Tạo OU cấu trúc cho doanh nghiệp
New-ADOrganizationalUnit -Name "Company" -Path "DC=company,DC=local"
New-ADOrganizationalUnit -Name "Users" -Path "OU=Company,DC=company,DC=local"
New-ADOrganizationalUnit -Name "Computers" -Path "OU=Company,DC=company,DC=local"
New-ADOrganizationalUnit -Name "Service Accounts" -Path "OU=Company,DC=company,DC=local"
New-ADOrganizationalUnit -Name "Groups" -Path "OU=Company,DC=company,DC=local"

# Tạo test users cho mailbox
New-ADUser -Name "Nguyen Van A" `
  -GivenName "Van A" -Surname "Nguyen" `
  -SamAccountName "nguyenvana" `
  -UserPrincipalName "nguyenvana@company.local" `
  -Path "OU=Users,OU=Company,DC=company,DC=local" `
  -AccountPassword (ConvertTo-SecureString "User@2026!" -AsPlainText -Force) `
  -Enabled $true

New-ADUser -Name "Tran Thi B" `
  -GivenName "Thi B" -Surname "Tran" `
  -SamAccountName "tranthib" `
  -UserPrincipalName "tranthib@company.local" `
  -Path "OU=Users,OU=Company,DC=company,DC=local" `
  -AccountPassword (ConvertTo-SecureString "User@2026!" -AsPlainText -Force) `
  -Enabled $true
```

### 5.2. Cài prerequisites trên EX01 và EX02

**Thực hiện trên EX01 và EX02 (lặp lại cho từng máy):**

Trước tiên join domain:

```powershell
# Trên EX01
Rename-Computer -NewName "EX01" -Restart
# Sau restart
$NIC = Get-NetAdapter | Where-Object {$_.Status -eq "Up"} | Select-Object -First 1
New-NetIPAddress -InterfaceAlias $NIC.Name `
  -IPAddress 10.10.200.33 -PrefixLength 24 -DefaultGateway 10.10.200.1
Set-DnsClientServerAddress -InterfaceAlias $NIC.Name `
  -ServerAddresses 10.10.200.31, 10.10.200.32
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

Add-Computer -DomainName "company.local" `
  -Credential (Get-Credential) -Restart
```

> Trên EX02: thay IP `10.10.200.34`, hostname `EX02`

Sau khi join domain, cài Windows prerequisites:

```powershell
# Cài .NET Framework 4.8 (nếu chưa có)
# Download: https://go.microsoft.com/fwlink/?linkid=2088631
# Restart sau khi cài

# Cài Windows features bắt buộc cho Exchange 2019
Install-WindowsFeature `
  Server-Media-Foundation, NET-Framework-45-Features, `
  RPC-over-HTTP-proxy, RSAT-Clustering, RSAT-Clustering-CmdInterface, `
  RSAT-Clustering-Mgmt, RSAT-Clustering-PowerShell, `
  WAS-Process-Model, Web-Asp-Net45, Web-Basic-Auth, Web-Client-Auth, `
  Web-Digest-Auth, Web-Dir-Browsing, Web-Dyn-Compression, Web-Http-Errors, `
  Web-Http-Logging, Web-Http-Redirect, Web-Http-Tracing, Web-ISAPI-Ext, `
  Web-ISAPI-Filter, Web-Lgcy-Mgmt-Console, Web-Metabase, Web-Mgmt-Console, `
  Web-Mgmt-Service, Web-Net-Ext45, Web-Request-Monitor, Web-Server, `
  Web-Stat-Compression, Web-Static-Content, Web-Windows-Auth, `
  Web-WMI, Windows-Identity-Foundation, RSAT-ADDS `
  -IncludeManagementTools

# Restart bắt buộc
Restart-Computer -Force
```

Sau restart, cài thêm Visual C++ Redistributables:

```powershell
# Visual C++ 2012 x64
# Download: https://www.microsoft.com/download/details.aspx?id=30679
# Visual C++ 2013 x64
# Download: https://aka.ms/highdpimfc2013x64enu
# Cài lần lượt, không cần restart giữa 2 bản

# IIS URL Rewrite Module
# Download: https://www.iis.net/downloads/microsoft/url-rewrite
```

> **Lưu ý:** Tất cả prerequisites phải được cài đầy đủ trước khi chạy Exchange setup. Nếu thiếu, Exchange setup sẽ báo lỗi và không tiếp tục được.

### 5.3. Extend AD Schema cho Exchange

**Chỉ làm 1 lần trên EX01** — chạy với quyền Schema Admin + Enterprise Admin:

```powershell
# Mount Exchange 2019 ISO, giả sử D:\
# Extend Schema (thực hiện lần đầu, trước khi cài Exchange)
D:\Setup.exe /PrepareSchema /IAcceptExchangeServerLicenseTerms_DiagnosticDataOFF

# Prepare AD
D:\Setup.exe /PrepareAD /OrganizationName:"Company-Mail" /IAcceptExchangeServerLicenseTerms_DiagnosticDataOFF

# Prepare tất cả domains trong forest
D:\Setup.exe /PrepareAllDomains /IAcceptExchangeServerLicenseTerms_DiagnosticDataOFF
```

Kiểm tra schema đã được extend:

```powershell
# Trên DC01 — kiểm tra Exchange schema objects
Get-ADObject -SearchBase "CN=ms-Exch-Schema-Version-Pt,CN=Schema,CN=Configuration,DC=company,DC=local" `
  -Filter * -Properties rangeUpper | Select-Object rangeUpper
# Exchange 2019 CU15: rangeUpper = 17003
```

<img src="../assets/img/2026-04-22-windows-ad-exchange-lab/03-exchange-schema.png"/>

---

## 6. Cài đặt Exchange 2019 trên EX01

**Thực hiện trên EX01 (10.10.200.33):**

```powershell
# Tạo thư mục database Exchange trên D:
New-Item -ItemType Directory -Path "D:\ExchangeDB\MBX01-DB01"
New-Item -ItemType Directory -Path "D:\ExchangeDB\MBX01-DB01\Logs"

# Chạy Exchange Setup GUI
D:\Setup.exe

# Hoặc silent install qua command line:
D:\Setup.exe /Mode:Install `
  /Role:Mailbox `
  /MdbName:"MBX01-DB01" `
  /DbFilePath:"D:\ExchangeDB\MBX01-DB01\MBX01-DB01.edb" `
  /LogFolderPath:"D:\ExchangeDB\MBX01-DB01\Logs" `
  /IAcceptExchangeServerLicenseTerms_DiagnosticDataOFF
```

> **Thời gian cài đặt:** Exchange 2019 mất khoảng **45-90 phút** để hoàn thành. Quá trình gồm 15 steps — bình thường để một số step chạy lâu.

Sau khi cài xong, kiểm tra services:

```powershell
# Kiểm tra Exchange services đang chạy
Get-Service | Where-Object {$_.DisplayName -like "Microsoft Exchange*"} | `
  Select-Object DisplayName, Status | Sort-Object DisplayName

# Kiểm tra Exchange Management Shell
Add-PSSnapin Microsoft.Exchange.Management.PowerShell.SnapIn
Get-ExchangeServer
```

Cấu hình Exchange URLs (dùng VIP / DNS Round-Robin `mail.company.local`):

```powershell
# Mở Exchange Management Shell (EMS) với quyền Admin
$ServerFQDN = "ex01.company.local"
$ExternalURL = "https://mail.company.local"
$InternalURL = "https://mail.company.local"

# OWA
Set-OwaVirtualDirectory -Identity "$ServerFQDN\owa (Default Web Site)" `
  -InternalUrl "$InternalURL/owa" -ExternalUrl "$ExternalURL/owa"

# ECP (Exchange Control Panel)
Set-EcpVirtualDirectory -Identity "$ServerFQDN\ecp (Default Web Site)" `
  -InternalUrl "$InternalURL/ecp" -ExternalUrl "$ExternalURL/ecp"

# EWS (Exchange Web Services)
Set-WebServicesVirtualDirectory -Identity "$ServerFQDN\EWS (Default Web Site)" `
  -InternalUrl "$InternalURL/EWS/Exchange.asmx" `
  -ExternalUrl "$ExternalURL/EWS/Exchange.asmx"

# ActiveSync (Mobile)
Set-ActiveSyncVirtualDirectory -Identity "$ServerFQDN\Microsoft-Server-ActiveSync (Default Web Site)" `
  -InternalUrl "$InternalURL/Microsoft-Server-ActiveSync" `
  -ExternalUrl "$ExternalURL/Microsoft-Server-ActiveSync"

# Autodiscover
Set-ClientAccessService -Identity EX01 `
  -AutoDiscoverServiceInternalUri "https://autodiscover.company.local/Autodiscover/Autodiscover.xml"
```

<img src="../assets/img/2026-04-22-windows-ad-exchange-lab/04-exchange-install.png"/>

---

## 7. Cài đặt Exchange 2019 trên EX02

**Thực hiện trên EX02 (10.10.200.34) — giống EX01:**

```powershell
# Tạo thư mục database
New-Item -ItemType Directory -Path "D:\ExchangeDB\MBX02-DB01"
New-Item -ItemType Directory -Path "D:\ExchangeDB\MBX02-DB01\Logs"

# Cài Exchange (KHÔNG chạy PrepareSchema/PrepareAD lại)
D:\Setup.exe /Mode:Install `
  /Role:Mailbox `
  /MdbName:"MBX02-DB01" `
  /DbFilePath:"D:\ExchangeDB\MBX02-DB01\MBX02-DB01.edb" `
  /LogFolderPath:"D:\ExchangeDB\MBX02-DB01\Logs" `
  /IAcceptExchangeServerLicenseTerms_DiagnosticDataOFF
```

Cấu hình URLs cho EX02 (giống EX01):

```powershell
$ServerFQDN = "ex02.company.local"
$ExternalURL = "https://mail.company.local"
$InternalURL = "https://mail.company.local"

Set-OwaVirtualDirectory -Identity "$ServerFQDN\owa (Default Web Site)" `
  -InternalUrl "$InternalURL/owa" -ExternalUrl "$ExternalURL/owa"
Set-EcpVirtualDirectory -Identity "$ServerFQDN\ecp (Default Web Site)" `
  -InternalUrl "$InternalURL/ecp" -ExternalUrl "$ExternalURL/ecp"
Set-WebServicesVirtualDirectory -Identity "$ServerFQDN\EWS (Default Web Site)" `
  -InternalUrl "$InternalURL/EWS/Exchange.asmx" `
  -ExternalUrl "$ExternalURL/EWS/Exchange.asmx"
Set-ActiveSyncVirtualDirectory -Identity "$ServerFQDN\Microsoft-Server-ActiveSync (Default Web Site)" `
  -InternalUrl "$InternalURL/Microsoft-Server-ActiveSync" `
  -ExternalUrl "$ExternalURL/Microsoft-Server-ActiveSync"
Set-ClientAccessService -Identity EX02 `
  -AutoDiscoverServiceInternalUri "https://autodiscover.company.local/Autodiscover/Autodiscover.xml"
```

---

## 8. Cấu hình DAG (Database Availability Group)

DAG là tính năng HA của Exchange — nhân bản Mailbox Database giữa các node qua log shipping, failover tự động khi 1 node lỗi. Với 2 node, cần **File Share Witness (FSW)** trên DC01 làm tie-breaker.

### 8.1. Tạo DAG

**Thực hiện trên EX01, mở Exchange Management Shell:**

```powershell
# Tạo thư mục FSW trên DC01
# (Chạy lệnh sau trên DC01 trước)
# New-Item -ItemType Directory -Path "C:\DAGWitness\DAG-Company"
# Cấp quyền cho Exchange Trusted Subsystem
# icacls "C:\DAGWitness\DAG-Company" /grant "COMPANY\Exchange Trusted Subsystem:(OI)(CI)F"

# Tạo DAG từ EX01
New-DatabaseAvailabilityGroup `
  -Name "DAG-Company" `
  -WitnessServer "DC01.company.local" `
  -WitnessDirectory "C:\DAGWitness\DAG-Company" `
  -DatabaseAvailabilityGroupIpAddresses 10.10.200.35

# Kiểm tra DAG vừa tạo
Get-DatabaseAvailabilityGroup -Identity "DAG-Company" -Status | `
  Format-List Name, DatabaseAvailabilityGroupIpAddresses, WitnessServer, OperationalServers
```

> **DAG IP (10.10.200.35):** Đây là IP của DAG cluster network — không phải mailbox access. Client kết nối đến `mail.company.local` (DNS Round-Robin) chứ không phải DAG IP.

### 8.2. Thêm node vào DAG

```powershell
# Thêm EX01 vào DAG
Add-DatabaseAvailabilityGroupServer -Identity "DAG-Company" -MailboxServer EX01

# Thêm EX02 vào DAG
Add-DatabaseAvailabilityGroupServer -Identity "DAG-Company" -MailboxServer EX02

# Kiểm tra member
Get-DatabaseAvailabilityGroup -Identity "DAG-Company" -Status | `
  Select-Object -ExpandProperty Servers

# Kiểm tra Cluster quorum
Get-DatabaseAvailabilityGroup -Identity "DAG-Company" -Status | `
  Select-Object Name, OperationalServers, PrimaryActiveManager, QuorumType
```

<img src="../assets/img/2026-04-22-windows-ad-exchange-lab/05-dag-members.png"/>

### 8.3. Tạo Mailbox Database và nhân bản

```powershell
# Kiểm tra databases hiện có (mặc định đã tạo khi cài)
Get-MailboxDatabase | Select-Object Name, Server, EdbFilePath

# Rename databases cho rõ ràng
Get-MailboxDatabase -Server EX01 | Set-MailboxDatabase -Name "DB-EX01"
Get-MailboxDatabase -Server EX02 | Set-MailboxDatabase -Name "DB-EX02"

# Thêm database copy — DB-EX01 nhân bản sang EX02
Add-MailboxDatabaseCopy -Identity "DB-EX01" -MailboxServer EX02 -ActivationPreference 2

# Thêm database copy — DB-EX02 nhân bản sang EX01
Add-MailboxDatabaseCopy -Identity "DB-EX02" -MailboxServer EX01 -ActivationPreference 2

# Kiểm tra replication status (chờ đến khi ContentIndexState = Healthy)
Get-MailboxDatabaseCopyStatus * | `
  Select-Object Name, Status, CopyQueueLength, ReplayQueueLength, ContentIndexState
```

Output mong đợi:

```
Name                      Status      CopyQueueLength  ReplayQueueLength  ContentIndexState
----                      ------      ---------------  -----------------  -----------------
DB-EX01\EX01              Mounted     0                0                  Healthy
DB-EX01\EX02              Healthy     0                0                  Healthy
DB-EX02\EX02              Mounted     0                0                  Healthy
DB-EX02\EX01              Healthy     0                0                  Healthy
```

<img src="../assets/img/2026-04-22-windows-ad-exchange-lab/06-dag-replication.png"/>

---

## 9. Cấu hình Email Flow

### 9.1. Send Connector

Send Connector cho phép Exchange gửi email ra domain bên ngoài:

```powershell
# Tạo Send Connector gửi email ra internet
New-SendConnector -Name "Internet Send Connector" `
  -Usage Internet `
  -AddressSpaces "*" `
  -DNSRoutingEnabled $true `
  -SourceTransportServers EX01, EX02 `
  -RequireTLS $false `
  -Fqdn "mail.company.local"

# Kiểm tra
Get-SendConnector | Select-Object Name, AddressSpaces, SourceTransportServers, Enabled
```

### 9.2. Receive Connector

```powershell
# Kiểm tra Receive Connectors mặc định
Get-ReceiveConnector | Select-Object Name, Server, Bindings, RemoteIPRanges

# Tạo Receive Connector cho relay nội bộ (các ứng dụng gửi mail qua Exchange)
New-ReceiveConnector -Name "Internal Relay" `
  -Server EX01 `
  -TransportRole FrontendTransport `
  -Bindings "0.0.0.0:25" `
  -RemoteIPRanges "10.10.200.0/24" `
  -PermissionGroups AnonymousUsers `
  -AuthMechanism None

# Cho phép Anonymous relay từ subnet nội bộ
Get-ReceiveConnector "EX01\Internal Relay" | `
  Add-ADPermission -User "NT AUTHORITY\ANONYMOUS LOGON" `
  -ExtendedRights "Ms-Exch-SMTP-Accept-Any-Recipient"
```

### 9.3. DNS MX Record

```powershell
# Trên DC01 — tạo MX record cho company.local
Add-DnsServerResourceRecordMX -ZoneName "company.local" `
  -Name "@" -MailExchange "mail.company.local" -Preference 10

# Kiểm tra email routing
Test-Mailflow -TargetMailboxServer EX02
# Kết quả: IsAchieved = True, LatencyInMilliseconds < 1000
```

---

## 10. Tạo Mailbox và kiểm tra

```powershell
# Enable mailbox cho users đã tạo ở bước 5.1
Enable-Mailbox -Identity "nguyenvana" -Database "DB-EX01"
Enable-Mailbox -Identity "tranthib" -Database "DB-EX02"

# Tạo user mới với mailbox luôn
New-Mailbox -Name "Le Van C" `
  -Alias "levanc" `
  -UserPrincipalName "levanc@company.local" `
  -SamAccountName "levanc" `
  -Password (ConvertTo-SecureString "User@2026!" -AsPlainText -Force) `
  -FirstName "Van C" -LastName "Le" `
  -Database "DB-EX01"

# Kiểm tra mailbox
Get-Mailbox | Select-Object DisplayName, Alias, Database, ServerName

# Test gửi email nội bộ
Send-MailMessage -SmtpServer "10.10.200.33" `
  -From "nguyenvana@company.local" `
  -To "tranthib@company.local" `
  -Subject "Test email" -Body "Hello from Exchange DAG lab"

# Kiểm tra message queue
Get-Queue -Server EX01
Get-Queue -Server EX02
```

**Truy cập OWA (Outlook Web Access):**

Truy cập `https://mail.company.local/owa` hoặc `http://10.10.200.33/owa`:

- Login: `COMPANY\nguyenvana` / `User@2026!`
- Gửi email thử đến `tranthib@company.local`

<img src="../assets/img/2026-04-22-windows-ad-exchange-lab/07-owa.png"/>

---

## 11. Kiểm tra DAG Failover

Test failover bằng cách dừng Exchange services trên EX01:

```powershell
# Trên EX01 — xem DB nào đang mounted
Get-MailboxDatabaseCopyStatus * | Where-Object {$_.Status -eq "Mounted"} | `
  Select-Object Name, Server

# Suspend EX01 khỏi DAG (mô phỏng maintenance)
Suspend-MailboxDatabaseCopy -Identity "DB-EX01\EX01" -SuspendComment "Testing failover"

# Hoặc shutdown toàn bộ Exchange services
Stop-Service MSExchangeIS, MSExchangeTransport -Force
```

Kiểm tra DB đã tự failover sang EX02:

```powershell
# Chạy trên EX02
Get-MailboxDatabaseCopyStatus * | Select-Object Name, Status, ActiveServerName

# Output mong đợi:
# DB-EX01\EX02   Mounted   EX02    ← DB-EX01 đã failover sang EX02
# DB-EX02\EX02   Mounted   EX02
```

<img src="../assets/img/2026-04-22-windows-ad-exchange-lab/08-dag-failover.png"/>

OWA vẫn accessible qua `mail.company.local` → DNS Round-Robin tự route sang EX02 nếu EX01 không respond.

**Khôi phục EX01:**

```powershell
# Start lại Exchange services trên EX01
Start-Service MSExchangeIS, MSExchangeTransport

# Resume database copy
Resume-MailboxDatabaseCopy -Identity "DB-EX01\EX01"

# Chờ reseed hoàn thành (CopyQueueLength về 0)
Get-MailboxDatabaseCopyStatus * | Select-Object Name, Status, CopyQueueLength

# Switchback DB-EX01 về EX01 (tùy chọn)
Move-ActiveMailboxDatabase -Identity "DB-EX01" -ActivateOnServer EX01 -MountDialOverride None
```

---

**Tổng kết Lab:**

| Component | IP | URL |
|---|---|---|
| DC01 (Primary DC + DNS + FSW) | 10.10.200.31 | — |
| DC02 (Secondary DC + DNS) | 10.10.200.32 | — |
| EX01 (Exchange Mailbox + DAG node 1) | 10.10.200.33 | — |
| EX02 (Exchange Mailbox + DAG node 2) | 10.10.200.34 | — |
| OWA / ECP / Autodiscover | Round-Robin DNS | `https://mail.company.local/owa` |
| DAG Cluster IP | 10.10.200.35 | — |

**Điểm quan trọng:**
- File Share Witness đặt trên DC01 — **không** đặt trên Exchange server
- DAG 2 node cần FSW để có quorum (2/3 votes: EX01 + EX02 + FSW)
- DNS Round-Robin cho `mail.company.local` — client tự retry node còn lại nếu 1 node không respond
- Database copy với `ActivationPreference 2` — node chính luôn được ưu tiên, node kia là standby
