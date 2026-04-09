---
title: "Active Directory - Additional Domain Tree"
categories:
- System
- Windows Server
- Active Directory
feature_image: "../assets/postbanner.jpg"
feature_text: |
  ### Lab sáp nhập Forest anhlx.demo vào Forest anhlx.lab thành Additional Domain Tree
---

Bài viết hướng dẫn sáp nhập forest `anhlx.demo` (đã tồn tại) vào forest `anhlx.lab` dưới dạng **Additional Domain Tree**. Quy trình gồm: backup dữ liệu AD, demote các DC của `anhlx.demo`, promote lại thành Tree Domain trong forest `anhlx.lab`, rồi restore lại toàn bộ dữ liệu.

- [1. Giới thiệu](#1-giới-thiệu)
  - [Phân tích downtime](#phân-tích-downtime)
- [2. Thiết kế lab](#2-thiết-kế-lab)
  - [2.1. Topology](#21-topology)
  - [2.2. Kế hoạch IP](#22-kế-hoạch-ip)
- [3. Yêu cầu trước khi bắt đầu](#3-yêu-cầu-trước-khi-bắt-đầu)
- [4. Backup dữ liệu AD của forest anhlx.demo](#4-backup-dữ-liệu-ad-của-forest-anhlxdemo)
  - [4.1. Tạo thư mục backup](#41-tạo-thư-mục-backup)
  - [4.2. Export Organizational Units (OU)](#42-export-organizational-units-ou)
  - [4.3. Export Users](#43-export-users)
  - [4.4. Export Groups](#44-export-groups)
  - [4.5. Backup Group Policy Objects (GPO)](#45-backup-group-policy-objects-gpo)
  - [4.6. Kiểm tra backup](#46-kiểm-tra-backup)
- [5. Demote các DC của forest anhlx.demo](#5-demote-các-dc-của-forest-anhlxdemo)
  - [5.1. Unjoin win-02 khỏi domain ✔️ Không downtime](#51-unjoin-win-02-khỏi-domain-️-không-downtime)
  - [5.2. Demote DC-B (Additional DC) ✔️ Không downtime](#52-demote-dc-b-additional-dc-️-không-downtime)
  - [5.3. Demote DC-A (Last DC in domain) ⚠️ Bắt đầu downtime](#53-demote-dc-a-last-dc-in-domain-️-bắt-đầu-downtime)
- [6. Promote DC-A thành Tree Domain Controller ⚠️ Downtime](#6-promote-dc-a-thành-tree-domain-controller-️-downtime)
  - [6.1. Cấu hình DNS trỏ về Forest Root](#61-cấu-hình-dns-trỏ-về-forest-root)
  - [6.2. Cài đặt AD DS và DNS](#62-cài-đặt-ad-ds-và-dns)
  - [6.3. Promote lên Domain Controller - New Tree](#63-promote-lên-domain-controller---new-tree)
- [7. Restore dữ liệu AD ⚡ Kết thúc downtime](#7-restore-dữ-liệu-ad--kết-thúc-downtime)
  - [7.1. Restore Organizational Units (OU)](#71-restore-organizational-units-ou)
  - [7.2. Restore Users](#72-restore-users)
  - [7.3. Restore Groups và Group Membership](#73-restore-groups-và-group-membership)
  - [7.4. Restore Group Policy Objects (GPO)](#74-restore-group-policy-objects-gpo)
  - [7.5. Kiểm tra sau restore](#75-kiểm-tra-sau-restore)
- [8. Promote DC-B thành Additional DC](#8-promote-dc-b-thành-additional-dc)
  - [8.1. Cấu hình DNS trỏ về DC-A](#81-cấu-hình-dns-trỏ-về-dc-a)
  - [8.2. Cài đặt AD DS và DNS](#82-cài-đặt-ad-ds-và-dns)
  - [8.3. Promote lên Additional Domain Controller](#83-promote-lên-additional-domain-controller)
- [9. Join domain lại cho win-02](#9-join-domain-lại-cho-win-02)
- [10. Kiểm tra Domain Tree và Trust](#10-kiểm-tra-domain-tree-và-trust)
  - [10.1. Kiểm tra Forest và Domain](#101-kiểm-tra-forest-và-domain)
  - [10.2. Kiểm tra Trust Relationship](#102-kiểm-tra-trust-relationship)
  - [10.3. Kiểm tra Replication](#103-kiểm-tra-replication)
  - [10.4. Kiểm tra DNS](#104-kiểm-tra-dns)
- [11. Login cross-domain](#11-login-cross-domain)
  - [11.1. Từ win-2022 (anhlx.lab) login user anhlx.demo](#111-từ-win-2022-anhlxlab-login-user-anhlxdemo)
  - [11.2. Từ win-02 (anhlx.demo) login user anhlx.lab](#112-từ-win-02-anhlxdemo-login-user-anhlxlab)

---

### 1. Giới thiệu

Trong Active Directory, một **Forest** có thể chứa nhiều **Domain Tree**. Tuy nhiên, **không thể merge 2 forest có sẵn** thành 1 forest. Để đưa domain `anhlx.demo` vào forest `anhlx.lab`, ta phải:

1. **Backup** toàn bộ dữ liệu AD (OU, User, Group, GPO) của forest `anhlx.demo`
2. **Demote** tất cả DC của forest `anhlx.demo` (xóa forest cũ)
3. **Promote lại** DC-A thành **New Tree Domain** `anhlx.demo` trong forest `anhlx.lab`
4. **Promote lại** DC-B thành Additional DC
5. **Join lại** client win-02 vào domain `anhlx.demo` mới
6. **Restore** dữ liệu AD từ bản backup

> **Lưu ý:** Quá trình demote sẽ **xóa toàn bộ dữ liệu AD** (user, group, GPO, OU) của forest `anhlx.demo` cũ. Tuy nhiên, ta sẽ **export trước và import lại** sau khi promote, đảm bảo không mất dữ liệu.

#### Phân tích downtime

**Không thể zero-downtime** khi merge 2 forest — domain name `anhlx.demo` không thể tồn tại đồng thời ở 2 forest khác nhau. Tuy nhiên, bài viết sắp xếp thứ tự tối ưu để **giảm downtime xuống tối thiểu**:

| Phase | Bước | Downtime anhlx.demo? |
|-------|------|---------------------|
| Chuẩn bị | 4. Backup dữ liệu AD | **Không** — forest vẫn chạy bình thường |
| Chuẩn bị | 5.1. Unjoin win-02 | **Không** — chỉ ảnh hưởng client này |
| Chuẩn bị | 5.2. Demote DC-B | **Không** — anhlx.demo vẫn chạy trên DC-A |
| ⚠️ Downtime | 5.3. Demote DC-A (last DC) | **BẮT ĐẦU DOWNTIME** — forest anhlx.demo bị xóa |
| ⚠️ Downtime | 6. Promote DC-A → Tree Domain | Đang rebuild... |
| ⚠️ Downtime | 7. Restore dữ liệu AD | **KẾT THÚC DOWNTIME** — domain hoạt động trở lại |
| Sau phục hồi | 8. Promote DC-B (redundancy) | **Không** — chạy nền |
| Sau phục hồi | 9. Join win-02 | **Không** — domain đã sẵn sàng |

**Cửa sổ downtime thực tế:** Chỉ từ bước 5.3 → 7 (demote DC-A cuối cùng → promote lại → restore data), ước tính **30-60 phút** tùy lượng data cần restore. Tất cả bước chuẩn bị (4, 5.1, 5.2) và bước sau (8, 9, 10, 11) đều **không gây downtime**.

Sau khi hoàn thành, cấu trúc sẽ là:

**Trước:**

```
Forest 1: anhlx.lab              Forest 2: anhlx.demo
├── DC1  (10.10.200.31)          ├── DC-A  (10.10.200.41)
├── DC2  (10.10.200.32)          ├── DC-B  (10.10.200.42)
└── win-2022 (10.10.200.33)      └── win-02 (10.10.200.43)
```

**Sau:**

```
Forest: anhlx.lab
├── Tree 1: anhlx.lab (Forest Root Domain)
│   ├── DC1 (10.10.200.31) - Tree Root DC
│   ├── DC2 (10.10.200.32) - Additional DC
│   └── win-2022 (10.10.200.33) - Client
│
└── Tree 2: anhlx.demo (Additional Domain Tree)
    ├── DC-A (10.10.200.41) - Tree Root DC
    ├── DC-B (10.10.200.42) - Additional DC
    └── win-02 (10.10.200.43) - Client
```

Giữa `anhlx.lab` và `anhlx.demo` sẽ tự động có **tree-root trust** (two-way transitive), cho phép user ở cả 2 domain xác thực qua lại.

### 2. Thiết kế lab

#### 2.1. Topology

![Topology](../assets/img/2026-04-09-active-directory-additional-domain-tree/01.png)

#### 2.2. Kế hoạch IP

| Vai trò | Hostname | IP | Subnet Mask | Gateway | DNS |
|---------|----------|----|-------------|---------|-----|
| DC1 (anhlx.lab) | DC1 | 10.10.200.31 | 255.255.255.0 | 10.10.200.1 | 127.0.0.1, 10.10.200.32 |
| DC2 (anhlx.lab) | DC2 | 10.10.200.32 | 255.255.255.0 | 10.10.200.1 | 10.10.200.31, 127.0.0.1 |
| Client (anhlx.lab) | win-2022 | 10.10.200.33 | 255.255.255.0 | 10.10.200.1 | 10.10.200.31, 10.10.200.32 |
| Tree DC-A (anhlx.demo) | DC-A | 10.10.200.41 | 255.255.255.0 | 10.10.200.1 | 10.10.200.31, 127.0.0.1 |
| Tree DC-B (anhlx.demo) | DC-B | 10.10.200.42 | 255.255.255.0 | 10.10.200.1 | 10.10.200.41, 127.0.0.1 |
| Client (anhlx.demo) | win-02 | 10.10.200.43 | 255.255.255.0 | 10.10.200.1 | 10.10.200.41, 10.10.200.42 |

**Thông tin domain:**

| Thuộc tính | Forest Root | Domain Tree mới |
|------------|-------------|-----------------|
| Domain name | `anhlx.lab` | `anhlx.demo` |
| NetBIOS name | `ANHLX` | `ANHLXDEMO` |
| Forest name | `anhlx.lab` | `anhlx.lab` (cùng forest) |
| Functional Level | Windows Server 2016 | Windows Server 2016 |

### 3. Yêu cầu trước khi bắt đầu

Trước khi thực hiện, cần đảm bảo:

1. **Forest `anhlx.lab` đã hoạt động** — DC1 (10.10.200.31), DC2 (10.10.200.32) đang chạy, win-2022 đã join domain
2. **Forest `anhlx.demo` đang hoạt động** — DC-A (10.10.200.41), DC-B (10.10.200.42) đang chạy, win-02 đã join domain
3. **Network thông suốt** — Tất cả các máy ping được lẫn nhau
4. **Backup dữ liệu AD của anhlx.demo** — Export OU, User, Group, GPO ra file để restore sau khi promote

Kiểm tra trạng thái hiện tại:

```powershell
# Trên DC-A - kiểm tra forest anhlx.demo
Get-ADForest
Get-ADDomainController -Filter *

# Trên DC1 - kiểm tra forest anhlx.lab
Get-ADForest
Get-ADDomainController -Filter *
```

### 4. Backup dữ liệu AD của forest anhlx.demo

Trước khi demote, cần **export toàn bộ dữ liệu AD** của `anhlx.demo` ra file. Thực hiện trên **DC-A** (10.10.200.41):

#### 4.1. Tạo thư mục backup

```powershell
$BackupPath = "C:\AD-Backup-anhlx.demo"
New-Item -ItemType Directory -Path $BackupPath -Force
New-Item -ItemType Directory -Path "$BackupPath\GPOs" -Force
```

#### 4.2. Export Organizational Units (OU)

```powershell
Get-ADOrganizationalUnit -Filter * -Properties * |
    Select-Object Name, DistinguishedName, Description, ProtectedFromAccidentalDeletion |
    Export-Csv "$BackupPath\OUs.csv" -NoTypeInformation -Encoding UTF8
```

#### 4.3. Export Users

```powershell
# Export tất cả user (trừ built-in accounts)
Get-ADUser -Filter {SamAccountName -ne 'Administrator' -and SamAccountName -ne 'Guest' -and SamAccountName -ne 'krbtgt'} -Properties * |
    Select-Object SamAccountName, Name, GivenName, Surname, DisplayName,
        UserPrincipalName, EmailAddress, Description, Department, Title, Company,
        Enabled, PasswordNeverExpires, CannotChangePassword,
        @{N='MemberOf';E={($_.MemberOf | ForEach-Object { ($_ -split ',')[0] -replace 'CN=' }) -join ';'}},
        @{N='ParentOU';E={($_.DistinguishedName -split ',',2)[1]}} |
    Export-Csv "$BackupPath\Users.csv" -NoTypeInformation -Encoding UTF8
```

#### 4.4. Export Groups

```powershell
# Export custom groups (trừ built-in)
Get-ADGroup -Filter {GroupCategory -eq 'Security' -or GroupCategory -eq 'Distribution'} -Properties * |
    Where-Object { $_.DistinguishedName -notmatch 'CN=Builtin,' } |
    Select-Object Name, SamAccountName, GroupScope, GroupCategory, Description,
        @{N='ParentOU';E={($_.DistinguishedName -split ',',2)[1]}} |
    Export-Csv "$BackupPath\Groups.csv" -NoTypeInformation -Encoding UTF8

# Export group membership
Get-ADGroup -Filter {GroupCategory -eq 'Security' -or GroupCategory -eq 'Distribution'} |
    Where-Object { $_.DistinguishedName -notmatch 'CN=Builtin,' } |
    ForEach-Object {
        $groupName = $_.Name
        Get-ADGroupMember -Identity $_ -ErrorAction SilentlyContinue |
            Select-Object @{N='GroupName';E={$groupName}}, Name, SamAccountName, ObjectClass
    } | Export-Csv "$BackupPath\GroupMembers.csv" -NoTypeInformation -Encoding UTF8
```

#### 4.5. Backup Group Policy Objects (GPO)

```powershell
# Backup tất cả GPO
Backup-GPO -All -Path "$BackupPath\GPOs"

# Export danh sách GPO và link
Get-GPO -All | Select-Object DisplayName, Id, GpoStatus, CreationTime |
    Export-Csv "$BackupPath\GPOs-List.csv" -NoTypeInformation -Encoding UTF8

# Export GPO links (OU nào link GPO nào)
Get-ADOrganizationalUnit -Filter * | ForEach-Object {
    $ou = $_
    (Get-GPInheritance -Target $_.DistinguishedName).GpoLinks | ForEach-Object {
        [PSCustomObject]@{
            OUName = $ou.Name
            OUDN   = $ou.DistinguishedName
            GPOName = $_.DisplayName
            GPOId   = $_.GpoId
            Enabled = $_.Enabled
            Order   = $_.Order
        }
    }
} | Export-Csv "$BackupPath\GPO-Links.csv" -NoTypeInformation -Encoding UTF8
```

#### 4.6. Kiểm tra backup

```powershell
Get-ChildItem $BackupPath -Recurse | Select-Object FullName, Length
```

Kết quả mong đợi:

```
C:\AD-Backup-anhlx.demo\OUs.csv
C:\AD-Backup-anhlx.demo\Users.csv
C:\AD-Backup-anhlx.demo\Groups.csv
C:\AD-Backup-anhlx.demo\GroupMembers.csv
C:\AD-Backup-anhlx.demo\GPOs-List.csv
C:\AD-Backup-anhlx.demo\GPO-Links.csv
C:\AD-Backup-anhlx.demo\GPOs\{...}\  (thư mục backup GPO)
```

> **Quan trọng:** Copy thư mục `C:\AD-Backup-anhlx.demo` sang nơi an toàn (USB, network share) phòng trường hợp mất dữ liệu trên DC-A. Đảm bảo file backup tồn tại trước khi tiếp tục demote.

### 5. Demote các DC của forest anhlx.demo

#### 5.1. Unjoin win-02 khỏi domain ✔️ Không downtime

Trước khi demote DC, phải unjoin client ra khỏi domain trước.

Trên `win-02` (10.10.200.43), đăng nhập bằng local admin:

```powershell
# Unjoin khỏi domain và chuyển về Workgroup
Remove-Computer -UnjoinDomainCredential (Get-Credential DEMO\Administrator) -WorkgroupName "WORKGROUP" -Restart
```

Hoặc bằng GUI: **System Properties** → **Computer Name** → **Change** → chọn **Workgroup** → nhập `WORKGROUP` → Restart.

![Unjoin win-02](../assets/img/2026-04-09-active-directory-additional-domain-tree/win02-unjoin.png)

#### 5.2. Demote DC-B (Additional DC) ✔️ Không downtime

Demote DC-B trước vì nó là Additional DC (không phải last DC). Domain `anhlx.demo` vẫn hoạt động bình thường trên DC-A.

Đăng nhập `DEMO\Administrator` trên DC-B (10.10.200.42):

**Cách 1: GUI**

Mở **Server Manager** → **Manage** → **Remove Roles and Features** → bỏ tick **Active Directory Domain Services** → sẽ hiện wizard **Demote** → làm theo wizard.

**Cách 2: PowerShell**

```powershell
# Demote DC-B
Uninstall-ADDSDomainController `
    -Credential (Get-Credential DEMO\Administrator) `
    -LocalAdministratorPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) `
    -Force:$true
```

![Demote DC-B](../assets/img/2026-04-09-active-directory-additional-domain-tree/dc-b-demote.png)

Sau khi restart, DC-B trở thành member server. Gỡ role AD DS:

```powershell
Uninstall-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
```

#### 5.3. Demote DC-A (Last DC in domain) ⚠️ Bắt đầu downtime

DC-A là **DC cuối cùng** trong forest `anhlx.demo`, nên cần thêm flag `-LastDomainControllerInDomain` và `-ForceRemoval`.

> **⚠️ DOWNTIME BẮT ĐẦU TỪ ĐÂY** — Sau bước này, forest `anhlx.demo` sẽ bị xóa hoàn toàn. User không thể đăng nhập cho đến khi promote + restore xong (ước tính 30-60 phút).

Đăng nhập `DEMO\Administrator` trên DC-A (10.10.200.41):

```powershell
# Demote DC-A - Last DC in forest
Uninstall-ADDSDomainController `
    -LastDomainControllerInDomain `
    -RemoveApplicationPartitions `
    -LocalAdministratorPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) `
    -Force:$true
```

![Demote DC-A](../assets/img/2026-04-09-active-directory-additional-domain-tree/dc-a-demote.png)

Sau khi restart, gỡ role:

```powershell
Uninstall-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
```

> **Lúc này forest `anhlx.demo` đã bị xóa hoàn toàn.** DC-A và DC-B đều là standalone server.

### 6. Promote DC-A thành Tree Domain Controller ⚠️ Downtime

#### 6.1. Cấu hình DNS trỏ về Forest Root

DC-A cần tìm thấy forest `anhlx.lab` để tạo thêm domain tree:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 10.10.200.31, 127.0.0.1
```

Kiểm tra:

```powershell
nslookup anhlx.lab 10.10.200.31
ping DC1.anhlx.lab
```

> **Quan trọng:** DNS phải trỏ về DC1 (10.10.200.31) của `anhlx.lab` trước, vì DC-A cần tìm thấy forest `anhlx.lab` để tạo thêm domain tree.

#### 6.2. Cài đặt AD DS và DNS

```powershell
Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
```

![Cài đặt AD DS Role](../assets/img/2026-04-09-active-directory-additional-domain-tree/dc-a-add-role.png)

#### 6.3. Promote lên Domain Controller - New Tree

Đây là bước quan trọng nhất — tạo **New Domain Tree** `anhlx.demo` trong forest `anhlx.lab`.

**Cách 1: GUI**

Trong Server Manager, click **Promote this server to a domain controller**:

1. Chọn **Add a new domain to an existing forest**
2. Select domain type: **Tree Domain**
3. Forest name: `anhlx.lab`
4. New domain name: `anhlx.demo`
5. Supply credentials: `ANHLX\Administrator` (Enterprise Admin của forest root)
6. Forest/Domain functional level: **Windows Server 2016**
7. Tick **Domain Name System (DNS) server** và **Global Catalog (GC)**
8. Đặt **DSRM password**
9. Tạo **DNS delegation** (nếu được hỏi, có thể bỏ qua)
10. NetBIOS domain name: `DEMO`
11. Giữ mặc định đường dẫn Database, Log, SYSVOL
12. Next → Install → Server sẽ tự restart

![Promote DC-A - New Tree Domain](../assets/img/2026-04-09-active-directory-additional-domain-tree/dc-a-promote-tree.png)

**Cách 2: PowerShell**

```powershell
Install-ADDSDomain `
    -NewDomainName "anhlx.demo" `
    -ParentDomainName "anhlx.lab" `
    -DomainType "TreeDomain" `
    -NewDomainNetbiosName "DEMO" `
    -DomainMode "WinThreshold" `
    -InstallDNS:$true `
    -CreateDnsDelegation:$true `
    -Credential (Get-Credential ANHLX\Administrator) `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) `
    -Force:$true
```

> **Lưu ý:** Credential phải là **Enterprise Admin** của forest root `anhlx.lab`, không phải local admin.

Sau khi restart, đăng nhập bằng `DEMO\Administrator`.

![DC-A đã promote thành công](../assets/img/2026-04-09-active-directory-additional-domain-tree/dc-a-promote-done.png)

**Sau khi promote xong, cập nhật DNS của DC-A:**

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 127.0.0.1, 10.10.200.31
```

### 7. Restore dữ liệu AD ⚡ Kết thúc downtime

Ngay sau khi promote DC-A xong, tiến hành **restore dữ liệu AD** ngay lập tức để **kết thúc downtime**. Thực hiện trên **DC-A** (10.10.200.41):

> **Lưu ý:** Đảm bảo thư mục `C:\AD-Backup-anhlx.demo` vẫn tồn tại trên DC-A (hoặc copy lại từ nơi backup).

#### 7.1. Restore Organizational Units (OU)

Tạo lại OU theo đúng thứ tự (parent trước, child sau):

```powershell
$BackupPath = "C:\AD-Backup-anhlx.demo"

Import-Csv "$BackupPath\OUs.csv" |
    Sort-Object { ($_.DistinguishedName -split ',').Count } |
    ForEach-Object {
        $parentDN = ($_.DistinguishedName -split ',', 2)[1]
        try {
            New-ADOrganizationalUnit -Name $_.Name -Path $parentDN `
                -Description $_.Description `
                -ProtectedFromAccidentalDeletion ([bool]::Parse($_.ProtectedFromAccidentalDeletion))
            Write-Host "[OK] OU: $($_.Name)" -ForegroundColor Green
        } catch {
            Write-Host "[SKIP] OU: $($_.Name) - $($_.Exception.Message)" -ForegroundColor Yellow
        }
    }
```

#### 7.2. Restore Users

```powershell
Import-Csv "$BackupPath\Users.csv" | ForEach-Object {
    try {
        $params = @{
            Name              = $_.Name
            SamAccountName    = $_.SamAccountName
            UserPrincipalName = $_.UserPrincipalName
            GivenName         = $_.GivenName
            Surname           = $_.Surname
            DisplayName       = $_.DisplayName
            Description       = $_.Description
            Department        = $_.Department
            Title             = $_.Title
            Company           = $_.Company
            EmailAddress      = $_.EmailAddress
            Path              = $_.ParentOU
            Enabled           = ([bool]::Parse($_.Enabled))
            AccountPassword   = (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force)
        }
        # Loại bỏ property rỗng
        $params = $params.GetEnumerator() |
            Where-Object { $_.Value -ne '' -and $null -ne $_.Value } |
            ForEach-Object -Begin { $h = @{} } -Process { $h[$_.Key] = $_.Value } -End { $h }

        New-ADUser @params
        Write-Host "[OK] User: $($_.SamAccountName)" -ForegroundColor Green
    } catch {
        Write-Host "[SKIP] User: $($_.SamAccountName) - $($_.Exception.Message)" -ForegroundColor Yellow
    }
}
```

> **Lưu ý:** Tất cả user sẽ được tạo với password tạm `P@ssw0rd123`. Yêu cầu user đổi password khi login lần đầu:

```powershell
Get-ADUser -Filter {SamAccountName -ne 'Administrator'} |
    Set-ADUser -ChangePasswordAtLogon $true
```

#### 7.3. Restore Groups và Group Membership

```powershell
# Tạo lại groups
Import-Csv "$BackupPath\Groups.csv" | ForEach-Object {
    try {
        New-ADGroup -Name $_.Name -SamAccountName $_.SamAccountName `
            -GroupScope $_.GroupScope -GroupCategory $_.GroupCategory `
            -Description $_.Description -Path $_.ParentOU
        Write-Host "[OK] Group: $($_.Name)" -ForegroundColor Green
    } catch {
        Write-Host "[SKIP] Group: $($_.Name) - $($_.Exception.Message)" -ForegroundColor Yellow
    }
}

# Restore group membership
Import-Csv "$BackupPath\GroupMembers.csv" | ForEach-Object {
    try {
        Add-ADGroupMember -Identity $_.GroupName -Members $_.SamAccountName
        Write-Host "[OK] $($_.SamAccountName) -> $($_.GroupName)" -ForegroundColor Green
    } catch {
        Write-Host "[SKIP] $($_.SamAccountName) -> $($_.GroupName) - $($_.Exception.Message)" -ForegroundColor Yellow
    }
}
```

#### 7.4. Restore Group Policy Objects (GPO)

```powershell
# Import GPOs
$gpoList = Import-Csv "$BackupPath\GPOs-List.csv"
$gpoBackups = Get-ChildItem "$BackupPath\GPOs" -Directory

foreach ($backup in $gpoBackups) {
    $gpoInfo = $gpoList | Where-Object { $_.Id -eq $backup.Name -or "{$($_.Id)}" -eq $backup.Name }
    if ($gpoInfo) {
        try {
            Import-GPO -BackupId $backup.Name -Path "$BackupPath\GPOs" -TargetName $gpoInfo.DisplayName -CreateIfNeeded
            Write-Host "[OK] GPO: $($gpoInfo.DisplayName)" -ForegroundColor Green
        } catch {
            Write-Host "[SKIP] GPO: $($gpoInfo.DisplayName) - $($_.Exception.Message)" -ForegroundColor Yellow
        }
    }
}

# Re-link GPO vào OU
Import-Csv "$BackupPath\GPO-Links.csv" | ForEach-Object {
    try {
        New-GPLink -Name $_.GPOName -Target $_.OUDN -LinkEnabled $_.Enabled
        Write-Host "[OK] Link: $($_.GPOName) -> $($_.OUName)" -ForegroundColor Green
    } catch {
        Write-Host "[SKIP] Link: $($_.GPOName) -> $($_.OUName) - $($_.Exception.Message)" -ForegroundColor Yellow
    }
}
```

#### 7.5. Kiểm tra sau restore

```powershell
# Đếm số object đã restore
Write-Host "=== Restore Summary ===" -ForegroundColor Cyan
Write-Host "OUs   : $((@(Get-ADOrganizationalUnit -Filter *)).Count)"
Write-Host "Users : $((@(Get-ADUser -Filter *)).Count)"
Write-Host "Groups: $((@(Get-ADGroup -Filter * | Where-Object {$_.DistinguishedName -notmatch 'CN=Builtin,'})).Count)"
Write-Host "GPOs  : $((@(Get-GPO -All)).Count)"
```

> **Tip:** So sánh số lượng với file backup CSV để đảm bảo không thiếu object nào.
>
> ✅ **Sau bước này, domain `anhlx.demo` đã hoạt động trở lại** với đầy đủ OU, User, Group, GPO. User có thể đăng nhập được. Downtime kết thúc.

### 8. Promote DC-B thành Additional DC

> Bước này chạy **sau khi domain đã phục hồi**, không gây downtime. Mục đích: thêm DC dự phòng cho `anhlx.demo`.

#### 8.1. Cấu hình DNS trỏ về DC-A

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 10.10.200.41, 127.0.0.1
```

#### 8.2. Cài đặt AD DS và DNS

```powershell
Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
```

#### 8.3. Promote lên Additional Domain Controller

Click **Promote this server to a domain controller**:

1. Chọn **Add a domain controller to an existing domain**
2. Domain: `anhlx.demo`
3. Credentials: `DEMO\Administrator`
4. Tick **DNS server** và **Global Catalog**
5. Đặt **DSRM password**
6. Chọn **Replicate from**: `DC-A.anhlx.demo`
7. Giữ mặc định đường dẫn → Install → Restart

![Promote DC-B](../assets/img/2026-04-09-active-directory-additional-domain-tree/dc-b-promote.png)

```powershell
Install-ADDSDomainController `
    -DomainName "anhlx.demo" `
    -Credential (Get-Credential DEMO\Administrator) `
    -InstallDNS:$true `
    -ReplicationSourceDC "DC-A.anhlx.demo" `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) `
    -Force:$true
```

Sau khi restart, đăng nhập bằng `DEMO\Administrator`.

### 9. Join domain lại cho win-02

> Domain `anhlx.demo` đã sẵn sàng, bước này không gây downtime.

Cấu hình DNS trỏ về DC-A và DC-B mới:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 10.10.200.41, 10.10.200.42
```

Join domain `anhlx.demo` (đã là tree domain trong forest `anhlx.lab`):

**Cách 1: GUI**

1. Mở **System Properties** → **Computer Name** → **Change**
2. Chọn **Domain**, nhập `anhlx.demo`
3. Nhập credentials `DEMO\Administrator`
4. Restart máy

![Join Domain win-02](../assets/img/2026-04-09-active-directory-additional-domain-tree/win02-join-domain.png)

**Cách 2: PowerShell**

```powershell
Add-Computer -DomainName "anhlx.demo" -Credential (Get-Credential DEMO\Administrator) -Restart
```

Sau khi restart, đăng nhập bằng `DEMO\Administrator` hoặc `administrator@anhlx.demo`.

### 10. Kiểm tra Domain Tree và Trust

#### 10.1. Kiểm tra Forest và Domain

Trên DC-A (anhlx.demo), kiểm tra cấu trúc forest:

```powershell
# Thông tin forest
Get-ADForest

# Liệt kê tất cả domain trong forest
(Get-ADForest).Domains

# Thông tin domain hiện tại
Get-ADDomain
```

Kết quả `(Get-ADForest).Domains` phải trả về cả 2 domain:

```
anhlx.lab
anhlx.demo
```

![Forest Info](../assets/img/2026-04-09-active-directory-additional-domain-tree/forest-info.png)

#### 10.2. Kiểm tra Trust Relationship

Khi tạo Domain Tree mới, hệ thống tự động tạo **tree-root trust** (two-way transitive) giữa `anhlx.demo` và `anhlx.lab`.

```powershell
# Xem trust relationship
Get-ADTrust -Filter *

# Kiểm tra chi tiết trust
Get-ADTrust -Identity "anhlx.lab"
```

![Trust Relationship](../assets/img/2026-04-09-active-directory-additional-domain-tree/trust-relationship.png)

Hoặc kiểm tra bằng GUI: mở **Active Directory Domains and Trusts** (domain.msc) → right-click `anhlx.demo` → **Properties** → tab **Trusts**.

Validate trust:

```powershell
# Validate trust từ DC-A (anhlx.demo)
netdom trust anhlx.demo /domain:anhlx.lab /verify
```

#### 10.3. Kiểm tra Replication

```powershell
# Replication giữa DC-A và DC-B trong anhlx.demo
repadmin /replsummary

# Chi tiết replication partners
repadmin /showrepl
```

![Replication Status](../assets/img/2026-04-09-active-directory-additional-domain-tree/replication-status.png)

#### 10.4. Kiểm tra DNS

Conditional Forwarder và DNS delegation cho phép 2 domain phân giải lẫn nhau:

```powershell
# Từ DC-A (anhlx.demo) - phân giải forest root
nslookup anhlx.lab 10.10.200.41

# Từ DC1 (anhlx.lab) - phân giải domain tree mới
nslookup anhlx.demo 10.10.200.31

# Kiểm tra SRV records
nslookup -type=srv _ldap._tcp.anhlx.demo
nslookup -type=srv _ldap._tcp.anhlx.lab
```

Nếu DNS không phân giải được cross-domain, cần thêm **Conditional Forwarder** thủ công:

```powershell
# Trên DC-A (anhlx.demo) - forward queries cho anhlx.lab sang DC1
Add-DnsServerConditionalForwarderZone -Name "anhlx.lab" -MasterServers 10.10.200.31

# Trên DC1 (anhlx.lab) - forward queries cho anhlx.demo sang DC-A
Add-DnsServerConditionalForwarderZone -Name "anhlx.demo" -MasterServers 10.10.200.41
```

### 11. Login cross-domain

Nhờ **tree-root trust** (two-way transitive) giữa `anhlx.lab` và `anhlx.demo`, client của mỗi domain đều có thể đăng nhập bằng user của domain kia.

#### 11.1. Từ win-2022 (anhlx.lab) login user anhlx.demo

Cấu hình DNS trên `win-2022` để phân giải được cả `anhlx.demo`:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 10.10.200.31, 10.10.200.41
```

Kiểm tra phân giải:

```powershell
nslookup anhlx.lab
nslookup anhlx.demo
```

Trên màn hình đăng nhập của `win-2022`, có thể login bằng:

| Domain | Cách đăng nhập | Ví dụ |
|--------|---------------|-------|
| anhlx.lab (domain gốc) | `ANHLX\username` hoặc `user@anhlx.lab` | `ANHLX\Administrator` |
| anhlx.demo (cross-domain) | `DEMO\username` hoặc `user@anhlx.demo` | `DEMO\nguyenvanb` |

#### 11.2. Từ win-02 (anhlx.demo) login user anhlx.lab

Cấu hình DNS trên `win-02` để phân giải được cả `anhlx.lab`:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 10.10.200.41, 10.10.200.31
```

Kiểm tra phân giải:

```powershell
nslookup anhlx.demo
nslookup anhlx.lab
```

Trên màn hình đăng nhập của `win-02`, có thể login bằng:

| Domain | Cách đăng nhập | Ví dụ |
|--------|---------------|-------|
| anhlx.demo (domain gốc) | `DEMO\username` hoặc `user@anhlx.demo` | `DEMO\Administrator` |
| anhlx.lab (cross-domain) | `ANHLX\username` hoặc `user@anhlx.lab` | `ANHLX\Administrator` |

**Tạo user test trên mỗi domain:**

```powershell
# Trên DC-A (anhlx.demo) - tạo user cho anhlx.demo
New-ADUser `
    -Name "Nguyen Van B" `
    -SamAccountName "nguyenvanb" `
    -UserPrincipalName "nguyenvanb@anhlx.demo" `
    -Path "CN=Users,DC=anhlx,DC=demo" `
    -AccountPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) `
    -Enabled $true
```

![Login cross-domain](../assets/img/2026-04-09-active-directory-additional-domain-tree/cross-domain-login.png)

**Kết quả mong đợi:**
- `win-2022` (join `anhlx.lab`): login được cả `ANHLX\user` và `DEMO\user`
- `win-02` (join `anhlx.demo`): login được cả `DEMO\user` và `ANHLX\user`

Nếu đăng nhập cross-domain thành công trên cả 2 client → trust giữa 2 domain tree đã hoạt động đúng.
