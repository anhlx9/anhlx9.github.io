---
canonical: https://anhlx2502.github.io/kubernetes/storage/cloud/2026/06/30/minio-aistor-dc-dr-multisite-lab.html
meta-article:author: Anh Le
meta-article:published_time: 2026-06-30T00:00:00+00:00
meta-author: Anh Le
meta-description: Lab MinIO AIStor Enterprise mô phỏng 2 site DC/DR — DC 4 node bare-metal EC4, DR 1 node k3s + Operator EC4, mã hóa đường truyền SSL/TLS toàn stack, mã hóa dữ liệu nghỉ qua Vault + KES, dữ liệu bất biến Object Lock chống ransomware, chứng thực tập trung Active Directory, one-way bucket replication, giám sát Prometheus/Grafana.
meta-og:description: Lab MinIO AIStor Enterprise mô phỏng 2 site DC/DR — DC 4 node bare-metal EC4, DR 1 node k3s + Operator EC4, mã hóa đường truyền SSL/TLS toàn stack, mã hóa dữ liệu nghỉ qua Vault + KES, dữ liệu bất biến Object Lock chống ransomware, chứng thực tập trung Active Directory, one-way bucket replication, giám sát Prometheus/Grafana.
meta-og:locale: en_US
meta-og:site_name: Anh Le Blog
meta-og:title: MinIO AIStor Enterprise DC/DR Multi-Site Lab — EC4, Vault/KES, AD, HAProxy, Prometheus
meta-og:type: article
meta-og:url: https://anhlx2502.github.io/kubernetes/storage/cloud/2026/06/30/minio-aistor-dc-dr-multisite-lab.html
meta-viewport: width=device-width, initial-scale=1.0
title: MinIO AIStor Enterprise DC/DR Multi-Site Lab — Anh Le Blog
---

# MinIO AIStor Enterprise: Lab DC/DR Multi-Site, EC4, Mã Hóa Tùy Chọn, Chứng Thực Tập Trung AD

*~ phút đọc*

MinIO AIStor là bản Enterprise của MinIO — object storage S3-compatible, bổ sung Site Replication, SSE-KMS qua KES/Vault, LDAP/AD identity, audit, hỗ trợ chính thức. Bài này mình build lab mô phỏng kiến trúc DC/DR thực tế mà nhiều SME/MSP đang cân nhắc khi chuyển từ "1 cụm object storage duy nhất" sang **multi-site có chứng thực tập trung, mã hóa tùy chọn theo bucket, và dữ liệu bất biến chống ransomware**.

Ý tưởng kiến trúc: **DC là cụm chính chạy 4 node bare-metal-style** (binary AIStor thuần, không qua Kubernetes) để mô phỏng on-prem truyền thống — đại diện cho cách nhiều MSP Việt Nam vẫn vận hành storage hiện nay. **DR là 1 node chạy k3s + AIStor Operator** để mô phỏng xu hướng "cloud Kubernetes" (VM trên nền ảo hóa, gắn thêm disk ảo, cài K8s lên trên). Ghi luôn vào DC, đọc từ DR — one-way bucket replication, không phải active-active. Toàn bộ lab áp dụng đủ 3 lớp bảo vệ dữ liệu: **mã hóa đường truyền** (SSL/TLS toàn stack, LDAPS, HTTPS), **mã hóa dữ liệu nghỉ** (SSE-KMS qua Vault/KES, tùy chọn theo bucket), và **bất biến dữ liệu** (Object Lock WORM chống xóa/sửa, kể cả khi bị ransomware mã hóa đè hay admin/insider thao tác nhầm).

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/architecture-overview.png)

Stack:

1. **AIStor Enterprise (trial 60 ngày)** — object storage, EC4 cả 2 site
2. **Active Directory (Windows Server 2025)** — AD DS + AD CS, chứng thực tập trung LDAPS cho cả 2 site, issue cert TLS nội bộ cho toàn bộ stack
3. **Vault + KES** — KMS backend cho SSE-KMS, mã hóa dữ liệu nghỉ tùy chọn theo bucket (có bucket mã hóa, có bucket không)
4. **Object Lock (WORM)** — bất biến dữ liệu mode Governance, chống ransomware/xóa nhầm, áp dụng cả 2 bucket và cả 2 site
5. **HAProxy** — load balancer L4/L7 trước 4 node DC, toàn bộ HTTPS
6. **k3s + AIStor Operator + DirectPV** — DR chạy trên Kubernetes, disk ảo gắn trực tiếp VM
7. **Prometheus + Grafana** — giám sát tập trung cả DC lẫn DR, dashboard và scrape đều qua HTTPS

> **Phạm vi bài**
>
> - Mình tập trung vào **luồng nghiệp vụ**: dựng 2 site, chứng thực tập trung AD, mã hóa tùy chọn theo bucket, dữ liệu bất biến chống ransomware, replication một chiều DC→DR, giám sát tập trung. Mình lược bỏ phần benchmark hiệu năng vì lab này không nhằm đo throughput.
> - License dùng bản **trial Enterprise 60 ngày** — đăng ký tại SUBNET (https://subnet.min.io), download license JWT để dùng cho cả 2 site.
> - Stack công nghệ và spec VM theo hạ tầng mình có. **Nếu áp dụng, bạn cần điều chỉnh spec VM, IP range, license cho phù hợp với môi trường của mình.**

---

## Mục lục

- [1. Kiến trúc & Spec Lab](#1-kiến-trúc--spec-lab)
- [2. Chuẩn bị OS — 7 VM](#2-chuẩn-bị-os--7-vm)
- [3. AD DS + AD CS trên Windows Server 2025](#3-ad-ds--ad-cs-trên-windows-server-2025)
- [4. Service Account, OU, LDAPS cho AIStor](#4-service-account-ou-ldaps-cho-aistor)
- [5. Issue Certificate từ AD CS cho AIStor](#5-issue-certificate-từ-ad-cs-cho-aistor)
- [6. Vault + KES + Prometheus + Grafana (Docker Compose)](#6-vault--kes--prometheus--grafana-docker-compose)
- [7. Cài AIStor Enterprise — 4 node DC, EC4](#7-cài-aistor-enterprise--4-node-dc-ec4)
- [8. HAProxy Load Balancer trước DC](#8-haproxy-load-balancer-trước-dc)
- [9. Đăng ký License Trial AIStor](#9-đăng-ký-license-trial-aistor)
- [10. Tích hợp LDAP/AD vào AIStor DC](#10-tích-hợp-ldapad-vào-aistor-dc)
- [11. KES/Vault & Object Lock — Mã Hóa Tùy Chọn, Dữ Liệu Bất Biến Chống Ransomware](#11-keסvault--object-lock--mã-hóa-tùy-chọn-dữ-liệu-bất-biến-chống-ransomware)
- [12. DR — k3s + AIStor Operator + DirectPV, EC4](#12-dr--k3s--aistor-operator--directpv-ec4)
- [13. One-way Bucket Replication DC → DR](#13-one-way-bucket-replication-dc--dr)
- [14. Giám sát Prometheus/Grafana cả 2 site](#14-giám-sát-prometheusgrafana-cả-2-site)
- [15. Demo & Test Logic](#15-demo--test-logic)
- [16. Service URLs & Credentials](#16-service-urls--credentials)
- [17. Lời kết](#17-lời-kết)

---

## 1. Kiến trúc & Spec Lab

### 1.1 Vì sao mô hình này đáng làm lab

SME/MSP Việt Nam thường có 1 cụm storage on-prem duy nhất, không có DR, identity quản lý rời rạc theo từng hệ thống. Lab này mô phỏng bước tiến hóa tự nhiên: thêm 1 site DR trên nền Kubernetes (xu hướng triển khai cloud phổ biến — VM rồi cài K8s lên trên), chứng thực tập trung qua AD sẵn có, mã hóa chọn lọc theo bucket thay vì mã hóa toàn bộ (tránh overhead không cần thiết cho dữ liệu không nhạy cảm).

Điểm khác biệt thật giữa 2 site mà bài sẽ thể hiện: DC dùng đĩa local gắn trực tiếp VM (latency thấp), DR tuy cũng gắn đĩa ảo trực tiếp vào VM nhưng nằm dưới một lớp k3s + DirectPV — kiến trúc vận hành khác hẳn (binary service vs Kubernetes Operator), đáng để so sánh thao tác quản trị giữa 2 mô hình.

### 1.2 Bảng VM và IP

| VM | Vai trò | IP | vCPU | RAM | Disk |
|---|---|---|---|---|---|
| **AD01** | Windows Server 2025 — AD DS + AD CS | 10.10.200.5 | 8 | 8 GB | 80 GB |
| **SVC01** | Ubuntu 24.04 — Vault + KES + HAProxy + Prometheus + Grafana (Docker Compose) | 10.10.200.10 | 8 | 8 GB | 100 GB |
| **DC01** | Ubuntu 24.04 — AIStor node 1 | 10.10.200.11 | 8 | 8 GB | OS 40GB + 4×50GB data |
| **DC02** | Ubuntu 24.04 — AIStor node 2 | 10.10.200.12 | 8 | 8 GB | OS 40GB + 4×50GB data |
| **DC03** | Ubuntu 24.04 — AIStor node 3 | 10.10.200.13 | 8 | 8 GB | OS 40GB + 4×50GB data |
| **DC04** | Ubuntu 24.04 — AIStor node 4 | 10.10.200.14 | 8 | 8 GB | OS 40GB + 4×50GB data |
| **DR01** | Ubuntu 24.04 — k3s + AIStor Operator + DirectPV | 10.10.200.21 | 8 | 8 GB | OS 40GB + 4×50GB data |

```
AD/LDAPS:          10.10.200.5    (636 — chỉ LDAPS, không mở 389)
Vault/KES/LB/Mon:  10.10.200.10   (Vault 8200, KES 7373, HAProxy 9000/9001/8404, Prometheus 9090, Grafana 3000 — tất cả HTTPS)
DC node 1-4:       10.10.200.11 - 10.10.200.14  (MinIO 9000/9001 mỗi node, HTTPS)
DR k3s node:       10.10.200.21  (expose qua NodePort HTTPS — chi tiết section 12)

Domain nội bộ:     anhlx.lab
DNS nội bộ:        10.10.200.5 (AD đảm nhiệm luôn DNS server)
```

> **Toàn bộ lab chạy HTTPS/TLS — không có dịch vụ nào nghe plain HTTP/LDAP.** Mọi cert (AIStor, Vault, KES, HAProxy, Prometheus, Grafana) đều issue từ AD CS Enterprise Root CA dựng ở section 3. Vì DNS đã trỏ về AD01, mọi client (kể cả `mc`/trình duyệt) resolve FQDN `*.anhlx.lab` qua AD — không cần sửa `/etc/hosts` thủ công sau khi AD DNS lên, chỉ cần trỏ resolver đúng (section 2). Test bằng trình duyệt thực hiện trên **AD01** (RDP vào AD01, mở Edge/Chrome gọi `https://...anhlx.lab`) vì AD01 đã tự động trust root CA của chính nó; test bằng `mc`/S3 API client thực hiện trên **SVC01** sau khi import root CA vào trust store (section 5.3).

> **ESXi:** mỗi VM Ubuntu cần thêm 4 đĩa ảo riêng (không chung VMDK với OS) cho DC01-04 và DR01 — tổng cộng 5 VM × 4 disk = 20 đĩa 50GB. VM AD01 và SVC01 chỉ cần 1 đĩa OS.

### 1.3 Sơ đồ luồng dữ liệu & xác thực

- **Ghi:** client → HAProxy (10.10.200.10:9000) → 1 trong 4 node DC → AIStor tính EC4, ghi phân tán 4 node.
- **Bất biến (Object Lock):** mọi object PUT vào `bucket-plain`/`bucket-encrypted` tự động gắn retention Governance 1 ngày — AIStor chặn mọi `DeleteObject`/`PutObject` đè lên object còn hạn, kể cả nếu malware có credential hợp lệ.
- **Mã hóa (tùy bucket):** AIStor DC → gọi KES (10.10.200.10:7373) → KES gọi Vault (10.10.200.10:8200) lấy/tạo data key → trả về AIStor để mã hóa object.
- **Xác thực:** user login Console/API → AIStor → LDAPS bind tới AD01 (10.10.200.5:636, TLS bắt buộc) → AD xác thực, trả nhóm/quyền.
- **Replicate:** DC (đã ghi xong, đã mã hóa nếu có, đã khóa WORM) → one-way bucket replication → DR (10.10.200.21), DR cũng tự áp Governance retention riêng cho bản sao của nó.
- **Đọc từ DR:** client đọc trực tiếp DR; nếu object mã hóa, DR cũng gọi cùng KES/Vault ở SVC01 để giải mã — đây là lý do bắt buộc dùng chung 1 KES cho cả 2 site.
- **Giám sát:** Prometheus (SVC01) scrape metrics từ cả 4 node DC và node DR.

---

## 2. Chuẩn bị OS — 7 VM

Áp dụng cho 5 VM Ubuntu 24.04 (SVC01, DC01-04, DR01). VM AD01 cài Windows Server 2025 riêng ở section 3.

```bash
# Swap off
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Chrony — đồng bộ giờ, AIStor cluster nhạy với clock skew
apt update && apt install -y chrony open-vm-tools curl wget vim git jq htop

tee /etc/chrony/chrony.conf <<'EOF'
server vn.pool.ntp.org iburst minpoll 4 maxpoll 6
makestep 1.0 -1
maxdistance 1.0
driftfile /var/lib/chrony/chrony.drift
rtcsync
EOF
systemctl restart chronyd
chronyc tracking | grep -E "System time|Leap status"

# sysctl cơ bản
tee /etc/sysctl.d/99-aistor.conf <<EOF
vm.swappiness            = 0
vm.max_map_count         = 262144
fs.file-max              = 2097152
net.core.somaxconn       = 32768
EOF
sysctl --system

# /etc/hosts — toàn bộ node trỏ AD làm DNS, nhưng vẫn set hosts cho chắc trước khi DNS lên
tee -a /etc/hosts <<EOF
10.10.200.5    ad01.anhlx.lab ad01
10.10.200.10   svc01.anhlx.lab svc01 vault.anhlx.lab kes.anhlx.lab minio-lb.anhlx.lab
10.10.200.11   dc01.anhlx.lab dc01
10.10.200.12   dc02.anhlx.lab dc02
10.10.200.13   dc03.anhlx.lab dc03
10.10.200.14   dc04.anhlx.lab dc04
10.10.200.21   dr01.anhlx.lab dr01
EOF

# Trỏ DNS resolver về AD (sau khi AD lên ở section 3, set lại resolved.conf)
mkdir -p /etc/systemd/resolved.conf.d
tee /etc/systemd/resolved.conf.d/dns.conf <<EOF
[Resolve]
DNS=10.10.200.5
Domains=anhlx.lab
EOF
systemctl restart systemd-resolved
```

Riêng DC01-04 và DR01, kiểm tra 4 đĩa ảo đã nhận diện đúng (chưa format, để nguyên cho AIStor/DirectPV format sau):

```bash
lsblk
# NAME   MAJ:MIN RM   SIZE RO TYPE
# sda      8:0    0    40G  0 disk   <- OS
# sdb      8:16   0    50G  0 disk   <- data 1
# sdc      8:32   0    50G  0 disk   <- data 2
# sdd      8:48   0    50G  0 disk   <- data 3
# sde      8:64   0    50G  0 disk   <- data 4
```

---

## 3. AD DS + AD CS trên Windows Server 2025

VM **AD01** (10.10.200.5) cài Windows Server 2025, hostname `AD01`, sau đó cài 2 role: **AD DS** (Domain Services) và **AD CS** (Certificate Services).

### 3.1 Đặt IP tĩnh và hostname

PowerShell (Run as Administrator):

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 10.10.200.5 -PrefixLength 24 -DefaultGateway 10.10.200.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 127.0.0.1

Rename-Computer -NewName "AD01" -Restart
```

### 3.2 Cài AD DS — tạo Forest mới `anhlx.lab`

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Import-Module ADDSDeployment
Install-ADDSForest `
  -DomainName "anhlx.lab" `
  -DomainNetbiosName "ANHLX" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd-DSRM-2026" -AsPlainText -Force) `
  -Force:$true
```

Máy sẽ tự reboot. Sau khi login lại, verify:

```powershell
Get-ADDomain
Get-Service ADWS, DNS, NTDS, Netlogon
```

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/01-addsforest.png)

### 3.3 Cài AD CS — Enterprise Root CA

```powershell
Install-WindowsFeature AD-Certificate, ADCS-Cert-Authority -IncludeManagementTools

Install-AdcsCertificationAuthority `
  -CAType EnterpriseRootCA `
  -CACommonName "ANHLX-Lab-Root-CA" `
  -KeyLength 2048 `
  -HashAlgorithmName SHA256 `
  -ValidityPeriod Years `
  -ValidityPeriodUnits 10 `
  -Force
```

Verify CA đã chạy:

```powershell
certutil -CAInfo
Get-Service CertSvc
```

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/02-adcs.png)

### 3.4 Tạo Certificate Template cho AIStor (Web Server + Client Auth)

Mở **Certificate Authority MMC** (`certsrv.msc`) → Certificate Templates → Manage:

1. Duplicate template **Web Server**.
2. Đặt tên `AIStor-TLS-Server`, General tab → Validity period: 2 years.
3. Subject Name tab → chọn "Supply in the request" (để CSR tự định nghĩa CN/SAN).
4. Extensions tab → Application Policies → đảm bảo có cả **Server Authentication** và **Client Authentication** (cần Client Auth vì AIStor↔KES dùng mTLS).
5. Security tab → thêm quyền **Enroll** cho group chứa các máy AIStor (hoặc Domain Computers nếu join domain; ở lab này các VM Ubuntu không join domain Windows nên dùng **standalone request** — xem section 5).

Publish template trong CA: chuột phải **Certificate Templates** → New → Certificate Template to Issue → chọn `AIStor-TLS-Server`.

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/03-certtemplate.png)

### 3.5 Bật Web Enrollment (để các VM Linux request cert qua HTTPS, không cần join domain)

```powershell
Install-WindowsFeature ADCS-Web-Enrollment -IncludeManagementTools
Install-AdcsWebEnrollment -Force
```

Trang enrollment sẽ có tại `https://ad01.anhlx.lab/certsrv` — dùng để request/download cert thủ công hoặc qua `certreq` từ Linux (section 5).

---

## 4. Service Account, OU, LDAPS cho AIStor

### 4.1 Tạo OU và Service Account bind LDAP

PowerShell trên AD01:

```powershell
Import-Module ActiveDirectory

New-ADOrganizationalUnit -Name "AIStor" -Path "DC=anhlx,DC=lab"
New-ADOrganizationalUnit -Name "Users" -Path "OU=AIStor,DC=anhlx,DC=lab"
New-ADOrganizationalUnit -Name "ServiceAccounts" -Path "OU=AIStor,DC=anhlx,DC=lab"
New-ADOrganizationalUnit -Name "Groups" -Path "OU=AIStor,DC=anhlx,DC=lab"

# Service account dùng để AIStor bind/lookup LDAP
New-ADUser -Name "svc-minio-ldap" `
  -SamAccountName "svc-minio-ldap" `
  -UserPrincipalName "svc-minio-ldap@anhlx.lab" `
  -Path "OU=ServiceAccounts,OU=AIStor,DC=anhlx,DC=lab" `
  -AccountPassword (ConvertTo-SecureString "Sv c-M1nio-LDAP-2026!" -AsPlainText -Force) `
  -PasswordNeverExpires $true `
  -Enabled $true

# Group cho user được phép truy cập AIStor (map sang Policy trong AIStor ở section 10)
New-ADGroup -Name "AIStor-Admins" -GroupScope Global -Path "OU=Groups,OU=AIStor,DC=anhlx,DC=lab"
New-ADGroup -Name "AIStor-ReadOnly" -GroupScope Global -Path "OU=Groups,OU=AIStor,DC=anhlx,DC=lab"

# User test
New-ADUser -Name "minio-admin-user" `
  -SamAccountName "minioadmin1" `
  -UserPrincipalName "minioadmin1@anhlx.lab" `
  -Path "OU=Users,OU=AIStor,DC=anhlx,DC=lab" `
  -AccountPassword (ConvertTo-SecureString "MinioUser-2026!" -AsPlainText -Force) `
  -PasswordNeverExpires $true `
  -Enabled $true

Add-ADGroupMember -Identity "AIStor-Admins" -Members "minioadmin1"
```

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/04-adou.png)

### 4.2 Bật LDAPS (636)

Vì AD CS là Enterprise CA tích hợp sẵn AD, AD01 **tự động nhận cert LDAPS** từ chính CA của nó sau khi cài AD CS (cert Domain Controller template tự enroll). Verify LDAPS đã hoạt động:

```powershell
# Kiểm tra cert đã gắn vào LDAP
Get-ChildItem -Path Cert:\LocalMachine\My | Where-Object {$_.Subject -like "*AD01*"}

# Test LDAPS bằng ldp.exe hoặc PowerShell
$port = 636
Test-NetConnection -ComputerName ad01.anhlx.lab -Port $port
```

Nếu chưa thấy cert tự động, request thủ công qua MMC `certlm.msc` → Personal → Request New Certificate → chọn template **Domain Controller** hoặc **Kerberos Authentication**.

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/05-ldaps.png)

---

## 5. Issue Certificate từ AD CS cho AIStor

Mỗi node AIStor (DC01-04, DR01, SVC01-KES) cần 1 cert TLS do AD CS issue, dùng cho network encryption. Vì các VM Ubuntu không join domain Windows, mình dùng `certreq`/OpenSSL tạo CSR rồi submit qua Web Enrollment.

### 5.1 Tạo CSR trên từng node Ubuntu

Chạy trên **mỗi** node DC01-04 và DR01 (đổi CN/SAN theo từng node):

```bash
mkdir -p /etc/minio/certs && cd /etc/minio/certs

# Ví dụ cho DC01 — đổi CN/IP cho DC02, DC03, DC04, DR01 tương ứng
openssl req -new -nodes -newkey rsa:2048 \
  -keyout private.key \
  -out dc01.csr \
  -subj "/C=VN/O=AnhLX-Lab/CN=dc01.anhlx.lab" \
  -addext "subjectAltName=DNS:dc01.anhlx.lab,DNS:dc01,IP:10.10.200.11"
```

### 5.2 Submit CSR lên AD CS (Web Enrollment / certreq thủ công)

Với Ubuntu không join domain, cách ổn định nhất cho lab là copy CSR sang AD01 và issue thủ công qua `certreq`:

```powershell
# Trên AD01, sau khi copy file dc01.csr từ Ubuntu sang (qua scp/winscp hoặc share)
certreq -submit -attrib "CertificateTemplate:AIStor-TLS-Server" dc01.csr dc01.cer

# Lấy luôn cert chain CA gốc
certutil -ca.cert root-ca.cer
```

Copy `dc01.cer` và `root-ca.cer` ngược lại về node Ubuntu:

```bash
# Trên DC01, sau khi nhận lại dc01.cer và root-ca.cer
cd /etc/minio/certs
mv dc01.cer public.crt
mv root-ca.cer CAs/ad01-root-ca.crt 2>/dev/null || { mkdir -p CAs && mv root-ca.cer CAs/ad01-root-ca.crt; }

# AIStor đọc cert/key theo cấu trúc cố định
ls /etc/minio/certs
# private.key  public.crt  CAs/ad01-root-ca.crt

chown -R minio-user:minio-user /etc/minio/certs 2>/dev/null || true
chmod 600 /etc/minio/certs/private.key
```

> Lặp lại bước 5.1 và 5.2 cho **DC02, DC03, DC04, DR01**, và cho **SVC01** (dùng cho KES server cert + HAProxy frontend cert + Prometheus/Grafana TLS). Đổi đúng CN/SAN theo IP từng node.

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/06-certreq.png)

### 5.3 Import Root CA vào trust store — bắt buộc để bỏ `--insecure`

Vì toàn bộ lab chạy HTTPS với cert do AD CS (CA nội bộ) issue, mọi client cần trust root CA này, nếu không `mc`/`curl`/trình duyệt sẽ báo lỗi chain không tin cậy. **AD01 tự động trust CA của chính nó** (không cần làm gì thêm) — đây là lý do bài chọn test bằng trình duyệt ngay trên AD01. Với **SVC01** (nơi chạy `mc` và Docker Compose), cần import thủ công:

```bash
# Trên SVC01 — copy root-ca.cer từ AD01 về (giống cách lấy ở bước 5.2)
cp root-ca.cer /usr/local/share/ca-certificates/anhlx-lab-root-ca.crt
update-ca-certificates
# Updating certificates in /etc/ssl/certs... 1 added

# Verify trust hoạt động — không còn cảnh báo cert
curl -v https://ad01.anhlx.lab/certsrv 2>&1 | grep -E "SSL certificate|subject"
```

Lặp lại import này trên **mọi node DC01-04 và DR01** (cần để chính AIStor binary verify lẫn nhau khi network encryption bật, và để DirectPV/k3s verify khi gọi LDAPS/KES):

```bash
cp /etc/minio/certs/CAs/ad01-root-ca.crt /usr/local/share/ca-certificates/anhlx-lab-root-ca.crt
update-ca-certificates
```

Từ đây trở đi, mọi lệnh `mc`/`curl` trong bài **không còn dùng `--insecure`** — cert được verify đầy đủ qua chain CA vừa import.

---

## 6. Vault + KES + Prometheus + Grafana (Docker Compose)

Toàn bộ chạy trên **SVC01** (10.10.200.10) bằng Docker Compose — Vault production mode (file storage, cần unseal thủ công), KES làm KMS gateway, Prometheus scrape cả DC/DR, Grafana dashboard.

### 6.1 Cài Docker

```bash
curl -fsSL https://get.docker.com | sh
usermod -aG docker $USER
systemctl enable --now docker
```

### 6.2 Tạo cert cho Vault và KES (issue từ AD CS — lặp lại quy trình section 5)

```bash
mkdir -p /opt/lab/{vault,kes,prometheus,grafana,haproxy}/{config,data,certs}
cd /opt/lab

# CSR cho Vault
openssl req -new -nodes -newkey rsa:2048 \
  -keyout vault/certs/vault.key \
  -out vault/certs/vault.csr \
  -subj "/C=VN/O=AnhLX-Lab/CN=vault.anhlx.lab" \
  -addext "subjectAltName=DNS:vault.anhlx.lab,DNS:svc01,IP:10.10.200.10"

# CSR cho KES
openssl req -new -nodes -newkey rsa:2048 \
  -keyout kes/certs/kes-server.key \
  -out kes/certs/kes-server.csr \
  -subj "/C=VN/O=AnhLX-Lab/CN=kes.anhlx.lab" \
  -addext "subjectAltName=DNS:kes.anhlx.lab,DNS:svc01,IP:10.10.200.10"

# CSR cho HAProxy (stats page + frontend nếu cần TLS termination http)
openssl req -new -nodes -newkey rsa:2048 \
  -keyout haproxy/certs/haproxy.key \
  -out haproxy/certs/haproxy.csr \
  -subj "/C=VN/O=AnhLX-Lab/CN=minio-lb.anhlx.lab" \
  -addext "subjectAltName=DNS:minio-lb.anhlx.lab,DNS:svc01.anhlx.lab,DNS:svc01,IP:10.10.200.10"

# HAProxy cần cert+key gộp 1 file (định dạng PEM combined) — ghép sau khi nhận cert ở 6.2b

# CSR cho Prometheus
openssl req -new -nodes -newkey rsa:2048 \
  -keyout prometheus/certs/prometheus.key \
  -out prometheus/certs/prometheus.csr \
  -subj "/C=VN/O=AnhLX-Lab/CN=svc01.anhlx.lab" \
  -addext "subjectAltName=DNS:svc01.anhlx.lab,DNS:svc01,IP:10.10.200.10"

# CSR cho Grafana
openssl req -new -nodes -newkey rsa:2048 \
  -keyout grafana/certs/grafana.key \
  -out grafana/certs/grafana.csr \
  -subj "/C=VN/O=AnhLX-Lab/CN=svc01.anhlx.lab" \
  -addext "subjectAltName=DNS:svc01.anhlx.lab,DNS:svc01,IP:10.10.200.10"

# Submit toàn bộ 4 CSR này qua AD01 (certreq -submit, giống section 5.2), nhận lại .cer
# Copy kết quả về:
#   vault/certs/vault.crt + vault/certs/ca.crt
#   kes/certs/kes-server.crt + kes/certs/ca.crt
#   haproxy/certs/haproxy.crt + haproxy/certs/ca.crt
#   prometheus/certs/prometheus.crt + prometheus/certs/ca.crt
#   grafana/certs/grafana.crt + grafana/certs/ca.crt
```

Sau khi nhận cert về, gộp cert+key cho HAProxy (yêu cầu bắt buộc của HAProxy — 1 file PEM duy nhất):

```bash
cat haproxy/certs/haproxy.crt haproxy/certs/haproxy.key > haproxy/certs/haproxy-combined.pem
chmod 600 haproxy/certs/haproxy-combined.pem
```

### 6.3 Vault config — production mode, file storage

```bash
tee /opt/lab/vault/config/vault.hcl <<'EOF'
storage "file" {
  path = "/vault/data"
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/vault/certs/vault.crt"
  tls_key_file  = "/vault/certs/vault.key"
}

api_addr     = "https://vault.anhlx.lab:8200"
cluster_addr = "https://vault.anhlx.lab:8201"
ui           = true
disable_mlock = true
EOF
```

### 6.4 KES config — trỏ Vault làm KMS backend

```bash
tee /opt/lab/kes/config/kes-config.yaml <<'EOF'
version: v1

address: 0.0.0.0:7373

tls:
  key: /kes/certs/kes-server.key
  cert: /kes/certs/kes-server.crt

admin:
  identity: "${KES_ADMIN_IDENTITY}"   # điền sau khi tạo client cert (section 6.7)

cache:
  expiry:
    any: 5m
    unused: 20s

keystore:
  vault:
    endpoint: "https://vault.anhlx.lab:8200"
    namespace: ""
    prefix: "kes"
    approle:
      id: "${VAULT_APPROLE_ID}"
      secret: "${VAULT_APPROLE_SECRET}"
      retry: 15s
    tls:
      ca: /kes/certs/ca.crt
EOF
```

> Giá trị `KES_ADMIN_IDENTITY`, `VAULT_APPROLE_ID`, `VAULT_APPROLE_SECRET` sẽ điền ở bước 6.7-6.8 sau khi Vault unseal và tạo AppRole xong — đây là thứ tự bắt buộc vì KES cần Vault sẵn sàng trước.

### 6.4b Prometheus & Grafana TLS config

Prometheus bind plain HTTP theo mặc định — cần thêm `web-config.yml` để bật TLS:

```bash
tee /opt/lab/prometheus/config/web-config.yml <<'EOF'
tls_server_config:
  cert_file: /etc/prometheus/certs/prometheus.crt
  key_file: /etc/prometheus/certs/prometheus.key
EOF
```

Grafana bật TLS qua biến môi trường trong compose (section 6.5), không cần file config riêng — chỉ cần mount cert vào container.

### 6.5 docker-compose.yml — Vault + KES + HAProxy + Prometheus + Grafana

```bash
tee /opt/lab/docker-compose.yml <<'EOF'
version: "3.8"

networks:
  labnet:
    driver: bridge

services:
  vault:
    image: hashicorp/vault:1.18
    container_name: vault
    cap_add:
      - IPC_LOCK
    networks:
      labnet:
        ipv4_address: 172.28.0.10
    ports:
      - "8200:8200"
    volumes:
      - ./vault/config/vault.hcl:/vault/config/vault.hcl:ro
      - ./vault/data:/vault/data
      - ./vault/certs:/vault/certs:ro
    command: vault server -config=/vault/config/vault.hcl
    restart: unless-stopped

  kes:
    image: minio/kes:latest
    container_name: kes
    networks:
      labnet:
        ipv4_address: 172.28.0.11
    ports:
      - "7373:7373"
    volumes:
      - ./kes/config/kes-config.yaml:/kes/config/kes-config.yaml:ro
      - ./kes/certs:/kes/certs:ro
    command: server --config=/kes/config/kes-config.yaml
    depends_on:
      - vault
    restart: unless-stopped

  haproxy:
    image: haproxy:2.9
    container_name: haproxy
    networks:
      labnet:
        ipv4_address: 172.28.0.12
    ports:
      - "9000:9000"   # MinIO S3 API LB (TCP passthrough TLS)
      - "9001:9001"   # MinIO Console LB (TCP passthrough TLS)
      - "8404:8404"   # HAProxy stats page (HTTPS)
    volumes:
      - ./haproxy/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
      - ./haproxy/certs:/usr/local/etc/haproxy/certs:ro
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    networks:
      labnet:
        ipv4_address: 172.28.0.13
    ports:
      - "9090:9090"
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--web.config.file=/etc/prometheus/certs/web-config.yml"
      - "--storage.tsdb.path=/prometheus"
    volumes:
      - ./prometheus/config/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/config/web-config.yml:/etc/prometheus/certs/web-config.yml:ro
      - ./prometheus/certs:/etc/prometheus/certs:ro
      - ./prometheus/data:/prometheus
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    networks:
      labnet:
        ipv4_address: 172.28.0.14
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=Grafana-Lab-2026!
      - GF_SERVER_PROTOCOL=https
      - GF_SERVER_CERT_FILE=/etc/grafana/certs/grafana.crt
      - GF_SERVER_CERT_KEY=/etc/grafana/certs/grafana.key
      - GF_SERVER_DOMAIN=svc01.anhlx.lab
    volumes:
      - ./grafana/data:/var/lib/grafana
      - ./grafana/certs:/etc/grafana/certs:ro
    depends_on:
      - prometheus
    restart: unless-stopped
EOF
```

### 6.6 Khởi động Vault, init và unseal thủ công

```bash
cd /opt/lab
docker compose up -d vault
sleep 5

export VAULT_ADDR="https://vault.anhlx.lab:8200"
export VAULT_CACERT="/opt/lab/vault/certs/ca.crt"

docker exec -it vault vault operator init -key-shares=5 -key-threshold=3 \
  > /opt/lab/vault-init-output.txt

cat /opt/lab/vault-init-output.txt
# Unseal Key 1: xxxx
# Unseal Key 2: xxxx
# Unseal Key 3: xxxx
# Unseal Key 4: xxxx
# Unseal Key 5: xxxx
# Initial Root Token: xxxx
```

> **Lưu file `vault-init-output.txt` ở nơi an toàn, ngoài VM này** — đây là production mode nên Vault thật sự bị "sealed" mỗi lần container restart, cần đúng 3/5 unseal key để mở lại. Mất file này coi như mất Vault.

Unseal (chạy 3 lần với 3 key khác nhau):

```bash
docker exec -it vault vault operator unseal   # nhập Unseal Key 1
docker exec -it vault vault operator unseal   # nhập Unseal Key 2
docker exec -it vault vault operator unseal   # nhập Unseal Key 3

docker exec -it vault vault status
# Sealed: false
```

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/07-vaultinit.png)

### 6.7 Bật KV secrets engine + AppRole cho KES

```bash
export VAULT_TOKEN="<Initial Root Token từ bước 6.6>"

docker exec -e VAULT_TOKEN=$VAULT_TOKEN -it vault vault secrets enable -path=kes kv-v2

docker exec -e VAULT_TOKEN=$VAULT_TOKEN -it vault vault auth enable approle

# Policy cho KES — chỉ cho phép thao tác trong path kes/*
docker exec -e VAULT_TOKEN=$VAULT_TOKEN -it vault vault policy write kes-policy - <<'EOF'
path "kes/data/*" {
  capabilities = ["create", "read", "update", "delete"]
}
path "kes/metadata/*" {
  capabilities = ["list", "read", "delete"]
}
EOF

docker exec -e VAULT_TOKEN=$VAULT_TOKEN -it vault vault write auth/approle/role/kes-role \
  token_policies="kes-policy" \
  token_ttl=1h \
  token_max_ttl=4h

docker exec -e VAULT_TOKEN=$VAULT_TOKEN -it vault vault read auth/approle/role/kes-role/role-id
# role_id: xxxx  -> điền vào VAULT_APPROLE_ID

docker exec -e VAULT_TOKEN=$VAULT_TOKEN -it vault vault write -f auth/approle/role/kes-role/secret-id
# secret_id: xxxx  -> điền vào VAULT_APPROLE_SECRET
```

### 6.8 Tạo client cert cho KES admin identity, hoàn tất config, start KES

```bash
# Client cert dùng để KES xác định "admin" — identity = SHA256 hash của public key cert
openssl req -new -nodes -newkey rsa:2048 \
  -keyout /opt/lab/kes/certs/admin.key \
  -out /opt/lab/kes/certs/admin.csr \
  -subj "/C=VN/O=AnhLX-Lab/CN=kes-admin"

# Submit qua AD CS như section 5, nhận admin.crt

# Lấy identity hash (yêu cầu kes CLI hoặc tính tay bằng openssl)
docker run --rm -v /opt/lab/kes/certs:/certs minio/kes:latest identity of /certs/admin.crt
# Identity: <hash> -> điền vào KES_ADMIN_IDENTITY trong kes-config.yaml
```

Cập nhật `kes-config.yaml` với 3 giá trị đã lấy được, rồi start KES:

```bash
sed -i "s|\${KES_ADMIN_IDENTITY}|<identity-hash-vừa-lấy>|" /opt/lab/kes/config/kes-config.yaml
sed -i "s|\${VAULT_APPROLE_ID}|<role_id>|" /opt/lab/kes/config/kes-config.yaml
sed -i "s|\${VAULT_APPROLE_SECRET}|<secret_id>|" /opt/lab/kes/config/kes-config.yaml

docker compose up -d kes
docker logs -f kes
# Should see: Starting kes server

# Tạo key mặc định cho AIStor dùng
docker run --rm --network lab_labnet \
  -v /opt/lab/kes/certs:/certs \
  minio/kes:latest key create \
  --key /certs/admin.key --cert /certs/admin.crt --ca /certs/ca.crt \
  -e https://kes.anhlx.lab:7373 \
  minio-default-key
```

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/08-kesready.png)

### 6.9 Prometheus config (scrape cả DC 4 node + DR)

```bash
tee /opt/lab/prometheus/config/prometheus.yml <<'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'aistor-dc'
    metrics_path: /minio/v2/metrics/cluster
    scheme: https
    tls_config:
      insecure_skip_verify: true
    static_configs:
      - targets:
          - 'dc01.anhlx.lab:9000'
        labels:
          site: 'dc'

  - job_name: 'aistor-dr'
    metrics_path: /minio/v2/metrics/cluster
    scheme: https
    tls_config:
      insecure_skip_verify: true
    static_configs:
      - targets:
          - 'dr01.anhlx.lab:9000'
        labels:
          site: 'dr'
EOF

docker compose up -d prometheus grafana
```

> Endpoint `/minio/v2/metrics/cluster` yêu cầu bearer token (Prometheus scrape token) — cách lấy token trình bày ở section 14 sau khi AIStor DC đã chạy.

---

## 7. Cài AIStor Enterprise — 4 node DC, EC4

Lặp lại các bước này trên **cả 4 node DC01-04**, chỉ đổi `node-ip`/hostname tương ứng.

### 7.1 Download và cài binary AIStor

```bash
# Trên mỗi node DC01-04
curl -O https://dl.min.io/aistor/minio/release/linux-amd64/minio
chmod +x minio
mv minio /usr/local/bin/

minio --version

# Tạo user hệ thống riêng cho minio
groupadd -r minio-user
useradd -M -r -g minio-user minio-user

# Format và mount 4 disk data (nếu lab muốn XFS giống production thật)
for d in b c d e; do
  mkfs.xfs /dev/sd${d}
  mkdir -p /mnt/disk${d}
  echo "/dev/sd${d}  /mnt/disk${d}  xfs  defaults,noatime  0  2" >> /etc/fstab
done
mount -a
lsblk -f

chown -R minio-user:minio-user /mnt/disk{b,c,d,e}
chown -R minio-user:minio-user /etc/minio/certs
```

### 7.2 Tạo file env cấu hình — `/etc/default/minio`

```bash
tee /etc/default/minio <<'EOF'
MINIO_VOLUMES="https://dc{01...04}.anhlx.lab:9000/mnt/disk{b,c,d,e}"
MINIO_OPTS="--console-address :9001 --certs-dir /etc/minio/certs"
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="MinioRoot-Lab-2026!"

# Storage class — EC4 nghĩa là 4 parity shard trên erasure-set 16 ổ (4 node x 4 disk)
MINIO_STORAGE_CLASS_STANDARD="EC:4"
EOF

chmod 600 /etc/default/minio
```

> **EC4 trên cấu hình 4 node × 4 disk = 16 ổ:** `MINIO_STORAGE_CLASS_STANDARD=EC:4` đặt parityShards=4 cho storage class STANDARD — tổng 16 ổ chia 4 data + 4 parity theo từng erasure-set, chịu được mất tối đa 4 ổ (hoặc 1 node hoàn toàn) mà vẫn đọc/ghi bình thường.

### 7.3 Tạo systemd unit

```bash
tee /etc/systemd/system/minio.service <<'EOF'
[Unit]
Description=MinIO AIStor
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
WorkingDirectory=/usr/local
User=minio-user
Group=minio-user
EnvironmentFile=-/etc/default/minio
ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
Restart=always
RestartSec=5
LimitNOFILE=1048576
TasksMax=infinity
OOMScoreAdjust=-1000
SendSIGKILL=no

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
```

### 7.4 Start đồng thời cả 4 node

Trên **mỗi node DC01-04** (start gần như cùng lúc để các node tìm thấy nhau lúc bootstrap cluster):

```bash
systemctl enable --now minio
journalctl -u minio -f
```

Verify cluster đã hình thành (chạy trên 1 node bất kỳ):

```bash
mc alias set dc https://dc01.anhlx.lab:9000 minioadmin 'MinioRoot-Lab-2026!'

mc admin info dc
# 4 node, mỗi node 4 drive, tổng dung lượng, storage class EC:4
```

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/09-aistorcluster.png)

---

## 8. HAProxy Load Balancer trước DC

Cấu hình trên **SVC01**, LB cho S3 API (9000) và Console (9001) của 4 node DC.

```bash
tee /opt/lab/haproxy/config/haproxy.cfg <<'EOF'
global
    log stdout format raw local0
    maxconn 4096

defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 5s
    timeout client  120s
    timeout server  120s

frontend s3_api_front
    bind *:9000
    mode tcp
    default_backend s3_api_back

backend s3_api_back
    mode tcp
    balance roundrobin
    option tcp-check
    server dc01 10.10.200.11:9000 check
    server dc02 10.10.200.12:9000 check
    server dc03 10.10.200.13:9000 check
    server dc04 10.10.200.14:9000 check

frontend console_front
    bind *:9001
    mode tcp
    default_backend console_back

backend console_back
    mode tcp
    balance roundrobin
    option tcp-check
    server dc01 10.10.200.11:9001 check
    server dc02 10.10.200.12:9001 check
    server dc03 10.10.200.13:9001 check
    server dc04 10.10.200.14:9001 check

frontend stats
    bind *:8404 ssl crt /usr/local/etc/haproxy/certs/haproxy-combined.pem
    mode http
    stats enable
    stats uri /
    stats refresh 10s
EOF

docker compose up -d haproxy
```

Verify qua VIP LB thay vì gọi thẳng node:

```bash
mc alias set dc-lb https://svc01.anhlx.lab:9000 minioadmin 'MinioRoot-Lab-2026!'
mc admin info dc-lb
```

Truy cập HAProxy stats (HTTPS): `https://svc01.anhlx.lab:8404` để xem trạng thái health-check 4 backend.

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/10-haproxystats.png)

> Vì mode `tcp` (L4 passthrough), TLS handshake đi xuyên qua HAProxy tới tận AIStor node — cert client thấy là cert của node AIStor (`dc01.anhlx.lab`...), không phải cert HAProxy. Nếu muốn cert đồng nhất 1 tên `minio-lb.anhlx.lab`, cần SAN cert AIStor bao gồm cả tên đó, hoặc chuyển HAProxy sang mode `http` với SSL termination — không làm trong lab này để giữ đơn giản.

---

## 9. Đăng ký License Trial AIStor

```bash
# Trên DC01 (hoặc bất kỳ node nào có mc alias dc)
mc license register dc
# Mở trình duyệt theo URL được in ra, login SUBNET, xác nhận đăng ký license trial 60 ngày

mc license info dc
# License: Enterprise Trial
# Expiry: <60 ngày kể từ hôm nay>
```

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/11-licensetrial.png)

> Sau khi đăng ký license ở 1 node, license áp dụng cho toàn cluster DC (4 node) vì chúng cùng 1 deployment. **DR là 1 deployment riêng (cluster k8s khác)** nên cần đăng ký license trial **lần thứ 2** riêng cho DR ở section 12.

---

## 10. Tích hợp LDAP/AD vào AIStor DC

```bash
mc idp ldap add dc \
  server_addr="ad01.anhlx.lab:636" \
  server_insecure="off" \
  tls_skip_verify="off" \
  lookup_bind_dn="cn=svc-minio-ldap,ou=ServiceAccounts,ou=AIStor,dc=anhlx,dc=lab" \
  lookup_bind_password="Svc-M1nio-LDAP-2026!" \
  user_dn_search_base_dn="ou=Users,ou=AIStor,dc=anhlx,dc=lab" \
  user_dn_search_filter="(sAMAccountName=%s)" \
  group_search_base_dn="ou=Groups,ou=AIStor,dc=anhlx,dc=lab" \
  group_search_filter="(member=%d)"

mc admin service restart dc
```

Verify LDAP đã bật:

```bash
mc idp ldap info dc
```

Tạo Policy và map vào AD group (group `AIStor-Admins` đã tạo ở section 4.1):

```bash
cat > /tmp/admin-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*"],
      "Resource": ["arn:aws:s3:::*"]
    }
  ]
}
EOF

mc admin policy create dc aistor-admin-policy /tmp/admin-policy.json

# Map policy vào DN của group AD (LDAP group DN, không phải tên thường)
mc idp ldap policy attach dc aistor-admin-policy \
  --group "cn=AIStor-Admins,ou=Groups,ou=AIStor,dc=anhlx,dc=lab"
```

Test login bằng user AD `minioadmin1`:

```bash
mc alias set dc-ldap https://svc01.anhlx.lab:9000 \
  --api "s3v4"
# Dùng STS AssumeRoleWithLDAPIdentity hoặc login qua Console UI bằng minioadmin1/MinioUser-2026!
```

Truy cập Console `https://svc01.anhlx.lab:9001`, chọn login bằng LDAP, nhập `minioadmin1` / `MinioUser-2026!` — verify đăng nhập thành công và thấy quyền theo policy đã map.

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/12-ldaplogin.png)

---

## 11. KES/Vault & Object Lock — Mã Hóa Tùy Chọn, Dữ Liệu Bất Biến Chống Ransomware

### 11.1 Cấu hình AIStor DC trỏ KES

```bash
tee -a /etc/default/minio <<'EOF'
MINIO_KMS_KES_ENDPOINT="https://kes.anhlx.lab:7373"
MINIO_KMS_KES_KEY_NAME="minio-default-key"
MINIO_KMS_KES_CERT_FILE="/etc/minio/certs/kes-client.crt"
MINIO_KMS_KES_KEY_FILE="/etc/minio/certs/kes-client.key"
MINIO_KMS_KES_CA_PATH="/etc/minio/certs/CAs/ad01-root-ca.crt"
EOF
```

> Cần tạo thêm 1 client cert riêng (`kes-client.crt/key`) cho từng node AIStor, issue qua AD CS giống section 5, và **thêm identity của từng cert này vào policy KES** (`kes-policy` ở Vault không quản chuyện này — KES tự quản identity-based policy riêng, cần `kes policy create` gán quyền `Create/Read/Generate` cho identity của từng node AIStor).

```bash
# Trên SVC01 — tạo policy KES cho phép AIStor tạo/đọc data key
docker run --rm --network lab_labnet \
  -v /opt/lab/kes/certs:/certs \
  minio/kes:latest policy create \
  --key /certs/admin.key --cert /certs/admin.crt --ca /certs/ca.crt \
  -e https://kes.anhlx.lab:7373 \
  minio-node-policy \
  --allow "/v1/key/create/*" \
  --allow "/v1/key/generate/*" \
  --allow "/v1/key/decrypt/*"

# Gán policy này cho identity của từng cert kes-client.crt (4 node DC + 1 node DR)
docker run --rm --network lab_labnet \
  -v /opt/lab/kes/certs:/certs \
  minio/kes:latest identity of /certs/dc01-kes-client.crt
# Lấy hash, rồi:
docker run --rm --network lab_labnet \
  -v /opt/lab/kes/certs:/certs \
  minio/kes:latest policy assign minio-node-policy \
  --identity "<hash-dc01>" \
  --key /certs/admin.key --cert /certs/admin.crt --ca /certs/ca.crt \
  -e https://kes.anhlx.lab:7373

# Lặp lại assign cho dc02, dc03, dc04, dr01
```

Restart AIStor sau khi thêm KES env:

```bash
systemctl restart minio
journalctl -u minio -f
# Tìm dòng: KMS: configured using KES
```

### 11.2 Tạo bucket — Object Lock (WORM) + Versioning, mã hóa tùy chọn

Object Lock **bắt buộc bật ngay lúc tạo bucket** — không thể bật bổ sung cho bucket đã tồn tại. Cờ `--with-lock` tự động bật kèm Versioning (Object Lock yêu cầu versioning để hoạt động).

```bash
mc mb dc/bucket-plain --with-lock
mc mb dc/bucket-encrypted --with-lock

# Verify Object Lock + Versioning đã bật
mc version info dc/bucket-plain
# Versioning: Enabled
mc retention info dc/bucket-plain
# (chưa set default — sẽ set ở bước dưới)

# Bật SSE-KMS mặc định cho bucket-encrypted — mọi object PUT vào đây tự động mã hóa
mc encrypt set sse-kms minio-default-key dc/bucket-encrypted

mc encrypt info dc/bucket-encrypted
# Encryption Type: sse-kms, Key: minio-default-key

mc encrypt info dc/bucket-plain
# auto encryption is not configured
```

Đặt **default retention** cho cả 2 bucket — mode Governance, 1 ngày, áp dụng tự động cho mọi object mới upload (không cần chỉ định retention thủ công từng lần):

```bash
mc retention set governance 1d dc/bucket-plain --default
mc retention set governance 1d dc/bucket-encrypted --default

mc retention info dc/bucket-plain
# Mode: governance, Duration: 1 day, Status: ON
```

> **Governance vs Compliance:** Governance cho phép user có quyền `s3:BypassGovernanceRetention` (thường là root hoặc policy đặc biệt) override để xóa/sửa sớm hơn hạn — phù hợp lab test vì không bị khóa cứng nếu cần dọn dẹp. Compliance khóa tuyệt đối kể cả root, đúng nghĩa WORM chống ransomware/insider, nên dùng cho production thật khi cần tuân thủ pháp lý (lưu trữ hồ sơ, backup chống mã hóa tống tiền).

Test upload và verify:

```bash
echo "plain data - khong ma hoa" > /tmp/test-plain.txt
echo "secret data - co ma hoa" > /tmp/test-encrypted.txt

mc cp /tmp/test-plain.txt dc/bucket-plain/
mc cp /tmp/test-encrypted.txt dc/bucket-encrypted/

mc stat dc/bucket-plain/test-plain.txt
# Không có trường Encryption, có trường Retention: GOVERNANCE, Retain Until: <+1 ngày>

mc stat dc/bucket-encrypted/test-encrypted.txt
# Encryption: SSE-KMS, Key ID: minio-default-key
# Retention: GOVERNANCE, Retain Until: <+1 ngày>
```

Test thử xóa object trong lúc còn hạn retention (kỳ vọng lỗi):

```bash
mc rm dc/bucket-plain/test-plain.txt
# mc: <ERROR> Failed to remove ... Object is WORM protected and cannot be overwritten

# Vì mode Governance, root có thể bypass nếu thật sự cần xóa sớm
mc rm dc/bucket-plain/test-plain.txt --bypass-governance-retention
# Xóa thành công — đúng đặc tính Governance (Compliance sẽ không cho phép dù có cờ này)
```

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/13-bucketencryption.png)

---

## 12. DR — k3s + AIStor Operator + DirectPV, EC4

Trên **DR01** (10.10.200.21) — VM có 4 đĩa ảo gắn trực tiếp giống DC, nhưng chạy qua k3s thay vì binary trực tiếp.

### 12.1 Cài k3s (single-node, disable local-path mặc định)

```bash
curl -sfL https://get.k3s.io | sh -s - \
  --disable local-storage \
  --write-kubeconfig-mode 644 \
  --tls-san dr01.anhlx.lab

mkdir -p ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc

kubectl get nodes
# dr01   Ready   control-plane,master   1m   v1.3x.x+k3s1
```

### 12.2 Cài DirectPV — quản lý 4 đĩa ảo

```bash
curl -sfL https://github.com/minio/directpv/releases/latest/download/kubectl-directpv_linux_amd64 \
  -o kubectl-directpv
chmod +x kubectl-directpv
mv kubectl-directpv /usr/local/bin/

kubectl directpv install

kubectl directpv discover
# Hiển thị /dev/sdb, /dev/sdc, /dev/sdd, /dev/sde chưa init

kubectl directpv init drives.yaml --dangerous

kubectl directpv list drives
# 4 drive STATUS=Ready
```

### 12.3 Cài AIStor Operator (Helm)

```bash
helm repo add minio https://operator.min.io/
helm repo update

helm install aistor-operator minio/aistor-objectstore-operator \
  --namespace aistor --create-namespace \
  --set license="<PASTE_LICENSE_JWT_TỪ_SUBNET>"

kubectl -n aistor rollout status deployment/aistor-operator
```

> Nếu chưa có license JWT cho deployment DR (license của DC không áp dụng cho cluster k8s riêng), đăng ký license trial **lần 2** trên SUBNET — chọn loại deployment "Kubernetes", lấy JWT token dán vào `--set license=`.

### 12.4 Tạo namespace + cert Secret cho ObjectStore

```bash
kubectl create namespace dr-objectstore

# Cert đã issue từ AD CS cho dr01 (section 5) — đưa vào Secret
kubectl -n dr-objectstore create secret generic dr01-tls \
  --from-file=public.crt=/etc/minio/certs/public.crt \
  --from-file=private.key=/etc/minio/certs/private.key \
  --from-file=ca.crt=/etc/minio/certs/CAs/ad01-root-ca.crt

# Secret cho LDAP bind password
kubectl -n dr-objectstore create secret generic dr-ldap-secret \
  --from-literal=password='Svc-M1nio-LDAP-2026!'

# Secret cho KES client cert (đã issue + assign policy ở section 11.1)
kubectl -n dr-objectstore create secret generic dr-kes-client \
  --from-file=kes-client.crt=/etc/minio/certs/dr01-kes-client.crt \
  --from-file=kes-client.key=/etc/minio/certs/dr01-kes-client.key
```

### 12.5 ObjectStore CR — EC4, 1 node 4 volume, LDAP + KES

```bash
tee /opt/lab/dr-objectstore-values.yaml <<'EOF'
objectStore:
  name: dr-object-store
  pools:
    - name: pool-0
      servers: 1
      volumesPerServer: 4
      size: 50Gi
      storageClassName: directpv-min-io

  storageClass:
    standard: "EC:4"

  certSecret: dr01-tls

  configuration:
    env:
      - name: MINIO_IDENTITY_LDAP_SERVER_ADDR
        value: "ad01.anhlx.lab:636"
      - name: MINIO_IDENTITY_LDAP_LOOKUP_BIND_DN
        value: "cn=svc-minio-ldap,ou=ServiceAccounts,ou=AIStor,dc=anhlx,dc=lab"
      - name: MINIO_IDENTITY_LDAP_LOOKUP_BIND_PASSWORD
        valueFrom:
          secretKeyRef:
            name: dr-ldap-secret
            key: password
      - name: MINIO_IDENTITY_LDAP_USER_DN_SEARCH_BASE_DN
        value: "ou=Users,ou=AIStor,dc=anhlx,dc=lab"
      - name: MINIO_IDENTITY_LDAP_USER_DN_SEARCH_FILTER
        value: "(sAMAccountName=%s)"
      - name: MINIO_IDENTITY_LDAP_GROUP_SEARCH_BASE_DN
        value: "ou=Groups,ou=AIStor,dc=anhlx,dc=lab"
      - name: MINIO_IDENTITY_LDAP_GROUP_SEARCH_FILTER
        value: "(member=%d)"
      - name: MINIO_KMS_KES_ENDPOINT
        value: "https://kes.anhlx.lab:7373"
      - name: MINIO_KMS_KES_KEY_NAME
        value: "minio-default-key"

  services:
    minio:
      serviceType: NodePort
      nodePort: 30900
    console:
      serviceType: NodePort
      nodePort: 30901
EOF

helm install dr-objectstore minio/aistor-objectstore \
  --namespace dr-objectstore \
  -f /opt/lab/dr-objectstore-values.yaml

kubectl -n dr-objectstore get pods -w
```

> EC4 trên 1 node × 4 disk: parityShards=4 trên 4 ổ nghĩa là **mọi ổ đều là parity hoặc data tùy block**, thực chất với đúng 4 ổ cấu hình EC:4 sẽ tự hạ về mức tối đa hệ thống cho phép trên 1 node (kiểm tra qua `mc admin info` sau khi lên — AIStor tự điều chỉnh nếu EC:4 không khả thi với 4 ổ, xem ghi chú verify bên dưới).

### 12.6 Verify DR cluster + đăng ký license

```bash
mc alias set dr https://dr01.anhlx.lab:30900 minioadmin '<password-tự-sinh-xem-Secret>'

# Lấy root credentials Operator tự sinh
kubectl -n dr-objectstore get secret dr-object-store-env-configuration \
  -o jsonpath='{.data.config\.env}' | base64 -d

mc admin info dr
# 1 node, 4 drive, storage class hiển thị EC:4 hoặc mức tự điều chỉnh

mc license register dr
mc license info dr
```

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/14-drobjectstore.png)

---

## 13. One-way Bucket Replication DC → DR

Replicate cả 2 bucket (plain + encrypted) để demo cả 2 trường hợp. Versioning ở DC đã bật sẵn từ lúc `mc mb --with-lock` (section 11.2), không cần bật lại.

```bash
# Tạo bucket tương ứng bên DR — cũng với-lock để giữ tính bất biến đồng nhất 2 site
mc mb dr/bucket-plain --with-lock
mc mb dr/bucket-encrypted --with-lock

mc retention set governance 1d dr/bucket-plain --default
mc retention set governance 1d dr/bucket-encrypted --default

# Lấy access/secret key của DR để remote target dùng
DR_ACCESS=minioadmin
DR_SECRET='<password-DR-từ-Secret>'

mc replicate add dc/bucket-plain \
  --remote-bucket "https://${DR_ACCESS}:${DR_SECRET}@dr01.anhlx.lab:30900/bucket-plain" \
  --replicate "delete,delete-marker,existing-objects"

mc replicate add dc/bucket-encrypted \
  --remote-bucket "https://${DR_ACCESS}:${DR_SECRET}@dr01.anhlx.lab:30900/bucket-encrypted" \
  --replicate "delete,delete-marker,existing-objects"
```

Verify trạng thái replication:

```bash
mc replicate status dc/bucket-plain
mc replicate status dc/bucket-encrypted
```

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/15-replicatestatus.png)

> Object trong `bucket-encrypted` khi replicate sang DR vẫn ở dạng ciphertext (không giải mã rồi mã hóa lại) — DR đọc được vì cùng trỏ 1 KES/Vault key `minio-default-key` ở SVC01.
>
> **Object Lock và flag `delete` trong replicate:** vì cả 2 site đều có Governance retention, một thao tác xóa hợp lệ ở DC (sau khi hết hạn retention, hoặc bypass bằng quyền đặc biệt) sẽ đồng bộ xóa xuống DR nhờ flag `delete`. Nhưng nếu object ở DR còn trong thời hạn retention riêng của nó (tính độc lập từ lúc object đó được ghi vào DR qua replication, không phải từ lúc ghi gốc ở DC), thao tác xóa đồng bộ có thể bị WORM tại DR chặn lại — đây là hành vi đúng, không phải lỗi: **DR giữ object an toàn ngay cả khi DC bị xóa nhầm/bị ransomware tấn công**, miễn retention DR chưa hết hạn.

---

## 14. Giám sát Prometheus/Grafana cả 2 site

### 14.1 Tạo Prometheus scrape token

```bash
mc admin prometheus generate dc
# Lưu Bearer Token, dán vào prometheus.yml (DC) — section 6.9 đã set sẵn target,
# bổ sung authorization header:

mc admin prometheus generate dr
# Bearer Token riêng cho DR
```

Cập nhật `prometheus.yml` thêm bearer token:

```bash
tee -a /opt/lab/prometheus/config/prometheus.yml <<'EOF'
    bearer_token: "<TOKEN_DC_TỪ_LỆNH_TRÊN>"
EOF
# Sửa thủ công đúng vị trí job 'aistor-dc' và 'aistor-dr' tương ứng — copy lại toàn bộ
# file nếu cần format YAML đúng (mỗi job 1 token riêng)

docker compose restart prometheus
```

Verify targets UP: `https://svc01.anhlx.lab:9090/targets`

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/16-prometheustargets.png)

### 14.2 Import Grafana dashboard MinIO chính thức

Truy cập `https://svc01.anhlx.lab:3000`, login `admin` / `Grafana-Lab-2026!`:

1. Add Data Source → Prometheus → URL `https://prometheus:9090` (tên service trong docker network) → bật **Skip TLS Verify** (vì cert Prometheus issue cho `svc01.anhlx.lab`, không khớp tên service nội bộ `prometheus`) → Save & Test.
2. Dashboards → Import → nhập ID **13502** (MinIO Dashboard chính thức) → chọn data source vừa tạo.
3. Dashboard hiển thị theo label `site` (dc/dr) — lọc riêng từng site qua biến template.

![](https://anhle.com.vn/assets/img/2026-06-30-minio-aistor-dc-dr-lab/17-grafanadashboard.png)

---

## 15. Demo & Test Logic

### 15.1 Test chứng thực tập trung AD

```bash
# Login Console DC bằng user AD, verify quyền theo policy map
# Login Console DR bằng CÙNG user AD đó — verify cùng identity, không cần tạo lại user
```

### 15.2 Test mã hóa tùy chọn theo bucket

```bash
# Upload vào bucket-plain — không mã hóa
# Upload vào bucket-encrypted — tự động mã hóa qua KES
# So sánh mc stat 2 object để thấy khác biệt trường Encryption
```

### 15.2b Test dữ liệu bất biến — mô phỏng ransomware

Mô phỏng kịch bản ransomware/insider có credential hợp lệ cố tình ghi đè hoặc xóa object để tống tiền:

```bash
echo "du lieu goc - quan trong" > /tmp/important.txt
mc cp /tmp/important.txt dc/bucket-encrypted/important.txt
mc retention info dc/bucket-encrypted/important.txt
# Mode: GOVERNANCE, Retain Until: <+1 ngày từ giờ upload>

# Mô phỏng ransomware: cố ghi đè (PutObject overwrite) bằng nội dung mã hóa giả
echo "ENCRYPTED_BY_RANSOMWARE_PAY_NOW" > /tmp/ransom.txt
mc cp /tmp/ransom.txt dc/bucket-encrypted/important.txt
# Vì versioning bật, lệnh này thực chất tạo VERSION MỚI chứ không ghi đè version cũ —
# verify version gốc vẫn còn nguyên:
mc ls --versions dc/bucket-encrypted/important.txt
# 2 version: version gốc (giữ nguyên, còn WORM) + version "ransomware" mới

# Mô phỏng ransomware cố xóa hẳn version gốc để xóa dấu vết
mc rm --version-id <version-id-gốc> dc/bucket-encrypted/important.txt
# mc: <ERROR> Object is WORM protected and cannot be modified or deleted

# Thậm chí root cũng cần cờ bypass tường minh mới xóa được (Governance) —
# ransomware tự động hóa thông thường sẽ không biết/không có quyền này
mc rm --version-id <version-id-gốc> dc/bucket-encrypted/important.txt --bypass-governance-retention
# Chỉ admin chủ động và có chủ đích mới làm được, không xảy ra "vô tình" hay tự động
```

> Đây chính là giá trị cốt lõi của Object Lock trước ransomware: kẻ tấn công dù chiếm được access key hợp lệ và gọi đúng API xóa/ghi đè, AIStor vẫn từ chối ở tầng storage — không phụ thuộc việc OS hay ứng dụng phía trên có bị xâm nhập hay không. Bản sao tại DR (cũng WORM) là lớp phòng thủ thứ hai nếu kẻ tấn công bằng cách nào đó có quyền bypass ở DC.

### 15.3 Test replication 1 chiều

```bash
mc cp /tmp/newfile.txt dc/bucket-encrypted/
sleep 5
mc ls dr/bucket-encrypted/
# newfile.txt phải xuất hiện ở DR

# Test chiều ngược lại KHÔNG đồng bộ (đúng tinh thần one-way)
mc cp /tmp/dr-only-file.txt dr/bucket-encrypted/
sleep 5
mc ls dc/bucket-encrypted/
# dr-only-file.txt KHÔNG xuất hiện ở DC — xác nhận đúng one-way
```

### 15.4 Test tắt KES → quan sát hành vi

```bash
docker stop kes
mc cp /tmp/test2.txt dc/bucket-encrypted/
# Kỳ vọng: lỗi "KMS is not configured" hoặc timeout — object KHÔNG ghi được vào bucket mã hóa
mc cp /tmp/test2.txt dc/bucket-plain/
# Kỳ vọng: vẫn ghi bình thường — bucket-plain không phụ thuộc KES

docker start kes
```

### 15.5 Test failover đọc từ DR khi DC down

```bash
systemctl stop minio   # chạy trên cả 4 node DC để mô phỏng DC mất hoàn toàn
mc ls dr/bucket-encrypted/
# Vẫn đọc được — DR hoạt động độc lập, dữ liệu đã replicate trước đó còn nguyên

# Khởi động lại DC, verify resync queue tự xử lý phần chênh lệch nếu có ghi trong lúc DC down
systemctl start minio
mc replicate status dc/bucket-encrypted
```

---

## 16. Service URLs & Credentials

| Service | URL | User/Pass | Test từ đâu |
|---|---|---|---|
| AD DS / DNS | `ad01.anhlx.lab` | `Administrator` (domain) | RDP AD01 |
| AD CS Web Enrollment | `https://ad01.anhlx.lab/certsrv` | domain admin | Trình duyệt trên AD01 |
| Vault UI | `https://vault.anhlx.lab:8200/ui` | root token (file `vault-init-output.txt`) | Trình duyệt trên AD01 |
| KES | `https://kes.anhlx.lab:7373` | mTLS client cert | `kes` CLI trên SVC01 |
| HAProxy stats | `https://svc01.anhlx.lab:8404` | — | Trình duyệt trên AD01 |
| Prometheus | `https://svc01.anhlx.lab:9090` | — | Trình duyệt trên AD01 |
| Grafana | `https://svc01.anhlx.lab:3000` | `admin` / `Grafana-Lab-2026!` | Trình duyệt trên AD01 |
| AIStor DC Console (qua LB) | `https://svc01.anhlx.lab:9001` | `minioadmin` / `MinioRoot-Lab-2026!` hoặc LDAP user | Trình duyệt trên AD01 |
| AIStor DC node trực tiếp | `https://dc0{1-4}.anhlx.lab:9001` | như trên | Trình duyệt trên AD01 |
| AIStor DR Console | `https://dr01.anhlx.lab:30901` | root từ Secret Operator sinh | Trình duyệt trên AD01 |
| S3 API (DC qua LB / DR) | `https://svc01.anhlx.lab:9000` / `https://dr01.anhlx.lab:30900` | access/secret key | `mc` trên SVC01 |

> Toàn bộ URL trên đều resolve được nhờ DNS đã trỏ về AD01 (section 2) — không cần `/etc/hosts` thủ công ngoài bước ban đầu trước khi AD DNS sẵn sàng. Trình duyệt trên AD01 hiển thị khóa xanh ngay (tự trust CA của chính nó); `mc` trên SVC01 cũng không còn cảnh báo cert sau khi import root CA ở section 5.3.

---

## 17. Lời kết

Lab này mô phỏng đúng pattern hybrid mà nhiều MSP/SME Việt Nam đang tiến tới: giữ cụm chính on-prem ổn định, mở rộng thêm site DR trên nền Kubernetes (xu hướng triển khai cloud phổ biến hiện nay), chứng thực tập trung qua AD sẵn có thay vì quản lý identity rời rạc, và áp đủ 3 lớp bảo vệ dữ liệu thay vì chỉ dừng ở "có HTTPS là đủ": mã hóa đường truyền (SSL/TLS toàn stack), mã hóa dữ liệu nghỉ (SSE-KMS chọn lọc theo bucket), và bất biến dữ liệu (Object Lock WORM) — ba lớp này độc lập nhau, mất 1 lớp không có nghĩa mất an toàn hoàn toàn, nhưng thiếu lớp WORM thì TLS và mã hóa đĩa vẫn không ngăn được ransomware có credential hợp lệ xóa/ghi đè dữ liệu.

Vài điểm rút ra đáng ghi nhớ khi vận hành thật: KMS và identity provider là **single point of failure ẩn** một khi đã centralize — KES/Vault hay AD down không làm mất dữ liệu cũ nhưng chặn mọi thao tác ghi mới vào bucket mã hóa và mọi login mới; one-way replication chỉ bảo vệ một chiều, DR không phải bản sao có thể ghi ngược; EC4 trên 1 node DR vẫn không thay thế được tính chịu lỗi thật của EC4 trên 4 node DC — DR ở đây đóng vai trò "bản sao đọc được", không phải HA thật sự; và Governance retention dù chống được ransomware tự động, vẫn có thể bị admin/insider có quyền bypass chủ động vượt qua — production thật cần cân nhắc Compliance mode cho dữ liệu thật sự quan trọng (hồ sơ pháp lý, backup chống tống tiền), chấp nhận đánh đổi không ai sửa được kể cả khi cần thiết.

Bước tiếp theo nếu muốn nâng lab này lên gần production hơn: thêm DR thứ 2 (multi-destination replication), benchmark throughput qua VPN/WAN thật giữa 2 site, thử nghiệm Site Replication active-active thay vì one-way nếu nhu cầu chuyển sang multi-site đối xứng, và chuyển một phần bucket nhạy cảm sang Compliance mode để đo thực tế độ "cứng" khi không ai — kể cả chính admin — override được.
