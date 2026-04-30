---
title: "Triển khai Hybrid Identity: AD on-premise đồng bộ lên Microsoft 365 với Entra Connect"
categories:
- System
- Windows Server
- Active Directory
- Microsoft 365
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Lab thực tế: Dựng 2 Domain Controller, đồng bộ AD lên Microsoft Entra ID, sử dụng Exchange Online (M365 Business Basic)
---

Bài lab hướng dẫn triển khai mô hình **Hybrid Identity** chuẩn production cho doanh nghiệp vừa và nhỏ (SME): 2 VM Windows Server 2022 làm Domain Controller, đồng bộ tài khoản AD lên **Microsoft Entra ID** bằng **Microsoft Entra Connect**, sau đó **đăng nhập Outlook Web / M365 bằng tài khoản AD đã sync**.

> **Mục tiêu lab**: Tạo user trong AD on-premise → sync lên Entra ID → login `user1@anhle.com.vn` vào Outlook Web thành công. Lab này **không cấu hình mail flow** (không thêm MX record), chỉ tập trung vào phần **Identity Sync** và **xác thực cloud**.

- [1. Giới thiệu và mô hình triển khai](#1-giới-thiệu-và-mô-hình-triển-khai)
  - [1.1. Tại sao chọn mô hình này?](#11-tại-sao-chọn-mô-hình-này)
  - [1.2. Phân bổ vai trò 2 VM](#12-phân-bổ-vai-trò-2-vm)
  - [1.3. Trải nghiệm người dùng (User Experience)](#13-trải-nghiệm-người-dùng-user-experience)
- [2. Thiết kế lab](#2-thiết-kế-lab)
  - [2.1. Topology](#21-topology)
  - [2.2. Kế hoạch IP và VM](#22-kế-hoạch-ip-và-vm)
- [3. Chuẩn bị M365 Tenant](#3-chuẩn-bị-m365-tenant)
  - [3.1. Đăng ký M365 Business Basic Trial](#31-đăng-ký-m365-business-basic-trial)
  - [3.2. Verify domain anhle.com.vn](#32-verify-domain-anhlecomvn)
- [4. Cài đặt DC1 — Primary Domain Controller](#4-cài-đặt-dc1--primary-domain-controller)
  - [4.1. Cấu hình IP tĩnh và hostname](#41-cấu-hình-ip-tĩnh-và-hostname)
  - [4.2. Cài đặt AD DS và DNS](#42-cài-đặt-ad-ds-và-dns)
  - [4.3. Promote lên Domain Controller (New Forest)](#43-promote-lên-domain-controller-new-forest)
  - [4.4. Thêm UPN Suffix anhle.com.vn](#44-thêm-upn-suffix-anhlecomvn)
  - [4.5. Tạo OU Structure và Users](#45-tạo-ou-structure-và-users)
  - [4.6. Cài Exchange Management Tools](#46-cài-exchange-management-tools)
- [5. Cài đặt DC2 — Additional DC + Microsoft Entra Connect](#5-cài-đặt-dc2--additional-dc--microsoft-entra-connect)
  - [5.1. Cấu hình IP tĩnh, hostname và join domain](#51-cấu-hình-ip-tĩnh-hostname-và-join-domain)
  - [5.2. Promote lên Additional Domain Controller](#52-promote-lên-additional-domain-controller)
  - [5.3. Cài đặt Microsoft Entra Connect](#53-cài-đặt-microsoft-entra-connect)
  - [5.4. Cấu hình Password Hash Sync + Seamless SSO](#54-cấu-hình-password-hash-sync--seamless-sso)
  - [5.5. Bật Password Writeback (SSPR)](#55-bật-password-writeback-sspr)
- [6. Gán License và kiểm tra đồng bộ](#6-gán-license-và-kiểm-tra-đồng-bộ)
  - [6.1. Kiểm tra User đã sync lên Entra ID](#61-kiểm-tra-user-đã-sync-lên-entra-id)
  - [6.2. Gán M365 Business Basic License](#62-gán-m365-business-basic-license)
  - [6.3. Đăng nhập Outlook Web bằng tài khoản AD sync](#63-đăng-nhập-outlook-web-bằng-tài-khoản-ad-sync)
- [7. Force Sync và một số lệnh quản trị hay dùng](#7-force-sync-và-một-số-lệnh-quản-trị-hay-dùng)
  - [7.1. Tạo user mới và kiểm tra sync tự động lên Entra ID](#71-tạo-user-mới-và-kiểm-tra-sync-tự-động-lên-entra-id)
- [8. Tổng kết](#8-tổng-kết)

---

### 1. Giới thiệu và mô hình triển khai

#### 1.1. Tại sao chọn mô hình này?

Với các doanh nghiệp vừa và nhỏ (SMB/SME) tại Việt Nam, mô hình kết hợp **Active Directory on-premise + Microsoft 365** là lựa chọn phổ biến nhất vì:

- **Không cần Exchange Server on-premise** — tiết kiệm tài nguyên phần cứng (Exchange tối thiểu 16 GB RAM/VM, chi phí license cao).
- **Mailbox nằm hoàn toàn trên cloud** (Exchange Online) — Microsoft lo hạ tầng, backup, uptime.
- **Quản lý user tập trung tại AD on-premise** — IT vẫn dùng ADUC/ADAC như bình thường, mọi thay đổi tự đồng bộ lên cloud.
- **Single Sign-On**: nhân viên chỉ cần nhớ 1 mật khẩu, dùng cho cả máy tính nội bộ lẫn Outlook/Teams.

#### 1.2. Phân bổ vai trò 2 VM

| Thành phần | VM 1 — DC1 (PDC) | VM 2 — DC2 (ADC) |
|------------|:----------------:|:----------------:|
| **Role AD DS** | ✅ FSMO Roles (PDC Emulator, RID Master, Infrastructure Master, Schema Master, Domain Naming Master) | ✅ Additional DC (Replication) |
| **DNS** | ✅ Primary DNS | ✅ Secondary DNS |
| **Phần mềm thêm** | Exchange Management Tools | Microsoft Entra Connect |
| **Tại sao chia vậy?** | Tách biệt công cụ quản lý mail attributes khỏi bộ đồng bộ cloud | Nếu 1 máy bảo trì, máy kia vẫn cho nhân viên login Windows được |
| **IP** | `10.10.200.11` | `10.10.200.12` |

> **Exchange Management Tools** (không phải Exchange Server) chỉ là bộ console/PowerShell module để quản lý thuộc tính mail của user trong AD (EmailAddress, TargetAddress, ProxyAddresses...). Nhẹ, không tốn RAM, cài trên DC được.

#### 1.3. Trải nghiệm người dùng (User Experience)

Trong lab này, **UPN của user trong AD được đặt thẳng là `user1@anhle.com.vn`** (không phải `@anhlx.lab`). Kết quả: người dùng chỉ cần nhớ đúng **1 địa chỉ** cho mọi thứ.

| Tình huống | Thông tin đăng nhập | Giao thức |
|-----------|-------------------|-----------|
| **Login máy tính Windows** (domain joined) | `user1@anhle.com.vn` | Kerberos (DC xác thực qua UPN) |
| **Login Outlook Web / Teams / M365** | `user1@anhle.com.vn` | OAuth 2.0 / Entra ID (cloud) |
| **Mật khẩu** | **Cùng 1 mật khẩu duy nhất** | Password Hash Sync |
| **Đổi mật khẩu** | Đổi trên máy tính → tự đồng bộ lên cloud ~2 phút | Password Writeback |
| **NetBIOS fallback** | `ANHLX\user1` vẫn hoạt động | NTLM |

> **Tại sao login máy tính dùng được `user1@anhle.com.vn`?**  
> Windows xác thực vào domain bằng **UPN (User Principal Name)** — không phải tên domain forest. Vì UPN của user trong AD là `user1@anhle.com.vn`, DC sẽ tra cứu đúng account và cấp Kerberos ticket. Forest name `anhlx.lab` chỉ dùng nội bộ cho replication và DNS — người dùng cuối không bao giờ cần biết đến nó.

---

### 2. Thiết kế lab

#### 2.1. Topology

![Topology](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/00.svg)

#### 2.2. Kế hoạch IP và VM

| VM | Hostname | IP | OS | vCPU | RAM | Disk | Role | Phần mềm thêm |
|----|----------|----|----|:----:|:---:|:----:|------|--------------|
| VM1 | DC1 | 10.10.200.11 | Windows Server 2022 | 8 | 8 GB | 200 GB | AD DS + DNS (PDC) | Exchange Management Tools |
| VM2 | DC2 | 10.10.200.12 | Windows Server 2022 | 8 | 8 GB | 200 GB | AD DS + DNS (ADC) | Microsoft Entra Connect |

**Thông tin domain và tenant:**

| Thông số | Giá trị |
|---------|---------|
| Internal AD Domain (Forest) | `anhlx.lab` |
| NetBIOS Name | `ANHLX` |
| Forest/Domain Functional Level | Windows Server 2016 |
| UPN Suffix thêm vào AD | `anhle.com.vn` |
| Default Gateway | `10.10.200.1` |
| Primary DNS (DC1) | `10.10.200.11` |
| Secondary DNS (DC2) | `10.10.200.12` |
| M365 Tenant | `AnhLe267.onmicrosoft.com` |
| M365 Admin Account | `AnhLe@AnhLe267.onmicrosoft.com` |
| M365 Plan | Microsoft 365 Business Basic |
| Domain verify trên M365 | `anhle.com.vn` |

**Network yêu cầu** — cần mở outbound cho 2 VM ra internet:

| Port | Protocol | Bắt buộc | Dịch vụ |
|------|----------|:--------:|---------|
| 443 | TCP | ✅ | Entra Connect sync, M365 authentication |
| 80 | TCP | ✅ | HTTP redirect (CRL check) |
| 53 | UDP/TCP | ✅ | DNS resolution ra ngoài |
| 123 | UDP | ✅ | NTP — đồng bộ giờ với DC (lệch >5 phút sẽ lỗi Kerberos) |

> **Lưu ý **: Tất cả 4 port trên đều là **outbound** từ VM ra internet, không cần mở inbound. Entra Connect không cần port nào từ cloud về on-prem.

---

### 3. Chuẩn bị M365 Tenant

#### 3.1. Đăng ký M365 Business Basic Trial

1. Truy cập [https://www.microsoft.com/microsoft-365/business/compare-all-plans](https://www.microsoft.com/microsoft-365/business/compare-all-plans)
2. Chọn **Microsoft 365 Business Basic** → **Try free for 1 month**
3. Đăng ký với email cá nhân, tạo tenant domain dạng `<TenantName>.onmicrosoft.com`
4. Ghi nhận thông tin:
   - **Tenant domain**: `AnhLe267.onmicrosoft.com`
   - **Global Admin**: `AnhLe@AnhLe267.onmicrosoft.com`
   - **Tenant ID**: Vào [https://entra.microsoft.com](https://entra.microsoft.com) → Identity → Overview

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/01.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/02.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/03.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/04.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/05.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/06.png)


#### 3.2. Verify domain anhle.com.vn

Để user AD có thể sync với UPN `user@anhle.com.vn`, phải verify domain `anhle.com.vn` với M365 tenant trước.

1. Đăng nhập [https://admin.cloud.microsoft](https://admin.cloud.microsoft) bằng Global Admin
2. Vào **Settings** → **Domains** → **Add domain**
3. Nhập `anhle.com.vn` → **Use this domain**
4. Chọn phương thức verify: **Add a TXT record to the domain's DNS records**
5. Ghi nhận TXT record được cung cấp (dạng: `MS=msXXXXXXXX`)
6. Đăng nhập vào DNS provider đang quản lý domain `anhle.com.vn`, thêm TXT record:
   - **Type**: TXT
   - **Name**: `@`
   - **Value**: `MS=msXXXXXXXX`
   - **TTL**: 3600
7. Quay lại M365 Admin → **Verify**
8. Ở bước **Add DNS records**: **bỏ tick** ô `Exchange and Exchange Online Protection` → nhấn **Continue**
9. Sau khi hoàn tất wizard, domain status hiển thị **"No services selected"** — đây là kết quả **đúng và mong đợi** cho lab này

> **"No services selected" không phải lỗi**: Domain đã được Microsoft xác nhận quyền sở hữu ✅. Status này chỉ có nghĩa là chưa gán MX/CNAME record cho Exchange Online — đúng với mục tiêu lab (chỉ sync Identity, không mail flow). Entra Connect chỉ cần domain được *add vào tenant*, không cần status "Healthy". Nếu thấy "Healthy" tức là đã thêm MX record.

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/07.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/08.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/09.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/10.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/11.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/12.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/13.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/14.png)

---

### 4. Cài đặt DC1 — Primary Domain Controller

#### 4.1. Cấu hình IP tĩnh và hostname

Mở **Server Manager** → **Local Server** → click **Ethernet** → **Properties** → **IPv4**:

- **IP Address**: `10.10.200.11`
- **Subnet Mask**: `255.255.255.0`
- **Default Gateway**: `10.10.200.1`
- **Preferred DNS**: `127.0.0.1`
- **Alternate DNS**: `10.10.200.12` _(điền sau khi DC2 sẵn sàng)_

Hoặc bằng PowerShell:

```powershell
# Cấu hình IP tĩnh
New-NetIPAddress -InterfaceAlias "Ethernet0" `
    -IPAddress "10.10.200.11" `
    -PrefixLength 24 `
    -DefaultGateway "10.10.200.1"

Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" `
    -ServerAddresses ("127.0.0.1","10.10.200.12")
```

Đổi hostname về `DC1`:

```powershell
Rename-Computer -NewName "DC1" -Restart
```

#### 4.2. Cài đặt AD DS và DNS

Sau khi reboot, mở PowerShell với quyền Administrator:

```powershell
# Cài AD DS, DNS, RSAT tools
Install-WindowsFeature -Name AD-Domain-Services, DNS `
    -IncludeManagementTools `
    -Verbose
```
![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/15.png)

#### 4.3. Promote lên Domain Controller (New Forest)

```powershell
# Import module ADDSDeployment
Import-Module ADDSDeployment

# Promote lên DC, tạo New Forest anhlx.lab
Install-ADDSForest `
    -DomainName "anhlx.lab" `
    -DomainNetbiosName "ANHLX" `
    -DomainMode "WinThreshold" `
    -ForestMode "WinThreshold" `
    -InstallDns:$true `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd2024!" -AsPlainText -Force) `
    -Force:$true
```

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/16.png)

Máy sẽ tự reboot sau khi promote thành công. Sau reboot, login bằng `ANHLX\Administrator`.

**Kiểm tra FSMO Roles:**

```powershell
netdom query fsmo
```

Output mong đợi — tất cả 5 FSMO roles đang nằm trên DC1:

```
Schema master               DC1.anhlx.lab
Domain naming master        DC1.anhlx.lab
PDC                         DC1.anhlx.lab
RID pool manager            DC1.anhlx.lab
Infrastructure master       DC1.anhlx.lab
```

**Cấu hình DNS Forwarder — bắt buộc để DC resolve ra internet:**

Sau khi promote, DNS Server trên DC1 mặc định không có forwarder → không resolve được domain ngoài internet → Entra Connect sync sẽ thất bại.

```powershell
# Thêm DNS Forwarder 8.8.8.8 (Google DNS)
Add-DnsServerForwarder -IPAddress "8.8.8.8" -PassThru

# Thêm thêm 1.1.1.1 làm backup (Cloudflare)
Add-DnsServerForwarder -IPAddress "1.1.1.1" -PassThru

# Kiểm tra forwarder đã được thêm
Get-DnsServerForwarder
```

Hoặc qua GUI: Mở **DNS Manager** → click chuột phải vào `DC1` → **Properties** → tab **Forwarders** → **Edit** → thêm `8.8.8.8` và `1.1.1.1` → **OK**.

**Kiểm tra DNS resolve ra ngoài:**

```powershell
# Test resolve domain ngoài internet
Resolve-DnsName -Name "login.microsoftonline.com" -Server 127.0.0.1
```

Nếu trả về IP address là forwarder hoạt động đúng.

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/17.png)

#### 4.4. Thêm UPN Suffix anhle.com.vn

Bước quan trọng: thêm `anhle.com.vn` làm UPN Suffix thứ hai trong AD. Các user khi được tạo với UPN `user@anhle.com.vn` sẽ match với domain đã verify trên M365.

**Cách 1 — GUI**: Mở **Active Directory Domains and Trusts** → Click chuột phải vào gốc cây **Active Directory Domains and Trusts** → **Properties** → Tab **UPN Suffixes** → **Add** → nhập `anhle.com.vn` → **OK**.

**Cách 2 — PowerShell:**

```powershell
Get-ADForest | Set-ADForest -UPNSuffixes @{Add="anhle.com.vn"}

# Kiểm tra kết quả
(Get-ADForest).UPNSuffixes
# Output: anhle.com.vn
```

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/18.png)

#### 4.5. Tạo OU Structure và Users

Thiết kế OU chuẩn cho doanh nghiệp:

```
anhlx.lab
└── ANHLX (OU root)
    ├── Users
    │   ├── IT
    │   ├── Sales
    │   └── HR
    ├── Computers
    │   ├── Workstations
    │   └── Servers
    └── Groups
        ├── DL (Distribution List)
        └── SG (Security Group)
```

```powershell
# Tạo OU structure
$base = "DC=anhlx,DC=lab"

New-ADOrganizationalUnit -Name "ANHLX" -Path $base
New-ADOrganizationalUnit -Name "Users" -Path "OU=ANHLX,$base"
New-ADOrganizationalUnit -Name "IT" -Path "OU=Users,OU=ANHLX,$base"
New-ADOrganizationalUnit -Name "Sales" -Path "OU=Users,OU=ANHLX,$base"
New-ADOrganizationalUnit -Name "HR" -Path "OU=Users,OU=ANHLX,$base"
New-ADOrganizationalUnit -Name "Computers" -Path "OU=ANHLX,$base"
New-ADOrganizationalUnit -Name "Workstations" -Path "OU=Computers,OU=ANHLX,$base"
New-ADOrganizationalUnit -Name "Groups" -Path "OU=ANHLX,$base"
```

**Tạo user với UPN suffix anhle.com.vn:**

```powershell
# Tạo user1 cho phòng IT
$password = ConvertTo-SecureString "User@12345" -AsPlainText -Force

New-ADUser `
    -Name "User One" `
    -GivenName "User" `
    -Surname "One" `
    -SamAccountName "user1" `
    -UserPrincipalName "user1@anhle.com.vn" `
    -EmailAddress "user1@anhle.com.vn" `
    -Path "OU=IT,OU=Users,OU=ANHLX,DC=anhlx,DC=lab" `
    -AccountPassword $password `
    -PasswordNeverExpires $false `
    -ChangePasswordAtLogon $false `
    -Enabled $true

# Tạo thêm user2
New-ADUser `
    -Name "User Two" `
    -GivenName "User" `
    -Surname "Two" `
    -SamAccountName "user2" `
    -UserPrincipalName "user2@anhle.com.vn" `
    -EmailAddress "user2@anhle.com.vn" `
    -Path "OU=Sales,OU=Users,OU=ANHLX,DC=anhlx,DC=lab" `
    -AccountPassword $password `
    -PasswordNeverExpires $false `
    -ChangePasswordAtLogon $false `
    -Enabled $true
```

#### 4.6. Cài Exchange Management Tools

**Exchange Management Tools** (không phải Exchange Server) cung cấp Exchange Management Console và PowerShell module để quản lý mail attributes của user trong AD. Điều này giúp IT admin điền đúng `EmailAddress`, `ProxyAddresses`, `TargetAddress` cho user trước khi sync lên M365.

**Yêu cầu trước khi cài:**

```powershell
# Cài .NET Framework 4.8 (nếu chưa có)
# Tải từ: https://dotnet.microsoft.com/download/dotnet-framework/net48

# Cài Visual C++ 2012 Redistributable x64 — BẮT BUỘC cho Exchange 2019
# Tải từ: https://www.microsoft.com/download/details.aspx?id=30679
Start-Process -FilePath "vcredist_x64_2012.exe" -ArgumentList "/install /quiet /norestart" -Wait

# Cài Visual C++ 2013 Redistributable x64 (khuyến nghị)
# Tải từ: https://aka.ms/highdpimfc2013x64enu
Start-Process -FilePath "vcredist_x64_2013.exe" -ArgumentList "/install /quiet /norestart" -Wait

# Reboot trước khi cài Exchange (bắt buộc nếu có pending reboot)
Restart-Computer -Force

# Kích hoạt IIS và các features cần thiết
Install-WindowsFeature Web-Server, Web-Mgmt-Console, `
    Web-Metabase, Web-Lgcy-Mgmt-Console, `
    Web-Basic-Auth, Web-Windows-Auth, Web-Digest-Auth, `
    Web-Net-Ext45, Web-Asp-Net45, Web-ISAPI-Ext, `
    Web-ISAPI-Filter, Web-Http-Redirect, Web-Dav-Publishing, `
    Web-Log-Libraries, Web-Http-Tracing, Web-Stat-Compression, `
    Web-Dyn-Compression, Web-WMI, Web-Scripting-Tools `
    -IncludeManagementTools
```

**Tải Exchange Server 2019 ISO (chỉ dùng bộ cài để extract Management Tools):**

> Tải ISO từ: [https://www.microsoft.com/en-us/download/details.aspx?id=104131](https://www.microsoft.com/en-us/download/details.aspx?id=104131)

Mount ISO và chạy Setup với option **ManagementTools Only**:

```powershell
# Mount ISO 
Mount-DiskImage -ImagePath "C:\Users\Administrator\Downloads\ExchangeServer2019-x64-CU12.iso"

# Chạy cài đặt chỉ Management Tools
.\Setup.exe /Mode:Install /Role:ManagementTools /OrganizationName:"ANHLX" /IAcceptExchangeServerLicenseTerms_DiagnosticDataOFF
```

Sau khi cài xong, mở **Exchange Management Shell** hoặc **Exchange Admin Center** (EAC) để quản lý mail attributes.

**Kiểm tra mail attribute của user1 sau khi cài xong:**

```powershell
# Xác nhận EmailAddress đã được set (đã set lúc New-ADUser)
Get-ADUser -Identity "user1" -Properties EmailAddress, UserPrincipalName | `
    Select-Object SamAccountName, UserPrincipalName, EmailAddress
```

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/19.png)

> **Lưu ý**: Trong lab này không có Exchange Server on-prem. Management Tools chỉ dùng để set thuộc tính `mail`, `proxyAddresses` cho user trong AD trước khi sync. Khi user được gán M365 license, Exchange Online tự tạo mailbox trên cloud — không cần `Enable-Mailbox` on-prem.

---

### 5. Cài đặt DC2 — Additional DC + Microsoft Entra Connect

#### 5.1. Cấu hình IP tĩnh, hostname và join domain

Cấu hình IP cho DC2:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0" `
    -IPAddress "10.10.200.12" `
    -PrefixLength 24 `
    -DefaultGateway "10.10.200.1"

# DNS trỏ về DC1 trước (để join domain)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" `
    -ServerAddresses ("10.10.200.11","127.0.0.1")
```

Đổi hostname và join domain `anhlx.lab`:

```powershell
Rename-Computer -NewName "DC2"

# Join domain (sẽ reboot)
Add-Computer -DomainName "anhlx.lab" `
    -Credential (Get-Credential) `
    -Restart
```

Sau reboot, login bằng `ANHLX\Administrator`.

#### 5.2. Promote lên Additional Domain Controller

```powershell
Install-WindowsFeature -Name AD-Domain-Services, DNS `
    -IncludeManagementTools

Import-Module ADDSDeployment

Install-ADDSDomainController `
    -DomainName "anhlx.lab" `
    -InstallDns:$true `
    -ReplicationSourceDC "DC1.anhlx.lab" `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd2024!" -AsPlainText -Force) `
    -Force:$true
```

Sau reboot, kiểm tra replication:

```powershell
# Kiểm tra AD Replication
repadmin /replsummary

# Kiểm tra danh sách DC trong domain
Get-ADDomainController -Filter * | Select-Object Name, IPv4Address, IsGlobalCatalog
```

Output mong đợi:

```
Name  IPv4Address    IsGlobalCatalog
----  -----------    ---------------
DC1   10.10.200.11   True
DC2   10.10.200.12   True
```

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/20.png)

**Sau khi DC2 sẵn sàng**, quay lại DC1 cập nhật Alternate DNS:

```powershell
# Trên DC1: cập nhật Alternate DNS thành DC2
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" `
    -ServerAddresses ("127.0.0.1","10.10.200.12")
```

#### 5.3. Cài đặt Microsoft Entra Connect

> Tải Entra Connect trực tiếp từ Entra portal:  
> [https://entra.microsoft.com](https://entra.microsoft.com) → sidebar **Entra Connect** → tab **Manage** → nhấn **Download Connect Sync Agent**  
> Tên file: `AzureADConnect.msi`

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/21.png)

1. Copy file `AzureADConnect.msi` vào DC2, chạy cài đặt
2. Đợi cài xong, màn hình **Welcome to Azure AD Connect** xuất hiện
3. Chọn **Customize** (không dùng Express để có thêm tùy chọn)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/22.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/23.png)

#### 5.4. Cấu hình Password Hash Sync + Seamless SSO

Trong wizard Entra Connect:

**Bước 1 — Install required components:**
- Bỏ trống tất cả các checkbox (dùng SQL LocalDB tự cài, service account tự tạo)
- Nhấn **Install** — wizard tự cài Microsoft OLE DB Driver for SQL và các thành phần cần thiết

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/24.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/25.png)

**Bước 2 — User sign-in:**
- Chọn **Password Hash Synchronization** ✅
- ☑ **Enable single sign-on** ✅
- Nhấn **Next**

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/26.png)

**Bước 3 — Connect to Microsoft Entra ID:**
- Nhập Global Admin: `AnhLe@anhle.com.vn` (custom domain đã verify)
- Nhấn **Next** → popup Microsoft Sign In xuất hiện
- Nếu account bật MFA: mở Microsoft Authenticator app → approve request với số hiển thị trên màn hình

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/27.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/28.png)

**Bước 4 — Connect your directories:**
- **Forest**: `anhlx.lab` (đã hiển thị sẵn)
- Nhấn **Add Directory**
- Chọn **Create new AD account**
- **ENTERPRISE ADMIN USERNAME**: `ANHLX\Administrator`
- **PASSWORD**: password của Administrator DC
- Nhấn **OK** → `anhlx.lab (Active Directory) ✅` xuất hiện trong **Configured Directories**
- Nhấn **Next**

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/29.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/30.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/31.png)

**Bước 5 — Microsoft Entra sign-in configuration:**
- Kiểm tra UPN Suffix `anhle.com.vn` có trạng thái **Verified** ✅
- `anhlx.lab` hiển thị **Not Added** — bình thường, đây là internal domain
- **USER PRINCIPAL NAME**: chọn `userPrincipalName`
- Nhấn **Next**

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/32.png)

**Bước 6 — Domain and OU filtering:**
- Chọn **Sync selected domains and OUs**
- Bỏ tick tất cả, chỉ expand `ANHLX` → tick **Users** (gồm HR, IT, Sales)
- Nhấn **Next**

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/33.png)

**Bước 7 — Uniquely identifying your users:**
- Để mặc định → **Next**

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/34.png)

**Bước 8 — Filter users and devices:**
- Chọn **Synchronize all users and devices** → **Next**

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/35.png)

**Bước 9 — Optional features:**
- ☑ **Password hash synchronization** ✅
- ☑ **Password writeback** ✅
- Còn lại bỏ trống (không có Exchange on-prem, không cần Group/Device writeback)
- Nhấn **Next**

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/36.png)

**Bước 10 — Enable single sign-on:**
- Nhấn **Enter credentials**
- Popup **Forest Credentials** xuất hiện:
  - **Username**: `administrator`
  - **Password**: password DC
  - **Domain**: `ANHLX` (tự hiển thị)
- Nhấn **OK** → `anhlx.lab ✅` xác nhận thành công
- Nhấn **Next**

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/37.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/38.png)

**Bước 11 — Ready to configure:**
- ☑ **Start the synchronization process when configuration completes**
- ☐ **Enable staging mode** — để trống (staging mode sẽ không sync thật)
- Nhấn **Install**

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/39.png)

Quá trình cài đặt và sync đầu tiên khoảng 5-10 phút. Sau khi hoàn tất, màn hình hiển thị **Configuration complete**.

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/40.png)

#### 5.5. Bật Password Writeback (SSPR)

Sau khi Entra Connect cài xong, bật SSPR trên M365 portal:

1. Đăng nhập [https://entra.microsoft.com](https://entra.microsoft.com)
2. Vào **Protection** → **Password reset**
3. **Properties**: Chọn **All** (cho phép tất cả user reset password)
4. **Authentication methods**: Bật **Mobile phone** và **Email** làm method xác thực
5. **On-premises integration**:
   - ☑ **Write back passwords to your on-premises directory** → **Yes**
   - ☑ **Allow users to unlock accounts without resetting their password** → **Yes**
6. Nhấn **Save**

---

### 6. Gán License và kiểm tra đồng bộ

#### 6.1. Kiểm tra User đã sync lên Entra ID

Trên DC2, mở PowerShell kiểm tra trạng thái sync:

```powershell
# Kết quả sync lần gần nhất
Import-Module ADSync
Get-ADSyncScheduler | Select-Object LastSyncCycleStartedDate, LastSyncCycleResult, SyncCycleEnabled

# Lưu ý: Get-ADSyncConnectorRunStatus chỉ hiển thị output khi đang có sync chạy.
# Khi idle (không có sync cycle đang chạy) sẽ trả về rỗng — đây là bình thường.
```

Trên M365 Admin Center ([https://admin.cloud.microsoft](https://admin.cloud.microsoft)):

1. Vào **Users** → **Active users**
2. Kiểm tra `user1@anhle.com.vn` và `user2@anhle.com.vn` đã xuất hiện
3. Cột **Sync status** phải hiển thị **Synced from on-premises**

Hoặc kiểm tra trên Entra ID portal ([https://entra.microsoft.com](https://entra.microsoft.com)):

1. Vào **Identity** → **Users** → **All users**
2. Tìm `user1` → kiểm tra **On-premises sync enabled: Yes**
3. **Source**: `Windows Server AD`

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/41.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/42.png)


#### 6.2. Gán M365 Business Basic License

Gán thủ công trên M365 Admin Center ([https://admin.cloud.microsoft](https://admin.cloud.microsoft)):

1. **Users** → **Active users** → chọn `user1@anhle.com.vn`
2. Tab **Licenses and apps**
3. Tick ☑ **Microsoft 365 Business Basic**
4. Nhấn **Save changes**

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/43.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/44.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/45.png)

#### 6.3. Đăng nhập Outlook Web bằng tài khoản AD sync

1. Mở trình duyệt **ẩn danh** (InPrivate / Incognito)
2. Truy cập [https://outlook.office.com](https://outlook.office.com)
3. Nhập email: `user1@anhle.com.vn`
4. Nhập password: mật khẩu đã tạo trong AD (ví dụ `User@12345`)
5. Đăng nhập thành công → Outlook Web mở ra ✅

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/46.png)

**Kết quả mong đợi:**
- Outlook Web load được, giao diện mailbox hiển thị — xác nhận Identity Sync và xác thực cloud hoạt động đúng
- Mailbox hiện **trống** hoặc chỉ có mail welcome từ Microsoft — đây là bình thường vì lab không cấu hình MX record, không có mail flow thực tế

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/47.png)

**Kiểm tra thêm:**
- Vào [https://teams.microsoft.com](https://teams.microsoft.com) — login được bằng cùng account ✅

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/48.png)

- Vào [https://portal.office.com](https://portal.office.com) — thấy các app M365 ✅

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/49.png)

> **Phạm vi lab đã hoàn thành**: AD on-prem → Entra Connect sync → Entra ID cloud → login M365 bằng credential AD. Nếu cần mail flow thực tế (gửi/nhận email qua `@anhle.com.vn`), bước tiếp theo sẽ là thêm MX record DNS — nhưng đó là chủ đề riêng và sẽ ảnh hưởng đến cấu hình DNS domain.

---

### 7. Force Sync và một số lệnh quản trị hay dùng

#### 7.1. Tạo user mới và kiểm tra sync tự động lên Entra ID

Sau khi Entra Connect đã cài và sync lần đầu thành công, mọi user tạo mới trong AD sẽ **tự động sync lên Entra ID trong vòng tối đa 30 phút** (chu kỳ mặc định). Hoặc có thể force sync ngay.

**Bước 1 — Tạo user3 trên DC1:**

```powershell
# Chạy trên DC1
$password = ConvertTo-SecureString "User@12345" -AsPlainText -Force

New-ADUser `
    -Name "User Three" `
    -GivenName "User" `
    -Surname "Three" `
    -SamAccountName "user3" `
    -UserPrincipalName "user3@anhle.com.vn" `
    -EmailAddress "user3@anhle.com.vn" `
    -Path "OU=HR,OU=Users,OU=ANHLX,DC=anhlx,DC=lab" `
    -AccountPassword $password `
    -PasswordNeverExpires $false `
    -ChangePasswordAtLogon $false `
    -Enabled $true
```

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/50.png)

**Bước 2 — Force Delta Sync ngay lập tức (trên DC2):**

```powershell
# Chạy trên DC2
Import-Module ADSync
Start-ADSyncSyncCycle -PolicyType Delta
```

Delta sync chỉ đẩy các thay đổi mới (user3 vừa tạo) — nhanh hơn full sync, thường hoàn tất trong 1-2 phút.

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/51.png)

**Bước 3 — Kiểm tra user3 đã xuất hiện trên Entra ID:**

Trên M365 Admin Center ([https://admin.cloud.microsoft](https://admin.cloud.microsoft)):
1. **Users** → **Active users**
2. `user3@anhle.com.vn` xuất hiện với **Sync status: Synced from on-premises** ✅

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/52.png)

![](/assets/img/2026-04-30-m365-hybrid-identity-2dc-lab/53.png)

**Bước 4 — Gán license và test login:**

1. Trên M365 Admin Center → chọn `user3@anhle.com.vn` → tab **Licenses and apps** → tick **Microsoft 365 Business Basic** → **Save**
2. Mở browser ẩn danh → vào [https://outlook.office.com](https://outlook.office.com) → login `user3@anhle.com.vn` / `User@12345` ✅

> **Kết luận**: Workflow thực tế cho IT admin — tạo user trong AD → chạy `Start-ADSyncSyncCycle -PolicyType Delta` → gán license trên M365 portal → user sẵn sàng dùng email trong ~5 phút.

---

```powershell
# === TRÊN DC2 (Entra Connect server) ===

# Force sync ngay lập tức (delta sync - chỉ sync thay đổi mới)
Start-ADSyncSyncCycle -PolicyType Delta

# Force full sync (sync toàn bộ)
Start-ADSyncSyncCycle -PolicyType Initial

# Xem lịch sử sync
Get-ADSyncScheduler

# Xem lỗi sync (nếu có)
$errors = Get-ADSyncCSObject -ConnectorName "anhlx.lab" | Where-Object {$_.ErrorCode -ne $null}
$errors | Select-Object DistinguishedName, ErrorCode, ErrorDescription

# Kiểm tra connectivity đến Entra ID
Test-ADSyncAzureServiceConnectivity -AzureEnvironment AzureCloud

# === TRÊN DC1 (quản lý AD) ===

# Xem tất cả user trong OU đã tạo
Get-ADUser -Filter * -SearchBase "OU=Users,OU=ANHLX,DC=anhlx,DC=lab" `
    -Properties UserPrincipalName, EmailAddress | `
    Select-Object SamAccountName, UserPrincipalName, EmailAddress

# Bulk tạo user từ CSV
# File users.csv: Name,SamAccountName,UPN,Email,OU
Import-Csv ".\users.csv" | ForEach-Object {
    New-ADUser `
        -Name $_.Name `
        -SamAccountName $_.SamAccountName `
        -UserPrincipalName $_.UPN `
        -EmailAddress $_.Email `
        -Path $_.OU `
        -AccountPassword (ConvertTo-SecureString "User@12345" -AsPlainText -Force) `
        -Enabled $true
}

# Đổi password user
Set-ADAccountPassword -Identity "user1" `
    -NewPassword (ConvertTo-SecureString "NewP@ss2025!" -AsPlainText -Force) `
    -Reset

# Kiểm tra AD Replication giữa 2 DC
repadmin /replsummary
repadmin /showrepl
```

---

### 8. Tổng kết

Sau khi hoàn thành lab này, bạn đã có một môi trường **Hybrid Identity** đầy đủ chức năng:

| Thành phần | Trạng thái | Mô tả |
|-----------|:---------:|-------|
| Active Directory Domain `anhlx.lab` | ✅ | 2 DC, High Availability, FSMO trên DC1 |
| AD Replication DC1 ↔ DC2 | ✅ | Tự động replication mọi thay đổi |
| UPN Suffix `anhle.com.vn` | ✅ | User AD có UPN match với M365 domain |
| Exchange Management Tools | ✅ | Quản lý mail attributes trực tiếp trong AD |
| Microsoft Entra Connect | ✅ | Sync AD → Entra ID mỗi 30 phút (hoặc force manual) |
| Password Hash Sync | ✅ | 1 mật khẩu cho AD và M365 |
| Password Writeback (SSPR) | ✅ | Reset pass trên cloud tự về AD |
| Seamless SSO | ✅ | Domain joined PC không cần nhập lại password |
| Exchange Online (M365 Business Basic) | ✅ | Mailbox cloud tạo tự động khi gán license, login Outlook Web thành công _(mail flow thực tế cần MX record — không cấu hình trong lab này)_ |

**Mô hình này bao phủ ~95% công việc thực tế của một System Administrator** tại doanh nghiệp SME Việt Nam hiện nay. Từ đây bạn có thể mở rộng thêm:

- **Group Policy**: Triển khai GPO để lock screen, cài phần mềm, map drive...
- **Intune**: Quản lý máy tính từ xa qua M365 Intune (đòi hỏi license cao hơn)
- **Microsoft Defender for Business**: Tích hợp bảo mật endpoint
- **Azure AD Joined PC**: Cho phép laptop không join on-prem AD vẫn dùng được M365 SSO
- **Conditional Access**: Chính sách bảo mật nâng cao (MFA, compliant device...)

