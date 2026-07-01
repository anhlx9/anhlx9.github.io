---
title: "MinIO AIStor Enterprise — Cluster HA, AD-IAM, Mã hóa riêng từng bucket & Immutable chống Ransomware"
categories:
- Storage
- MinIO-AIStor-Subscription
tags:
- minio
- aistor
- object-storage
- active-directory
- ransomware
- vault
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Cluster 4-node EC:4, AD làm IAM, mã hóa riêng từng bucket & WORM Compliance
---

**MinIO** là object storage mã nguồn mở, tương thích S3 API, kiến trúc distributed gọn nhẹ — chạy trên cụm node thường, dùng erasure coding để vừa chống lỗi vừa cho throughput cao. Khác với **Ceph** (nền tảng lưu trữ hợp nhất object/block/file qua RADOS — mạnh, linh hoạt nhưng nặng vận hành, nhiều thành phần MON/OSD/MGR), MinIO **chỉ tập trung object S3**: ít thành phần, triển khai nhanh, độ trễ thấp và dễ vận hành hơn — đổi lại không cung cấp block/file storage.

**MinIO AIStor** là bản thương mại của MinIO, thiết kế cho các hệ thống **tích hợp AI hoạt động hiệu suất cao** — phục vụ pipeline train/inference cần đọc dataset và ghi checkpoint/model với băng thông lớn, song song nhiều luồng. Kèm theo là bộ tính năng cấp doanh nghiệp mà bản community không có: scale cluster, KMS, audit, replication, cùng các cơ chế bảo vệ dữ liệu chặt chẽ.

Lab này dựng **cluster 4 node HA, EC:4** để POC các tính năng đó — 16 ổ chia **12 data + 4 parity**, dung lượng khả dụng ~75%, **chịu mất đồng thời tới 4 disk (tương đương trọn 1 node)** mà cluster vẫn đọc/ghi bình thường. Production cần dung sai cao hơn có thể cân nhắc **EC:6** (chịu mất 6 disk, đổi lại dung lượng khả dụng còn ~62%). Trọng tâm POC:

- **AD làm IAM** — xác thực LDAPS, phân quyền riêng từng bucket.
- **Mã hóa at-rest, khóa riêng từng bucket** — mỗi bucket nhạy cảm một khóa Vault riêng qua KES (mọi object luôn được mã hóa).
- **Versioning** — giữ nhiều phiên bản object, rollback được.
- **Object Lock Compliance mode** — lớp bất biến chống ransomware mà *kể cả root cũng không xóa được* trong thời hạn retention.

Bộ tính năng này đặc biệt phù hợp cho **doanh nghiệp mảng tài chính**: phân quyền chặt, mã hóa cô lập theo phòng ban, và lưu trữ bất biến đáp ứng yêu cầu tuân thủ.

> Cụm 4 node là tính năng thương mại nên cần **license** (lab này dùng bản 60 ngày). Bản community chỉ chạy single-node.

> **Ba hướng triển khai production:**
> - **Bare-metal:** MinIO cài thẳng trên server vật lý + NVMe JBOD — throughput/độ trễ tốt nhất, ít tầng trung gian.
> - **Ảo hóa (≥2 ESXi):** mỗi ESXi host gắn SSD/NVMe local làm datastore, tạo VM minio + virtual disk (VMDK) trên datastore đó, dùng **anti-affinity** để mỗi minio node nằm trên một host vật lý khác → mất 1 host vẫn còn quorum. Vừa giữ được **hạ tầng ảo hóa** (chạy thêm workload khác trên cụm ESXi) vừa có **cụm MinIO AIStor** trên storage SSD/NVMe. Lab này chính là bản thu nhỏ của hướng B — chỉ khác là dồn hết VM trên 1 host nên host là SPOF.
> - **Kubernetes (MinIO Operator):** khai báo tenant bằng CRD, Operator tự dựng StatefulSet + PVC (dùng **DirectPV / local NVMe PV** để bám ổ vật lý), hợp môi trường đã chuẩn hoá K8s, cần multi-tenant self-service + GitOps. **Đánh đổi:** thêm tầng trừu tượng (CNI, CSI, container) → **throughput/độ trễ thấp hơn bare-metal**, vận hành phức tạp hơn (cần cụm K8s khỏe, quản lý vòng đời Operator, CSI cho local disk). **Chỉ dùng khi thật sự cần** tính khai báo/đa tenant trên K8s — ưu tiên hiệu năng thuần thì chọn A hoặc B.



![](/assets/img/2026-07-01-minio-aistor-enterprise/topo.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/55.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/56.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/57.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/58.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/59.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/60.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/61.png)

- [Mục tiêu](#mục-tiêu)
- [Môi trường](#môi-trường)
- [Chuẩn bị](#chuẩn-bị)
- [Kiến trúc tổng thể](#kiến-trúc-tổng-thể)
- [Bảng VM \& tài nguyên](#bảng-vm--tài-nguyên)
- [IP / DNS / Domain](#ip--dns--domain)
  - [Port sử dụng trong thiết kế](#port-sử-dụng-trong-thiết-kế)
- [Tổng quan luồng triển khai](#tổng-quan-luồng-triển-khai)
- [Bước 1 — DC01: AD + DNS + NTP + AD CS](#bước-1--dc01-ad--dns--ntp--ad-cs)
  - [Cài AD DS và promote domain](#cài-ad-ds-và-promote-domain)
  - [Cấu hình NTP](#cấu-hình-ntp)
  - [Tạo OU, group, user, service account](#tạo-ou-group-user-service-account)
  - [Cài AD CS Enterprise Root CA](#cài-ad-cs-enterprise-root-ca)
  - [Cho phép SAN và xuất Root CA](#cho-phép-san-và-xuất-root-ca)
- [Bước 2 — Chuẩn bị chung VM Linux](#bước-2--chuẩn-bị-chung-vm-linux)
  - [Time sync (BẮT BUỘC)](#time-sync-bắt-buộc)
  - [Tin AD Root CA (mọi VM Linux)](#tin-ad-root-ca-mọi-vm-linux)
- [Bước 3 — Cấp cert từ AD CS cho dịch vụ Linux](#bước-3--cấp-cert-từ-ad-cs-cho-dịch-vụ-linux)
  - [svc01 — sinh key + CSR (SAN gộp 3 tên)](#svc01--sinh-key--csr-san-gộp-3-tên)
  - [minio1-4 — sinh key + CSR (multi-exec cả 4 node)](#minio1-4--sinh-key--csr-multi-exec-cả-4-node)
  - [Đưa CSR sang DC01](#đưa-csr-sang-dc01)
  - [DC01 — CA ký từng CSR](#dc01--ca-ký-từng-csr)
  - [Cài cert trên Linux](#cài-cert-trên-linux)
- [Bước 4 — svc01: Vault (unseal thủ công)](#bước-4--svc01-vault-unseal-thủ-công)
  - [docker-compose.yml](#docker-composeyml)
  - [Vault: init + unseal thủ công + KV/AppRole cho KES](#vault-init--unseal-thủ-công--kvapprole-cho-kes)
- [Bước 5 — svc01: KES + tạo key riêng từng bucket](#bước-5--svc01-kes--tạo-key-riêng-từng-bucket)
- [Bước 6 — svc01: HAProxy + Prometheus + Grafana](#bước-6--svc01-haproxy--prometheus--grafana)
  - [Prometheus (TLS)](#prometheus-tls)
  - [HAProxy (HTTPS đầu-cuối, verify backend bằng AD CA)](#haproxy-https-đầu-cuối-verify-backend-bằng-ad-ca)
- [Bước 7 — Cụm 4× MinIO AIStor](#bước-7--cụm-4-minio-aistor)
  - [Chuẩn bị ổ data (mỗi node)](#chuẩn-bị-ổ-data-mỗi-node)
  - [Cài gói + license + cert AD CS (mỗi node)](#cài-gói--license--cert-ad-cs-mỗi-node)
  - [Cấu hình chính MinIO — giống hệt cả 4 node](#cấu-hình-chính-minio--giống-hệt-cả-4-node)
  - [Khởi động](#khởi-động)
  - [Kiểm chứng EC:4](#kiểm-chứng-ec4)
- [AD làm IAM: phân quyền riêng từng bucket](#ad-làm-iam-phân-quyền-riêng-từng-bucket)
- [Object Versioning](#object-versioning)
- [Immutable / WORM chống ransomware (Object Lock Compliance)](#immutable--worm-chống-ransomware-object-lock-compliance)
- [Mã hóa riêng từng bucket](#mã-hóa-riêng-từng-bucket)
  - [Kiểm chứng mã hóa thật trên đĩa (console "No" ≠ plaintext)](#kiểm-chứng-mã-hóa-thật-trên-đĩa-console-no--plaintext)
- [Phân quyền riêng từng bucket (group readwrite)](#phân-quyền-riêng-từng-bucket-group-readwrite)
- [Note hiệu năng cho AI workload](#note-hiệu-năng-cho-ai-workload)
- [Client từ svc01: mc, python, mc mirror qua HTTPS](#client-từ-svc01-mc-python-mc-mirror-qua-https)
- [Giám sát Prometheus + Grafana](#giám-sát-prometheus--grafana)
  - [1. Kiểm Prometheus đã scrape được MinIO](#1-kiểm-prometheus-đã-scrape-được-minio)
  - [2. Grafana: thêm datasource Prometheus](#2-grafana-thêm-datasource-prometheus)
  - [3. Import dashboard MinIO](#3-import-dashboard-minio)
- [Kịch bản nghiệm thu](#kịch-bản-nghiệm-thu)
- [Lab → Production](#lab--production)
- [Kết luận](#kết-luận)

## Mục tiêu

- Cluster object storage **HA, EC:4** chịu mất 1 node vẫn đọc/ghi bình thường.
- **Xác thực qua Active Directory** (Windows Server 2025) bằng **LDAPS**, gán policy theo group, phân quyền tới từng bucket.
- **Mã hóa at-rest, khóa riêng từng bucket** (KES + Vault) — mọi object đều mã hóa; bucket nhạy cảm gán khóa riêng để cô lập.
- **Vault** làm keystore cho KES — lab unseal **thủ công**; production cần **auto-unseal** (transit/HSM/cloud KMS) để Vault tự mở sau restart.
- **Versioning + Object Lock Compliance** — chống ransomware/xóa nhầm/sửa đổi trái phép.
- **Toàn bộ kết nối HTTPS/LDAPS**, cert do **AD CS (Win2025)** cấp.
- **Giám sát** Prometheus + Grafana.

## Môi trường

| Thành phần | Thông số |
|-----------|---------|
| Storage node | 4× Ubuntu 24.04, EC:4 (16 ổ) |
| OS dịch vụ | Ubuntu 24.04 (Docker), Windows Server 2025 (AD) |
| Network | VLAN 200 — 10.10.200.0/24, GW 10.10.200.1 |
| MinIO | AIStor Enterprise (trial 60 ngày) |

> **Phạm vi lab:** mọi VM trên cùng 1 host vật lý → host là SPOF, đây là lab mô phỏng cơ chế enterprise chứ chưa phải HA production thật. Phần [Lab → Production](#lab--production) nêu các điểm cần nâng cấp khi triển khai thật.

## Chuẩn bị

- Ổ data cho minio node: lab dùng **HDD** là đủ; production khuyến nghị **NVMe/SSD**.
- License file `minio.license` từ SUBNET (lab dùng bản 60 ngày).
- DNS nội bộ do DC01 (AD) quản lý; mọi VM Linux trỏ DNS về `10.10.200.5`.
- **MTU 9000 (jumbo frame)** cho mạng node-to-node — khuyến nghị production để tối ưu băng thông.

## Kiến trúc tổng thể

```
                         ┌────────────────────────────┐
                         │  Win Server 2025 – DC01    │  AD DS + DNS + NTP + AD CS (Enterprise CA)
                         │  10.10.200.5  (anhlx.lab)  │  ← cấp toàn bộ cert
                         └─────────────┬──────────────┘
                   LDAPS 636 │ DNS 53 │ NTP 123 │ (cấp cert)
        ┌───────────────────────────────┼───────────────────────────────┐
        ▼                               ▼                               ▼
┌──────────────┐   HTTPS 443  ┌───────────────────────────────┐   ┌──────────────┐
│ Browser / mc ├───────────▶ │       svc01 (Ubuntu 24)       │   │  4× MinIO    │
│ python/boto3 │              │        Docker Compose         │   │  AIStor node │
└──────────────┘              │ HAProxy:443  Grafana:3000(TLS)│   │ minio1..4    │
  s3.anhlx.lab (API :9000)    │ Prometheus:9090(TLS)          │   │ .11–.14      │
  console.anhlx.lab(UI :9001) │ KES:7373 ── Vault:8200        │   │ 4×50GB data  │
  → 10.10.200.10              │   (Vault unseal thủ công)     │   │ EC:4 (12+4)  │
                              │                               │   └──────────────┘
                              │  10.10.200.10                 │   erasure set 16 ổ
                              └───────────────────────────────┘
   Subnet 10.10.200.0/24 | GW 10.10.200.1 | egress NAT 443/80/53/123
   *** MỌI kết nối dùng HTTPS/LDAPS, verify bằng AD Root CA, gọi qua FQDN ***
```

## Bảng VM & tài nguyên

| VM | OS | vCPU | RAM | Disk | Vai trò |
|----|----|------|-----|------|---------|
| DC01 | Windows Server 2025 | 4 | 8 GB | 200 GB | AD DS, DNS, NTP, **AD CS (Enterprise Root CA)** |
| svc01 | Ubuntu 24.04 | 8 | 8 GB | 100 GB | Docker: HAProxy, KES, Vault, Prometheus, Grafana |
| minio1–4 | Ubuntu 24.04 | 4 | 8 GB | 1×100 (OS) + **4×50 (data)** | AIStor node |

**EC:** 4 node × 4 ổ = **16 drive → EC:4 (12+4)**, ~600 GB usable (16×50GB raw), chịu mất 1 node (4 ổ).

> **Vì sao AIStor hợp workload AI:** đây là đặc tính của sản phẩm MinIO AIStor — thiết kế cho I/O song song, throughput cao của pipeline AI (đọc dataset, ghi checkpoint/model). Lab này chạy **HDD** để minh họa chức năng là đủ; production dùng **NVMe/SSD** + MTU 9000 mới khai thác hết tốc độ. EC:4 (parity vừa đủ) cho throughput cao hơn EC:6 vốn tốn thêm compute/băng thông ghi.

## IP / DNS / Domain

| Host | FQDN | IP |
|------|------|-----|
| DC01 | `DC01.anhlx.lab` | 10.10.200.5 |
| svc01 / S3 endpoint / Console | `svc01.anhlx.lab` / `s3.anhlx.lab` / `console.anhlx.lab` | 10.10.200.10 |
| minio1–4 | `minio1..4.anhlx.lab` | 10.10.200.11–14 |

- **Domain AD:** `anhlx.lab`. **DNS** cho mọi VM Linux = **10.10.200.5 (DC01)**.
- Tạo bản ghi A trên DNS AD: `s3`, `console`, `svc01`, `minio1..4`.
- **S3 API** vào qua `s3.anhlx.lab:443`, **Console** vào qua `console.anhlx.lab:443` — cùng HAProxy trên svc01, tách nhau bằng hostname (SNI).
- **Bắt buộc gọi nhau bằng FQDN** (không IP) để cert verify đúng hostname.

### Port sử dụng trong thiết kế

| Port | Dịch vụ | Chiều kết nối |
|------|---------|---------------|
| 53 | DNS (DC01) | mọi VM → DC01 |
| 123 | NTP (DC01) | mọi VM → DC01 |
| 636 | LDAPS (DC01) | minio → DC01 |
| 80 / 443 | HAProxy — S3 API (`s3`) + Console (`console`) (svc01) | client → svc01 |
| 7373 | KES (svc01) | minio → svc01 |
| 8200 | Vault (svc01) | admin/KES → svc01 |
| 9000 | MinIO S3 API | HAProxy / Prometheus / inter-node → minio |
| 9001 | MinIO Console | HAProxy (`console.anhlx.lab`) → minio |
| 3000 | Grafana (svc01) | admin → svc01 |
| 9090 | Prometheus (svc01) | admin → svc01 |

> Lab chạy trong **network isolated (VLAN 200)** nên **không bật host firewall (ufw)** trên từng VM cho gọn. DC01 dùng Windows Firewall mặc định (đã cho phép sẵn DNS/NTP/LDAPS/cert). Nếu cần lọc, mở các port ở trên tại firewall mạng/biên.

## Tổng quan luồng triển khai


1. **Bước 1 — DC01:** AD `anhlx.lab` + DNS + NTP + AD CS (dựng CA trước để cấp cert).
2. **Bước 2 — Chuẩn bị Linux:** netplan, chrony, tin AD Root CA cho 4 minio + svc01.
3. **Bước 3 — Cấp cert:** xin cert AD CS cho svc01 và từng minio node.
4. **Bước 4 — svc01/Vault:** init + unseal thủ công, tạo KV + AppRole cho KES.
5. **Bước 5 — svc01/KES:** cấu hình KES, tạo key riêng từng bucket.
6. **Bước 6 — svc01/Observability + LB:** Prometheus, Grafana, HAProxy.
7. **Bước 7 — Cụm MinIO:** cài 4 node, bật cluster EC:4.

> **Thứ tự bật lại sau reboot:** DC01 → Vault (**unseal thủ công**) → KES → MinIO → HAProxy/Prometheus/Grafana. Vault chưa unseal thì KES + S3 mã hóa chưa hoạt động.

## Bước 1 — DC01: AD + DNS + NTP + AD CS

Dựng Domain Controller `DC01` chạy domain `anhlx.lab`, kiêm DNS, NTP và **Enterprise Root CA** cấp cert cho toàn bộ dịch vụ Linux.

Chạy các lệnh dưới đây trong **PowerShell (Run as Administrator)** trên VM Windows Server 2025.

### Cài AD DS và promote domain

```powershell
# Cài role AD DS
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Promote thành forest mới anhlx.lab (DNS cài kèm). Reboot tự động sau khi xong.
Import-Module ADDSDeployment
Install-ADDSForest `
    -DomainName "anhlx.lab" `
    -DomainNetbiosName "ANHLX" `
    -ForestMode "WinThreshold" `
    -DomainMode "WinThreshold" `
    -InstallDns:$true `
    -DatabasePath "C:\Windows\NTDS" `
    -LogPath "C:\Windows\NTDS" `
    -SysvolPath "C:\Windows\SYSVOL" `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "Zxcasd123!@#" -AsPlainText -Force) `
    -Force:$true
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/01.png)

Sau reboot, đăng nhập `ANHLX\Administrator`. Trỏ DNS của DC về chính nó, thêm **forwarder ra 8.8.8.8** để VM trong mạng (trỏ DNS về DC `10.10.200.5`) vẫn phân giải được internet, rồi tạo bản ghi A:

```powershell
# DNS forwarder: query ngoài domain → 8.8.8.8 (để VM nội bộ ra được internet)
Add-DnsServerForwarder -IPAddress 8.8.8.8
# Tạo bản ghi A cho các host Linux
Add-DnsServerResourceRecordA -ZoneName "anhlx.lab" -Name "svc01"   -IPv4Address 10.10.200.10
Add-DnsServerResourceRecordA -ZoneName "anhlx.lab" -Name "s3"      -IPv4Address 10.10.200.10
Add-DnsServerResourceRecordA -ZoneName "anhlx.lab" -Name "console" -IPv4Address 10.10.200.10
Add-DnsServerResourceRecordA -ZoneName "anhlx.lab" -Name "minio1" -IPv4Address 10.10.200.11
Add-DnsServerResourceRecordA -ZoneName "anhlx.lab" -Name "minio2" -IPv4Address 10.10.200.12
Add-DnsServerResourceRecordA -ZoneName "anhlx.lab" -Name "minio3" -IPv4Address 10.10.200.13
Add-DnsServerResourceRecordA -ZoneName "anhlx.lab" -Name "minio4" -IPv4Address 10.10.200.14
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/02.png)

### Cấu hình NTP

DC làm nguồn giờ cho domain, tự sync với NTP ngoài (toàn lab phụ thuộc giờ chuẩn để LDAPS/Kerberos không lệch).

```powershell
w32tm /config /manualpeerlist:"vn.pool.ntp.org,0x8" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
w32tm /resync
w32tm /query /status        # kiểm tra Source = vn.pool.ntp.org
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/03.png)

### Tạo OU, group, user, service account

```powershell
# OU tổng cho toàn bộ lab + 3 OU con bên trong
New-ADOrganizationalUnit -Name "minio"    -Path "DC=anhlx,DC=lab"
New-ADOrganizationalUnit -Name "Accounts" -Path "OU=minio,DC=anhlx,DC=lab"   # KHÔNG đặt tên "Users": đụng container CN=Users mặc định
New-ADOrganizationalUnit -Name "Groups"   -Path "OU=minio,DC=anhlx,DC=lab"
New-ADOrganizationalUnit -Name "Service"  -Path "OU=minio,DC=anhlx,DC=lab"

# Group phân quyền MinIO (global security)
New-ADGroup -Name "minio-admins"    -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=minio,DC=anhlx,DC=lab"
New-ADGroup -Name "minio-readwrite" -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=minio,DC=anhlx,DC=lab"
New-ADGroup -Name "minio-readonly"  -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=minio,DC=anhlx,DC=lab"

# User test + service account để MinIO bind LDAP
$pw = ConvertTo-SecureString "Zxcasd123!@#" -AsPlainText -Force
New-ADUser -Name "alice" -SamAccountName "alice" -UserPrincipalName "alice@anhlx.lab" `
  -Path "OU=Accounts,OU=minio,DC=anhlx,DC=lab" -AccountPassword $pw -Enabled $true -PasswordNeverExpires $true
New-ADUser -Name "bob" -SamAccountName "bob" -UserPrincipalName "bob@anhlx.lab" `
  -Path "OU=Accounts,OU=minio,DC=anhlx,DC=lab" -AccountPassword $pw -Enabled $true -PasswordNeverExpires $true
New-ADUser -Name "svc-minio-ldap" -SamAccountName "svc-minio-ldap" -UserPrincipalName "svc-minio-ldap@anhlx.lab" `
  -Path "OU=Service,OU=minio,DC=anhlx,DC=lab" -AccountPassword $pw -Enabled $true -PasswordNeverExpires $true

# Gán user vào group
Add-ADGroupMember -Identity "minio-admins"   -Members "alice"
Add-ADGroupMember -Identity "minio-readonly" -Members "bob"
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/04.png)

### Cài AD CS Enterprise Root CA

```powershell
Install-WindowsFeature ADCS-Cert-Authority -IncludeManagementTools

Install-AdcsCertificationAuthority `
    -CAType EnterpriseRootCA `
    -CACommonName "ANHLX-CA" `
    -KeyLength 4096 `
    -HashAlgorithmName SHA256 `
    -ValidityPeriod Years -ValidityPeriodUnits 10 `
    -Force
```

> **LDAPS 636 tự bật:** ngay khi có Enterprise CA, DC tự enroll *Domain Controller* cert → LDAPS sống mà không cần cấu hình thêm. Kiểm tra:
> ```powershell
> Test-NetConnection DC01.anhlx.lab -Port 636
> ```

![](/assets/img/2026-07-01-minio-aistor-enterprise/05.png)

### Cho phép SAN và xuất Root CA

```powershell
# Cho CA chấp nhận SAN gửi kèm trong CSR (để cấp cert dịch vụ Linux có đúng FQDN)
certutil -setreg policy\EditFlags +EDITF_ATTRIBUTESUBJECTALTNAME2
net stop certsvc; net start certsvc

# Xuất Root CA cert (DER) thẳng từ cert store cho chắc
Get-ChildItem Cert:\LocalMachine\CA, Cert:\LocalMachine\Root |
  Where-Object { $_.Subject -like "*CN=ANHLX-CA*" } |
  Select-Object -First 1 |
  Export-Certificate -FilePath D:\anhlx-root-ca.cer

# Đổi DER → PEM ngay trên DC01 bằng certutil (khỏi cần openssl), ra file .crt để phân phối cho Linux (Bước 2)
certutil -encode D:\anhlx-root-ca.cer D:\anhlx-root-ca.crt
Get-Content D:\anhlx-root-ca.crt -TotalCount 1        # phải thấy: -----BEGIN CERTIFICATE-----
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/06.png)

> File `D:\anhlx-root-ca.crt` (PEM) là thứ copy sang mọi VM Linux ở [Bước 2](#bước-2--chuẩn-bị-chung-vm-linux). Windows không có `openssl` sẵn nên dùng `certutil -encode` — cùng kết quả PEM.

> **Production:** dùng lại chính Enterprise CA này (hoặc một **Subordinate CA** ký từ nó) để cấp cert cho cụm thật — không phát sinh CA mới.

## Bước 2 — Chuẩn bị chung VM Linux

Áp dụng cho **cả 4 minio node + svc01**. Lúc này CA đã có từ [Bước 1](#bước-1--dc01-ad--dns--ntp--ad-cs) nên Linux mới tin được.
> Trỏ  DNS server về DC01

### Time sync (BẮT BUỘC)

```bash
apt-get install -y chrony
echo "server 10.10.200.5 iburst" | sudo tee -a /etc/chrony/chrony.conf   # thêm DC01 làm nguồn giờ
systemctl restart chrony && chronyc tracking                            # kiểm: Reference ID ~ 10.10.200.5
```

### Tin AD Root CA (mọi VM Linux)

```bash
# Copy file anhlx-root-ca.crt (PEM, đã đổi bằng certutil -encode ở Bước 1) từ DC01 lên VM rồi:
vi /usr/local/share/ca-certificates/anhlx-root-ca.crt
update-ca-certificates

openssl s_client -connect 10.10.200.5:636 -CAfile /etc/ssl/certs/ca-certificates.crt </dev/null 2>&1 | grep -iE "verify (return code|error)|subject="
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/07.png)

## Bước 3 — Cấp cert từ AD CS cho dịch vụ Linux

**Ý tưởng:** mỗi máy (svc01, từng minio node) cần một cert riêng chứng minh đúng hostname của nó, do CA `ANHLX-CA` ký. **Private key (`.key`) sinh ra và giữ luôn trên Linux — không bao giờ gửi đi.** Chỉ gửi **CSR** (`.csr` — bản yêu cầu, KHÔNG chứa key bí mật) sang DC01 cho CA ký, rồi lấy cert (`.crt`) về.

```
Linux: sinh .key + .csr  ──gửi .csr──▶  DC01: CA ký  ──trả .cer──▶  Linux: cài .crt
        (giữ .key lại máy)                                          
```

Tổng cộng 5 cert: 1 cho svc01 + 4 cho minio1..4. Làm 4 minio bằng multi-exec cho nhanh, svc01 làm riêng.

### svc01 — sinh key + CSR (SAN gộp 3 tên)

```bash
mkdir -p ~/lab/certs && cd ~/lab/certs
openssl req -new -newkey rsa:2048 -nodes \
  -keyout svc01.key -out svc01.csr \
  -subj "/CN=svc01.anhlx.lab" \
  -addext "subjectAltName=DNS:svc01.anhlx.lab,DNS:s3.anhlx.lab,DNS:console.anhlx.lab"
```

→ ra 2 file: `svc01.key` (giữ bí mật trên svc01) + `svc01.csr` (đem đi ký). SAN gộp 3 tên vì svc01 vừa là host, vừa phục vụ `s3` (API) và `console` (UI) qua HAProxy.

![](/assets/img/2026-07-01-minio-aistor-enterprise/08.png)

### minio1-4 — sinh key + CSR (multi-exec cả 4 node)

Dùng `hostname -s` để mỗi node **tự điền đúng tên** (minio1..4), khỏi phải sửa tay từng máy:

```bash
mkdir -p ~/lab/certs && cd ~/lab/certs
H=$(hostname -s)                               # minio1 / minio2 / minio3 / minio4
openssl req -new -newkey rsa:2048 -nodes \
  -keyout private.key -out $H.csr \
  -subj "/CN=$H.anhlx.lab" \
  -addext "subjectAltName=DNS:$H.anhlx.lab"
```

→ mỗi node ra `private.key` + `<tên>.csr` (vd trên minio1 là `minio1.csr`).

### Đưa CSR sang DC01

Cách đơn giản, chắc ăn: in nội dung CSR ra rồi **copy-paste** sang DC. Trên từng máy Linux:

```bash
cat ~/lab/certs/svc01.csr        # copy TOÀN BỘ, gồm cả dòng -----BEGIN/END CERTIFICATE REQUEST-----
```

Trên DC01 (PowerShell), tạo thư mục rồi dán vào file **cùng tên**:

```powershell
mkdir D:\csr
notepad D:\csr\svc01.csr         # dán nội dung vừa copy → Save
```

Lặp cho `minio1.csr` … `minio4.csr` (mỗi node `cat` rồi dán sang `D:\csr\minioX.csr`).

> Quen `scp` thì nhanh hơn — DC01 có sẵn OpenSSH client, kéo thẳng về:
> `scp root@svc01.anhlx.lab:lab/certs/svc01.csr C:\csr\` (tương tự cho minio1..4).

### DC01 — CA ký từng CSR

Chạy PowerShell trên DC01. `-config "DC01\ANHLX-CA"` để không hiện popup chọn CA, template `WebServer`:

```powershell
cd D:\csr
certreq -submit -attrib "CertificateTemplate:WebServer" -config "DC01\ANHLX-CA" svc01.csr  svc01.cer
certreq -submit -attrib "CertificateTemplate:WebServer" -config "DC01\ANHLX-CA" minio1.csr minio1.cer
certreq -submit -attrib "CertificateTemplate:WebServer" -config "DC01\ANHLX-CA" minio2.csr minio2.cer
certreq -submit -attrib "CertificateTemplate:WebServer" -config "DC01\ANHLX-CA" minio3.csr minio3.cer
certreq -submit -attrib "CertificateTemplate:WebServer" -config "DC01\ANHLX-CA" minio4.csr minio4.cer
```

→ ra `svc01.cer`, `minio1..4.cer`. **Trả từng file về đúng máy Linux** của nó.

![](/assets/img/2026-07-01-minio-aistor-enterprise/10.png)

### Cài cert trên Linux

certreq thường trả **Base64 (PEM) sẵn**. Mở file kiểm tra: thấy `-----BEGIN CERTIFICATE-----` là PEM, chỉ cần đổi tên; nếu là nhị phân (DER) thì đổi bằng `openssl x509 -inform DER ...` như comment dưới.

**svc01** — đặt `svc01.crt` cạnh `svc01.key` trong `~/lab/certs` (file gộp `s3.pem` cho HAProxy sẽ tạo ở [Bước 4](#bước-4--svc01-vault-unseal-thủ-công)):

```bash
cd ~/lab/certs
cp svc01.cer svc01.crt            # nếu DER: openssl x509 -inform DER -in svc01.cer -out svc01.crt
openssl x509 -in svc01.crt -noout -text | grep -A1 "Subject Alternative Name"   # phải thấy svc01 + s3 + console
```

**mỗi minio node** (multi-exec) — chuẩn hóa thành `public.crt` cạnh `private.key`; việc copy vào thư mục MinIO làm ở [Bước 7](#cài-gói--license--cert-ad-cs-mỗi-node) sau khi cài gói (mới có user `minio-user`):

```bash
cd ~/lab/certs
H=$(hostname -s)
cp $H.cer public.crt             # nếu DER: openssl x509 -inform DER -in $H.cer -out public.crt
openssl x509 -in public.crt -noout -text | grep -A1 "Subject Alternative Name"   # phải thấy đúng minioX
```

> **Chốt lại vị trí cert:**
> - **svc01:** `~/lab/certs/svc01.crt` + `svc01.key` → dùng chung cho HAProxy / KES / Vault / Grafana / Prometheus (SAN đã bao `svc01`+`s3`+`console`).
> - **minio1..4:** `~/lab/certs/public.crt` + `private.key` → Bước 7 copy vào `/home/minio-user/.minio/certs/`, kèm CA tại `.../certs/CAs/anhlx-root-ca.crt`.

![](/assets/img/2026-07-01-minio-aistor-enterprise/11.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/12.png)


## Bước 4 — svc01: Vault (unseal thủ công)

Cài Docker, dựng `docker-compose.yml` cho toàn bộ stack svc01, rồi bật **Vault trước tiên** vì KES + MinIO mã hóa đều phụ thuộc nó.

```bash
apt-get install -y docker.io docker-compose-v2     # docker-compose-v2 (repo Ubuntu) cung cấp lệnh 'docker compose'
systemctl enable --now docker
docker compose version                             # xác nhận Compose v2 đã có

mkdir -p ~/lab/{certs,vault/config,vault/data,kes,prometheus,grafana,haproxy}
cd ~/lab
cp /usr/local/share/ca-certificates/anhlx-root-ca.crt certs/
# svc01.crt + svc01.key (từ Bước 3) phải nằm trong ~/lab/certs — nếu bạn tạo ở chỗ khác thì copy vào trước:
#   cp /certs/svc01.crt /certs/svc01.key ~/lab/certs/
cat certs/svc01.crt certs/svc01.key > certs/s3.pem
chmod 644 certs/svc01.key certs/s3.pem   # container Vault/KES/Grafana/Prometheus chạy non-root cần đọc key — lab; prod dùng uid/gid hoặc secret riêng
```

### docker-compose.yml

File định nghĩa sẵn cả 5 service; các bước sau sẽ `up` lần lượt theo thứ tự phụ thuộc. Paste nguyên khối:

```bash
cat > ~/lab/docker-compose.yml <<'EOF'
services:
  vault:                               # Vault keystore cho KES (unseal thủ công)
    image: hashicorp/vault:latest
    cap_add: ["IPC_LOCK"]
    ports: ["8200:8200"]
    volumes: ["./vault/config:/vault/config","./vault/data:/vault/data","./certs:/certs"]
    command: server

  kes:
    image: quay.io/minio/kes:latest
    depends_on: [vault]
    ports: ["7373:7373"]
    volumes: ["./kes:/etc/kes","./certs:/certs"]
    command: server --auth=off --config=/etc/kes/kes-config.yaml   # --auth=off: KES không verify client cert theo CA (API key là cert tự-ký) — vẫn tính identity để so policy

  prometheus:
    image: prom/prometheus:latest
    ports: ["9090:9090"]
    volumes: ["./prometheus:/etc/prometheus","./certs:/certs"]
    command: ["--config.file=/etc/prometheus/prometheus.yml","--web.config.file=/etc/prometheus/web-config.yml"]

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    volumes: ["./certs:/certs"]
    environment:
      GF_SERVER_PROTOCOL: https
      GF_SERVER_CERT_FILE: /certs/svc01.crt
      GF_SERVER_CERT_KEY: /certs/svc01.key
      GF_SECURITY_ADMIN_PASSWORD: "Zxcasd123!@#"

  haproxy:
    image: haproxy:lts
    ports: ["80:80","443:443"]
    volumes: ["./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro","./certs:/certs:ro"]
EOF
```

### Vault: init + unseal thủ công + KV/AppRole cho KES

Tạo config `~/lab/vault/config/vault.hcl`:

```bash
cat > ~/lab/vault/config/vault.hcl <<'EOF'
disable_mlock = true                     # container không mlock được → tắt; OK với storage "file"
storage "file" { path = "/vault/data" }

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/certs/svc01.crt"
  tls_key_file  = "/certs/svc01.key"
}
api_addr = "https://svc01.anhlx.lab:8200"
ui = true
EOF
```

Cài `vault` CLI trên svc01 (chạy các lệnh `vault ...` từ host, verify TLS qua AD CA):

```bash
apt-get install -y wget gpg
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com noble main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
apt-get update && apt-get install -y vault
systemctl disable --now vault 2>/dev/null || true   # chỉ dùng CLI — tắt vault server host kẻo chiếm cổng 8200 (đụng Vault Docker)
vault version                  # xác nhận CLI đã có
```

```bash
docker compose up -d vault
docker compose exec -u root vault chown -R vault:vault /vault/data   # bind mount do root tạo → cho user 'vault' ghi được
export VAULT_ADDR=https://svc01.anhlx.lab:8200 VAULT_CACERT=~/lab/certs/anhlx-root-ca.crt
vault status                   # Initialized: false, Sealed: true (chưa init — bình thường)
vault operator init            # LƯU 5 unseal key + root token

# Output mẫu (GIỮ BÍ MẬT — KHÔNG commit giá trị thật):
# Unseal Key 1: <unseal-key-1>
# Unseal Key 2: <unseal-key-2>
# Unseal Key 3: <unseal-key-3>
# Unseal Key 4: <unseal-key-4>
# Unseal Key 5: <unseal-key-5>
# Initial Root Token: <root-token>   (dạng hvs.xxxxxxxx)


vault operator unseal          # nhập 3/5 key — LẶP LẠI mỗi lần Vault restart
vault operator unseal          #
vault operator unseal          #
vault login <root-token>
# KV v1 + AppRole cho KES
vault secrets enable -version=1 -path=kv kv
printf 'path "kv/*" { capabilities = ["create","read","delete","list"] }\n' > /tmp/kes.hcl
vault policy write kes-policy /tmp/kes.hcl
vault auth enable approle
vault write auth/approle/role/kes-role token_policies=kes-policy token_ttl=20m token_max_ttl=30m secret_id_num_uses=0
vault read  auth/approle/role/kes-role/role-id        # → VAULT_APPROLE_ID
vault write -f auth/approle/role/kes-role/secret-id   # → VAULT_APPROLE_SECRET
```

> **Lab dùng unseal thủ công:** mỗi lần container `vault` restart, phải `vault operator unseal` lại (nhập 3/5 key) thì KES + S3 mã hóa mới hoạt động trở lại. Đơn giản, đủ cho POC.
> **Production cần auto-unseal:** thêm trust anchor (Vault-transit hoặc **HSM PKCS#11 / cloud KMS**) + cấu hình `seal "transit"` để Vault **tự mở khóa** sau restart, không phụ thuộc thao tác tay; Vault chạy **Raft HA 3 node**. Ngoài phạm vi lab này — xem [Lab → Production](#lab--production).

![](/assets/img/2026-07-01-minio-aistor-enterprise/13.png)

## Bước 5 — svc01: KES + tạo key riêng từng bucket

```bash
docker run --rm quay.io/minio/kes:latest identity new   # → API key kes:v1:... + Identity hash
```

Tạo config `~/lab/kes/kes-config.yaml` (thay 3 placeholder `<IDENTITY_HASH_API_KEY>`, `<VAULT_APPROLE_ID>`, `<VAULT_APPROLE_SECRET>` sau khi paste):

```bash
cat > ~/lab/kes/kes-config.yaml <<'EOF'
address: 0.0.0.0:7373
admin: { identity: disabled }
tls: { key: /certs/svc01.key, cert: /certs/svc01.crt }
policy:
  minio-server:
    allow: ["/v1/key/create/*","/v1/key/generate/*","/v1/key/decrypt/*","/v1/key/bulk/decrypt","/v1/key/list/*","/v1/status","/v1/metrics"]
    identities: ["<IDENTITY_HASH_API_KEY>"]
keystore:
  vault:
    endpoint: https://svc01.anhlx.lab:8200
    engine: kv
    version: v1
    approle: { id: "<VAULT_APPROLE_ID>", secret: "<VAULT_APPROLE_SECRET>" }
    tls: { ca: /certs/anhlx-root-ca.crt }
EOF
```

Thay 3 placeholder bằng **giá trị thật của bạn** (lấy ở các bước trên):

```bash
sed -i \
  -e 's|<IDENTITY_HASH_API_KEY>|<identity hash từ lệnh identity new>|' \
  -e 's|<VAULT_APPROLE_ID>|<role_id từ vault read .../role-id>|' \
  -e 's|<VAULT_APPROLE_SECRET>|<secret_id từ vault write .../secret-id>|' \
  ~/lab/kes/kes-config.yaml

grep -nE "identities|id:|secret:" ~/lab/kes/kes-config.yaml   # kiểm: không còn dấu < >
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/14.png)

```bash
docker compose up -d kes
docker logs lab-kes-1 --tail 20        # không còn lỗi Vault/TLS là OK

# kes CLI trên host: lấy binary ra khỏi image (host đã tin AD CA nên verify TLS được)
docker create --name kestmp quay.io/minio/kes:latest
docker cp kestmp:/kes /usr/local/bin/kes && docker rm kestmp

export KES_SERVER=https://svc01.anhlx.lab:7373 KES_API_KEY="kes:v1:..."   # API key thật ở lệnh 'identity new'

# Khóa cho backend (bắt buộc) + khóa riêng từng bucket:
kes key create minio-backend-default-key
kes key create key-bucketA
kes key create key-bucketB     # ví dụ cho phòng ban/khách hàng khác
kes key ls                     # liệt kê key (bản này dùng 'ls', không phải 'list')
```

Verify khóa đã nằm thật trong Vault (KES ↔ Vault thông suốt):

```bash
export VAULT_ADDR=https://svc01.anhlx.lab:8200 VAULT_CACERT=~/lab/certs/anhlx-root-ca.crt
vault kv list kv     # → minio-backend-default-key, key-bucketA, key-bucketB
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/15.png)

## Bước 6 — svc01: HAProxy + Prometheus + Grafana

### Prometheus (TLS)

Tạo `~/lab/prometheus/web-config.yml`:

```bash
cat > ~/lab/prometheus/web-config.yml <<'EOF'
tls_server_config: { cert_file: /certs/svc01.crt, key_file: /certs/svc01.key }
EOF
```

Tạo `~/lab/prometheus/prometheus.yml`:

```bash
cat > ~/lab/prometheus/prometheus.yml <<'EOF'
global: { scrape_interval: 15s }
scrape_configs:
  - job_name: minio-cluster
    metrics_path: /minio/v2/metrics/cluster
    scheme: https
    tls_config: { ca_file: /certs/anhlx-root-ca.crt }
    static_configs: [{ targets: ['minio1.anhlx.lab:9000','minio2.anhlx.lab:9000','minio3.anhlx.lab:9000','minio4.anhlx.lab:9000'] }]
  - job_name: minio-node
    metrics_path: /minio/v2/metrics/node
    scheme: https
    tls_config: { ca_file: /certs/anhlx-root-ca.crt }
    static_configs: [{ targets: ['minio1.anhlx.lab:9000','minio2.anhlx.lab:9000','minio3.anhlx.lab:9000','minio4.anhlx.lab:9000'] }]
EOF
```

### HAProxy (HTTPS đầu-cuối, verify backend bằng AD CA)

Tạo `~/lab/haproxy/haproxy.cfg`:

```bash
cat > ~/lab/haproxy/haproxy.cfg <<'EOF'
defaults
  mode http
  timeout connect 10s
  timeout client 5m
  timeout server 5m
  timeout tunnel 1h
frontend web
  bind *:80
  bind *:443 ssl crt /certs/s3.pem
  http-request redirect scheme https unless { ssl_fc }
  # Tách theo hostname (SNI): console.anhlx.lab → Console :9001, còn lại → S3 API :9000
  acl is_console hdr(host) -i console.anhlx.lab
  use_backend minio_console if is_console
  default_backend minio_s3
backend minio_s3
  balance roundrobin
  option httpchk
  http-check send meth GET uri /minio/health/live ver HTTP/1.1 hdr Host s3.anhlx.lab   # phải gửi Host: MinIO (đã set MINIO_SERVER_URL) trả 400 nếu thiếu
  http-check expect status 200
  server minio1 minio1.anhlx.lab:9000 check check-sni minio1.anhlx.lab sni str(minio1.anhlx.lab) ssl verify required ca-file /certs/anhlx-root-ca.crt
  server minio2 minio2.anhlx.lab:9000 check check-sni minio2.anhlx.lab sni str(minio2.anhlx.lab) ssl verify required ca-file /certs/anhlx-root-ca.crt
  server minio3 minio3.anhlx.lab:9000 check check-sni minio3.anhlx.lab sni str(minio3.anhlx.lab) ssl verify required ca-file /certs/anhlx-root-ca.crt
  server minio4 minio4.anhlx.lab:9000 check check-sni minio4.anhlx.lab sni str(minio4.anhlx.lab) ssl verify required ca-file /certs/anhlx-root-ca.crt
backend minio_console
  balance roundrobin
  server minio1 minio1.anhlx.lab:9001 check check-sni minio1.anhlx.lab sni str(minio1.anhlx.lab) ssl verify required ca-file /certs/anhlx-root-ca.crt
  server minio2 minio2.anhlx.lab:9001 check check-sni minio2.anhlx.lab sni str(minio2.anhlx.lab) ssl verify required ca-file /certs/anhlx-root-ca.crt
  server minio3 minio3.anhlx.lab:9001 check check-sni minio3.anhlx.lab sni str(minio3.anhlx.lab) ssl verify required ca-file /certs/anhlx-root-ca.crt
  server minio4 minio4.anhlx.lab:9001 check check-sni minio4.anhlx.lab sni str(minio4.anhlx.lab) ssl verify required ca-file /certs/anhlx-root-ca.crt
EOF
```

> Console MinIO dùng WebSocket — `timeout tunnel 1h` ở block `defaults` giữ kết nối không bị ngắt. HAProxy verify backend `:9001` bằng AD Root CA đúng như `:9000`, nên **toàn tuyến browser → HAProxy → node đều HTTPS, cert AD CS**.
> **`sni`/`check-sni` bắt buộc:** mỗi node có cert riêng `CN=minioX`, phải gửi đúng SNI = tên node thì MinIO mới trả đúng cert và HAProxy verify khớp. Thiếu SNI (hoặc SNI lấy nhầm từ `Host s3.anhlx.lab`) → lỗi *"certificate different from the expected one"*. `Host` (HTTP) và `sni` (TLS) tách biệt: Host = `s3.anhlx.lab` để MinIO không trả 400, SNI = `minioX` để khớp cert.

```bash
docker compose up -d prometheus grafana haproxy
```

> HAProxy health-check sẽ báo backend DOWN tới khi cụm MinIO ở [Bước 7](#bước-7--cụm-4-minio-aistor) chạy — bình thường.

![](/assets/img/2026-07-01-minio-aistor-enterprise/16.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/17.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/18.png)

## Bước 7 — Cụm 4× MinIO AIStor

### Chuẩn bị ổ data (mỗi node)

Mỗi node có 4 ổ data riêng (ngoài ổ OS `sda`) — format XFS rồi mount cố định `/mnt/data1..4`.

![](/assets/img/2026-07-01-minio-aistor-enterprise/19.png)

```bash
lsblk                                  # xác định 4 ổ data: thường sdb sdc sdd sde (KHÔNG đụng sda = ổ OS)
for d in b c d e; do sudo mkfs.xfs -f /dev/sd$d; done
sudo mkdir -p /mnt/data{1,2,3,4}

# Tự lấy UUID từng ổ rồi ghi vào /etc/fstab 
i=1
for dev in /dev/sdb /dev/sdc /dev/sdd /dev/sde; do
  uuid=$(blkid -s UUID -o value "$dev")
  grep -q " /mnt/data$i " /etc/fstab || echo "UUID=$uuid /mnt/data$i xfs defaults,noatime 0 2" | sudo tee -a /etc/fstab
  i=$((i+1))
done

sudo mount -a && df -h /mnt/data*      # 4 dòng /mnt/data1..4, mỗi ổ ~50G
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/20.png)

### Cài gói + license + cert AD CS (mỗi node)

```bash
curl -L https://dl.min.io/aistor/minio/release/linux-amd64/minio.deb -o minio.deb && sudo dpkg -i minio.deb
sudo mkdir -p /opt/minio /home/minio-user/.minio/certs/CAs
sudo cp minio.license /opt/minio/minio.license
sudo cp public.crt private.key /home/minio-user/.minio/certs/         # cert AD CS của node
sudo cp anhlx-root-ca.crt /home/minio-user/.minio/certs/CAs/            # tin AD CA (cho KES + LDAPS)
sudo chown -R minio-user:minio-user /opt/minio /home/minio-user/.minio /mnt/data1 /mnt/data2 /mnt/data3 /mnt/data4
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/21.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/22.png)

### Cấu hình chính MinIO — giống hệt cả 4 node

Paste nguyên khối (sau đó sửa 2 placeholder: `MINIO_KMS_KES_API_KEY` và `MINIO_IDENTITY_LDAP_LOOKUP_BIND_PASSWORD`):

```bash
sudo tee /etc/default/minio > /dev/null <<'EOF'
# --- License & topology (HTTPS) ---
MINIO_LICENSE="/opt/minio/minio.license"
MINIO_VOLUMES="https://minio{1...4}.anhlx.lab:9000/mnt/data{1...4}/minio"
MINIO_ROOT_USER="admin"
MINIO_ROOT_PASSWORD="Zxcasd123!@#"
MINIO_OPTS="--console-address :9001"
MINIO_STORAGE_CLASS_STANDARD="EC:4"
MINIO_SERVER_URL="https://s3.anhlx.lab"
MINIO_BROWSER_REDIRECT_URL="https://console.anhlx.lab"
MINIO_PROMETHEUS_AUTH_TYPE="public"

# --- KMS qua KES + Vault ---
MINIO_KMS_KES_ENDPOINT="https://svc01.anhlx.lab:7373"
MINIO_KMS_KES_API_KEY="kes:v1:...."
MINIO_KMS_KES_CAPATH="/home/minio-user/.minio/certs/CAs/anhlx-root-ca.crt"
MINIO_KMS_KES_KEY_NAME="minio-backend-default-key"
# AIStor tự mã hóa MỌI object bằng key này khi có KMS; gán SSE-KMS per-bucket để dùng KHÓA RIÊNG (không có bucket plaintext)

# --- Xác thực AD qua LDAPS (verify đầy đủ, KHÔNG skip) ---
MINIO_IDENTITY_LDAP_SERVER_ADDR="DC01.anhlx.lab:636"
MINIO_IDENTITY_LDAP_LOOKUP_BIND_DN="CN=svc-minio-ldap,OU=Service,OU=minio,DC=anhlx,DC=lab"
MINIO_IDENTITY_LDAP_LOOKUP_BIND_PASSWORD="<mật khẩu svc-minio-ldap>"
MINIO_IDENTITY_LDAP_USER_DN_SEARCH_BASE_DN="OU=Accounts,OU=minio,DC=anhlx,DC=lab"
MINIO_IDENTITY_LDAP_USER_DN_SEARCH_FILTER="(&(objectClass=user)(sAMAccountName=%s))"
MINIO_IDENTITY_LDAP_GROUP_SEARCH_BASE_DN="OU=Groups,OU=minio,DC=anhlx,DC=lab"
MINIO_IDENTITY_LDAP_GROUP_SEARCH_FILTER="(&(objectClass=group)(member=%d))"
EOF
```

```bash
# sudo sed -i \
#   -e 's|^MINIO_KMS_KES_API_KEY=.*|MINIO_KMS_KES_API_KEY="kes:v1:<API-KEY-THẬT>"|' \
#   -e 's|^MINIO_IDENTITY_LDAP_LOOKUP_BIND_PASSWORD=.*|MINIO_IDENTITY_LDAP_LOOKUP_BIND_PASSWORD="Zxcasd123!@#"|' \
#   /etc/default/minio

# grep -E "KES_API_KEY|BIND_PASSWORD" /etc/default/minio     # kiểm 2 dòng đã đúng, không còn '...' / '<...>'
```
Sau khi điền 2 placeholder, kiểm 4 node có **file y hệt nhau** (mọi node phải ra cùng 1 chuỗi hash):

```bash
sha256sum /etc/default/minio        # so sánh output giữa 4 node — phải giống hệt
```

> **EC:4 với 16 ổ:** topology `minio{1...4}` × `data{1...4}` = 1 erasure set 16 ổ (12 data + 4 parity). `MINIO_STORAGE_CLASS_STANDARD="EC:4"` → chịu mất tối đa 4 ổ, tương đương **mất trọn 1 node**.
> LDAPS verify được vì cert DC do **AD CS** cấp và Linux đã tin AD Root CA. Không cần `TLS_SKIP_VERIFY`.

### Khởi động

```bash
sudo systemctl enable --now minio
journalctl -u minio -f
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/23.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/24.png)

Cài **MinIO Client** trên svc01 để quản trị cụm — dùng cho toàn bộ phần sau (lưu ý gói `mc` của Ubuntu là **Midnight Commander**, trùng tên; cài MinIO `mc` vào `/usr/local/bin` để "che" nó):

```bash
curl -fsSL https://dl.min.io/aistor/mc/release/linux-amd64/mc -o /usr/local/bin/mc
chmod +x /usr/local/bin/mc
hash -r                              # xóa cache đường dẫn 'mc' cũ của bash
mc --version                         # PHẢI ra "mc version RELEASE..." (MinIO), không phải "GNU Midnight Commander"
mc alias set adm https://s3.anhlx.lab admin 'Zxcasd123!@#'   # nháy ĐƠN tránh bash nuốt dấu '!'
mc admin info adm                    # 4 servers, 16 drives online, 0 offline
```

### Kiểm chứng EC:4

Xác nhận cụm chạy đúng erasure coding **EC:4 (12 data + 4 parity)** — 3 cách:

```bash
# 1. Config storage class
mc admin config get adm storage_class        # → standard=EC:4

# 2. Object lớn (>1.5MB) chia đúng 16 shard (12+4) trải trên 16 ổ
mc mb adm/ec-test
head -c 5000000 /dev/zero > /tmp/big.bin
mc cp /tmp/big.bin adm/ec-test/big.bin
```

```bash
#mỗi node 4 shard → tổng 4 node = 16 = 12 data + 4 parity 
find /mnt/data*/minio/ec-test/big.bin -name part.1 | wc -l    # mỗi node = 4
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/52.png)

Dọn sau khi kiểm: `mc rb --force adm/ec-test`.

**3. Console — Pool 0 (rõ nhất):** vào Console → **Pool 0**, dòng thông số trên cùng hiện thẳng:

```
Erasure Sets 1 | Stripe Size 16 | Parity 4 | Servers 4 | Drives 16
```

→ **Stripe Size 16 + Parity 4 = EC:4** (16 ổ = 12 data + 4 parity). Kèm Grafana *Health Breakdown*: `Online Drives 16`, `Read Quorum 12` → parity = 16 − 12 = 4 — chịu mất tối đa 4 ổ (trọn 1 node) vẫn đọc/ghi.

![](/assets/img/2026-07-01-minio-aistor-enterprise/54.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/53.png)

## AD làm IAM: phân quyền riêng từng bucket

Làm **ngay sau khi cụm chạy**: gán policy cho group AD thì đăng nhập Console mới được — chưa gán, login sẽ báo *"no policies attached, unable to generate credentials"* (LDAP xác thực đúng nhưng không có quyền). `mc` + alias `adm` đã cài ở [Khởi động](#khởi-động). Gán policy trên svc01:

```bash
# Gán policy built-in theo group AD — đủ để alice/bob đăng nhập Console ngay
mc idp ldap policy attach adm consoleAdmin --group='CN=minio-admins,OU=Groups,OU=minio,DC=anhlx,DC=lab'
mc idp ldap policy attach adm readonly    --group='CN=minio-readonly,OU=Groups,OU=minio,DC=anhlx,DC=lab'

mc idp ldap policy entities adm
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/25.png)

Test: mở browser (máy đã tin AD Root CA) vào Console **`https://console.anhlx.lab`** (443, qua HAProxy → node :9001, cert AD CS) → đăng nhập `alice` (admin) / `bob` (read-only) bằng tài khoản AD. Quyền kế thừa từ group:

- `alice` (minio-admins) → toàn quyền.
- `bob` (minio-readonly) → chỉ đọc.

![](/assets/img/2026-07-01-minio-aistor-enterprise/26.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/27.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/28.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/29.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/30.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/31.png)

## Object Versioning

Versioning là **tiền đề bắt buộc** cho Object Lock/WORM và cũng là lớp phòng vệ đầu tiên: mỗi lần ghi đè/xóa tạo version mới thay vì phá hủy dữ liệu cũ.

```bash
mc mb adm/ai-datasets
mc version enable adm/ai-datasets
mc version info adm/ai-datasets        # → Enabled

# Ghi đè vẫn giữ version cũ
echo v1 > /tmp/f.txt && mc cp /tmp/f.txt adm/ai-datasets/f.txt
echo v2 > /tmp/f.txt && mc cp /tmp/f.txt adm/ai-datasets/f.txt
mc ls --versions adm/ai-datasets/f.txt # → 2 version

# Khôi phục version cũ
mc cp --version-id <VERSION-ID-V1> adm/ai-datasets/f.txt /tmp/restore.txt
```

> Với pipeline AI, versioning bảo vệ dataset/model artifact khỏi ghi đè nhầm trong quá trình train — luôn rollback được về snapshot trước.

![](/assets/img/2026-07-01-minio-aistor-enterprise/32.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/33.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/34.png)

## Immutable / WORM chống ransomware (Object Lock Compliance)

Đây là **trọng tâm POC cho khách tài chính**: Object Lock ở **Compliance mode** khóa object bất biến trong thời hạn retention — **kể cả root cũng KHÔNG xóa/ghi đè được**. Ransomware chiếm được credential admin cũng không thể phá hủy dữ liệu đã lock.

> Object Lock **phải bật ngay khi tạo bucket** (`--with-lock`), không bật được cho bucket đã tồn tại. Bucket có Object Lock luôn kèm versioning.

```bash
# Bucket immutable cho dữ liệu cần bảo toàn (log giao dịch, sao lưu)
mc mb --with-lock adm/worm-vault

# Đặt retention MẶC ĐỊNH: COMPLIANCE 30 ngày cho mọi object ghi vào
mc retention set --default COMPLIANCE 30d adm/worm-vault
mc retention info --default adm/worm-vault     # → COMPLIANCE, 30 days

# Ghi 1 object
echo "transaction-log" > /tmp/tx.log
mc cp /tmp/tx.log adm/worm-vault/

# --- Kiểm chứng tính bất biến ---
mc ls --versions adm/worm-vault/tx.log     # GHI LẠI version-id của object gốc

# (1) 'mc rm' thường KHÔNG phá dữ liệu — chỉ tạo delete marker (ẩn object):
mc rm adm/worm-vault/tx.log
#   → "Created delete marker ..."  — object bị ẩn, NHƯNG version gốc vẫn còn nguyên
mc ls --versions adm/worm-vault/tx.log     # version gốc vẫn nằm đó → khôi phục được

# (2) Xóa THẲNG version đã lock → ĐÂY mới là chỗ WORM chặn:
mc rm --version-id <VERSION-ID-gốc> adm/worm-vault/tx.log
#   → "Object or prefix is WORM-protected, blocking writes or deletes"  (kể cả root cũng KHÔNG destroy được)

# (3) Kịch bản ransomware: cố "nuke" mọi version → delete marker xóa được, version đã lock bị chặn:
mc rm --force --versions adm/worm-vault/tx.log
#   → xóa được delete marker; version dữ liệu: "Object or prefix is WORM-protected..."

# Ghi đè cũng chỉ tạo version MỚI, version gốc bị khóa vẫn nguyên vẹn:
echo "tampered" > /tmp/tx.log && mc cp /tmp/tx.log adm/worm-vault/tx.log
mc retention info adm/worm-vault/tx.log    # → COMPLIANCE, còn hạn tới ...
```

> **Phân biệt quan trọng:** `mc rm` (không `--version-id`) trên bucket versioned chỉ **tạo delete marker** — object bị *ẩn* nhưng dữ liệu thật **không mất**, xóa delete marker là hiện lại. Tính bất biến WORM thể hiện ở chỗ **không thể destroy version đã lock** (bước 2, 3) — kể cả root. Đây mới là điều chống ransomware: kẻ tấn công có thể tạo delete marker làm dữ liệu "biến mất" tạm thời, nhưng **không phá hủy được** bản gốc trong thời hạn retention → luôn khôi phục.

![](/assets/img/2026-07-01-minio-aistor-enterprise/35.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/36.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/37.png)

Ý nghĩa cho khách tài chính:

- **Compliance mode** đáp ứng yêu cầu lưu trữ bất biến (SEC 17a-4, chống chỉnh sửa chứng từ). Không có đường bypass — khác với Governance mode (user có quyền `s3:BypassGovernanceRetention` vẫn override được).
- **Chống ransomware:** dữ liệu đã lock không thể bị mã hóa/xóa bởi kẻ tấn công, kể cả khi chiếm được root credential.
- **Legal Hold** (giữ vô thời hạn, độc lập với retention) cho điều tra/tranh chấp:

```bash
mc legalhold set adm/worm-vault/tx.log
mc legalhold info adm/worm-vault/tx.log
```

## Mã hóa riêng từng bucket

> **Quan trọng (AIStor):** khi đã cấu hình KMS (KES), AIStor **tự mã hóa mọi object at-rest bằng khóa mặc định `minio-backend-default-key`** — **không có bucket nào plaintext**. Việc gán SSE-KMS cho từng bucket là để chỉ định **khóa RIÊNG** cho bucket đó (cô lập theo phòng ban/khách hàng), thay cho khóa mặc định dùng chung.

```bash
mc admin kms key status adm                 # KES/Vault OK

# Tên bucket phải DNS-compliant (chữ THƯỜNG, số, '-') — KHÔNG chữ hoa

# bucket-a: gán KHÓA RIÊNG key-bucketA
mc mb adm/bucket-a
mc encrypt set sse-kms key-bucketA adm/bucket-a

# bucket-b: KHÔNG gán khóa riêng → object tự mã hóa bằng khóa MẶC ĐỊNH minio-backend-default-key
mc mb adm/bucket-b

# bucket-c: khóa riêng khác — phòng ban khác
mc mb adm/bucket-c
mc encrypt set sse-kms key-bucketB adm/bucket-c

# --- Kiểm chứng ---
mc encrypt info adm/bucket-a     # → SSE-KMS, KeyID key-bucketA  (bucket có cấu hình khóa riêng)
mc encrypt info adm/bucket-b     # → "...configuration was not found"  (bucket KHÔNG có SSE config riêng)

# Object thực tế — CẢ HAI đều mã hóa, khác nhau ở KHÓA:
mc cp /tmp/tx.log adm/bucket-a/ && mc stat adm/bucket-a/tx.log | grep -i encrypt   # → SSE-KMS (key-bucketA)  ← khóa RIÊNG
mc cp /tmp/tx.log adm/bucket-b/ && mc stat adm/bucket-b/tx.log | grep -i encrypt   # → SSE-KMS (minio-backend-default-key)  ← khóa MẶC ĐỊNH
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/38.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/39.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/40.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/41.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/42.png)


Ý nghĩa cho khách tài chính:

- **Mọi object đều mã hóa at-rest** (AIStor auto-encrypt khi có KMS) — không có dữ liệu plaintext trên đĩa.
- **Bucket nhạy cảm gán khóa RIÊNG** → cô lập mã hóa theo phòng ban/khách hàng; **xoay/thu hồi 1 khóa không ảnh hưởng bucket khác**. Bucket không gán khóa riêng thì dùng chung `minio-backend-default-key`.
- **Backend hệ thống (IAM/config)** cũng luôn mã hóa bằng `minio-backend-default-key`.
- Xoay khóa 1 bucket: `mc admin kms key create adm key-bucketA-v2` rồi `mc encrypt set sse-kms key-bucketA-v2 adm/bucket-a`.

### Kiểm chứng mã hóa thật trên đĩa (console "No" ≠ plaintext)

Console/`mc encrypt info` báo `bucket-b: No` chỉ nghĩa **bucket không có SSE policy riêng** — KHÔNG phải object để plaintext. Chứng minh bằng cách đọc **file thô trên đĩa**: ghi 1 object toàn chữ `A`, rồi soi shard dữ liệu trên minio node.

```bash
# --- svc01: ghi object toàn 'A' vào bucket-b (không gán khóa riêng) ---
head -c 300000 /dev/zero | tr '\0' 'A' > /tmp/plain.txt
mc cp /tmp/plain.txt adm/bucket-b/plain.txt
```

```bash
# --- 1 minio node: đọc shard trên đĩa (object nhỏ → dữ liệu inline trong xl.meta) ---
f=/mnt/data1/minio/bucket-b/plain.txt/xl.meta
tr -cd 'A' < "$f" | wc -c        # số byte 'A' trong shard
xxd "$f" | tail -10              # xem phần dữ liệu
```

**Kết quả thực tế:** `wc -c` = **110** trên file 26KB (nếu plaintext phải ~25000 — 'A' liên tục). 110 ≈ 26KB/256 = đúng phân bố **ngẫu nhiên của ciphertext**; `xxd` toàn hex random, không có cụm `4141`.

→ **Ghi trên đĩa = ciphertext**, nhưng đọc qua API vẫn ra `A` (`mc cat adm/bucket-b/plain.txt`): MinIO giải mã khi đọc. Đây là bằng chứng **at-rest encryption** chắc chắn nhất — dù console ghi "Encryption: No", dữ liệu vẫn được mã hóa bằng khóa mặc định.

> Muốn thấy `part.1` riêng (thay vì inline): ghi object > ~1.5MB để mỗi shard vượt ngưỡng 128KB, rồi `xxd $(find /mnt/data*/minio/bucket-b/<obj> -name part.1 | head -1)`.

![](/assets/img/2026-07-01-minio-aistor-enterprise/43.png)

## Phân quyền riêng từng bucket (group readwrite)

Đến giờ cụm đã có nhiều bucket (`ai-datasets`, `worm-vault`, `bucket-a`, `bucket-b`...). Tạo policy tùy biến chỉ cho group `minio-readwrite` thao tác **đúng 1 bucket `ai-datasets`** — để chứng minh phân quyền cô lập theo bucket:

```bash
cat > /tmp/rw-ai.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow",
      "Action": ["s3:GetObject","s3:PutObject","s3:ListBucket"],
      "Resource": ["arn:aws:s3:::ai-datasets","arn:aws:s3:::ai-datasets/*"] }
  ]
}
EOF
mc admin policy create adm rw-ai /tmp/rw-ai.json
mc idp ldap policy attach adm rw-ai --group='CN=minio-readwrite,OU=Groups,OU=minio,DC=anhlx,DC=lab'
mc idp ldap policy entities adm
```

![](/assets/img/2026-07-01-minio-aistor-enterprise/44.png)

Test: thêm 1 user AD (vd tạo `carol` rồi `Add-ADGroupMember minio-readwrite carol`) → login Console → **chỉ thấy đúng `ai-datasets`**, ghi/đọc được; **không thấy** `worm-vault`/`bucket-a`/`bucket-b`. Đây là bằng chứng phân quyền riêng từng bucket qua AD group.

![](/assets/img/2026-07-01-minio-aistor-enterprise/45.png)

## Note hiệu năng cho AI workload

MinIO AIStor được thiết kế cho workload AI throughput cao; bản thân lab này chạy **HDD** để POC chức năng là đủ (không benchmark). Khi đưa cluster phục vụ pipeline AI thật (đọc dataset, ghi checkpoint/model), các yếu tố quyết định throughput:

- **Ổ data NVMe/SSD (production)** — yếu tố quan trọng nhất cho hot dataset; lab dùng HDD chỉ để minh họa chức năng.
- **MTU 9000 (jumbo frame)** node-to-node giảm overhead, tăng băng thông erasure stream.
- **EC:4 thay vì EC:6** — ít parity hơn → ghi nhanh hơn, hợp workload đọc-nhiều của AI. Đánh đổi: dung sai lỗi thấp hơn.
- Khi cần đo thật: dùng **`warp`** (S3 benchmark chính thức của MinIO) với object size sát workload (dataset shard / checkpoint).

## Client từ svc01: mc, python, mc mirror qua HTTPS

svc01 đã tin AD Root CA từ [Bước 2](#bước-2--chuẩn-bị-chung-vm-linux) (`update-ca-certificates`) nên mọi client đều verify được cert `s3.anhlx.lab` mà không cần tắt kiểm tra TLS. Endpoint dùng chung: **`https://s3.anhlx.lab`** (443, qua HAProxy).

**mc — CLI chính thức:**

```bash
mc alias set adm https://s3.anhlx.lab admin 'Zxcasd123!@#'   
mc ls adm                       # liệt kê bucket qua HTTPS
mc admin info adm               # trạng thái cụm 4 node
```

**mc mirror — thay cho rsync** (S3 không chạy được `rsync` gốc; `mc mirror` là bản đồng bộ thư mục ↔ bucket tương đương, chạy qua HTTPS):

```bash
# Đẩy 1 thư mục dataset local lên bucket, đồng bộ 2 chiều được với --watch/--remove
mc mirror /data/datasets adm/ai-datasets
mc mirror --overwrite --remove /data/datasets adm/ai-datasets   # đồng bộ khớp hệt nguồn
```

**python (boto3) — gọi S3 API, verify bằng AD Root CA:**

```bash
# Ubuntu 24.04 chặn 'pip install' hệ thống (PEP 668) → cài boto3 qua apt:
apt-get install -y python3-boto3
head -c 1000000 /dev/zero > /tmp/model.bin        # file mẫu để upload
```

```python
import boto3
from botocore.config import Config

s3 = boto3.client(
    "s3",
    endpoint_url="https://s3.anhlx.lab",
    aws_access_key_id="admin",
    aws_secret_access_key="Zxcasd123!@#",
    region_name="us-east-1",
    verify="/usr/local/share/ca-certificates/anhlx-root-ca.crt",   # verify TLS bằng AD CA, KHÔNG verify=False
    config=Config(s3={"addressing_style": "path"}),                # path-style: khỏi cần DNS bucket.s3.anhlx.lab
)
print("Buckets:", [b["Name"] for b in s3.list_buckets()["Buckets"]])
s3.upload_file("/tmp/model.bin", "ai-datasets", "checkpoints/model.bin")
r = s3.head_object(Bucket="ai-datasets", Key="checkpoints/model.bin")
print("Size:", r["ContentLength"], "| SSE:", r.get("ServerSideEncryption"), r.get("SSEKMSKeyId"))
```

> Production: không dùng root `admin` trong script. Tạo **service account** (access key) kế thừa policy từ group AD rồi thay vào `aws_access_key_id/secret`, hoặc lấy STS token qua LDAP.

![](/assets/img/2026-07-01-minio-aistor-enterprise/46.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/47.png)

## Giám sát Prometheus + Grafana

Prometheus + Grafana đã chạy sẵn (Bước 6). MinIO đã bật `MINIO_PROMETHEUS_AUTH_TYPE="public"` nên endpoint `/minio/v2/metrics/*` **không cần token** — Prometheus scrape thẳng qua HTTPS. Giờ chỉ còn nối Grafana ↔ Prometheus và import dashboard.

### 1. Kiểm Prometheus đã scrape được MinIO

Mở `https://svc01.anhlx.lab:9090` → **Status → Target health** (hoặc `/targets`): 2 job `minio-cluster` và `minio-node`, 4 target `minioX.anhlx.lab:9000` phải **UP**.

```bash
# Hoặc kiểm nhanh từ CLI:
curl -s https://svc01.anhlx.lab:9090/api/v1/targets | grep -o '"health":"[a-z]*"' | sort | uniq -c   # → toàn "up"
```

> Target DOWN thường do TLS: Prometheus verify cert node bằng `anhlx-root-ca.crt` (đã khai trong `prometheus.yml`) — cert node do AD CS cấp nên khớp.

### 2. Grafana: thêm datasource Prometheus

Login `https://svc01.anhlx.lab:3000` (`admin` / `Zxcasd123!@#`) → **Connections → Data sources → Add data source → Prometheus**:

| Trường | Giá trị |
|--------|---------|
| **Prometheus server URL** | `https://svc01.anhlx.lab:9090` |
| **Authentication method** | No Authentication |
| **TLS settings** | tick **Add self-signed certificate** → dán nội dung `cat ~/lab/certs/anhlx-root-ca.crt` vào ô **CA Certificate** (KHÔNG tick *Skip TLS certificate validation*) |

→ Kéo xuống cuối bấm **Save & test** → phải báo *"Successfully queried the Prometheus API"*.

> Dùng URL `svc01.anhlx.lab` (không phải `prometheus:9090`) để **khớp CN cert** (svc01). Grafana container vẫn tới được vì `svc01.anhlx.lab` → 10.10.200.10 (chính host). Muốn nhanh có thể tick **Skip TLS Verify** thay cho dán CA — nhưng dán CA mới đúng tinh thần lab.

![](/assets/img/2026-07-01-minio-aistor-enterprise/48.png)

### 3. Import dashboard MinIO

**Dashboards → New → Import** → nhập ID dashboard chính thức của MinIO từ grafana.com → **Load** → chọn datasource Prometheus vừa tạo:

- **25202** — *MinIO Dashboard* (tổng quan cluster: throughput, API, capacity, drive online/offline...)

![](/assets/img/2026-07-01-minio-aistor-enterprise/49.png)

→ Vào dashboard thấy số liệu realtime: dung lượng dùng, số object, request/s, healing, 16 drives online. 

![](/assets/img/2026-07-01-minio-aistor-enterprise/50.png)

![](/assets/img/2026-07-01-minio-aistor-enterprise/51.png)

## Kịch bản nghiệm thu

| # | Hạng mục | Cách kiểm | Kỳ vọng |
|---|----------|-----------|---------|
| 1 | HA EC:4 | poweroff minio4 | Vẫn đọc/ghi; bật lại → tự healing |
| 2 | Giới hạn EC:4 | tắt 2 node | Mất quorum (minh hoạ cần nhiều node ở prod) |
| 3 | Xác thực AD (LDAPS) | login alice/bob | Quyền đúng theo group; kết nối verify cert |
| 3b | Console qua browser HTTPS | mở `https://console.anhlx.lab` login AD | Trang khóa xanh (cert AD CS), login `alice`/`bob` được |
| 3c | Client từ svc01 (HTTPS) | `mc ls adm` + python boto3 `list_buckets` | Trả kết quả, verify AD CA, không tắt TLS |
| 4 | Phân quyền riêng từng bucket | login readwrite | Chỉ thấy/ghi được `ai-datasets` |
| 5 | Versioning | ghi đè + `mc ls --versions` | Giữ nhiều version, rollback được |
| 6 | **Immutable Compliance** | `mc rm --version-id <ver>` version đã lock | **Access Denied** (WORM), kể cả root — version gốc không destroy được |
| 7 | Legal Hold | set legalhold rồi xóa | Không xóa được tới khi gỡ hold |
| 8 | Mã hóa khóa riêng | `mc stat` bucket-a vs bucket-b | a: SSE-KMS **key-bucketA**; b: SSE-KMS **key mặc định** (cả 2 đều mã hóa) |
| 9 | Khóa riêng | `vault kv list kv` | key-bucketA/key-bucketB tách biệt |
| 10 | Unseal thủ công | restart container `vault` rồi `vault operator unseal` | Nhập 3/5 key xong, KES + S3 mã hóa hoạt động lại |
| 11 | Xoay khóa | tạo key mới + set lại bucket | Object mới dùng key mới |
| 12 | Giám sát | Grafana | Số liệu cluster realtime |

## Lab → Production

| Hạng mục | Lab POC | Production |
|----------|---------|-----------|
| Node | 4 VM trên 1 host | **A)** 8+ node **bare-metal** + NVMe JBOD; **B)** nhiều **ESXi** + datastore **SSD/NVMe** → VM minio + VMDK, **anti-affinity** mỗi node 1 host; **C)** **K8s + MinIO Operator** (DirectPV) — chỉ khi cần đa tenant/GitOps, đổi lại hiệu năng thấp hơn |
| EC | EC:4 (16 ổ) | EC:4/EC:6, 16 ổ/set, tách failure domain |
| KES | 1 container | **≥3 KES** HA, tách host |
| Vault | Vault đơn, **unseal thủ công** (file) | **Vault Raft HA 3 node** + **auto-unseal** (transit/HSM/cloud KMS) |
| CA | **AD CS Enterprise Root** | **Cùng AD CS** (hoặc Subordinate CA) – không đổi |
| TLS | HTTPS verify qua AD CA | Như lab, cert autoenrollment/gia hạn tự động |
| Object Lock | Compliance 30d | Compliance theo chính sách lưu trữ (vd 7 năm), kèm Legal Hold quy trình |
| LB | 1 HAProxy | 2 HAProxy + VRRP, 2 switch 100G |
| Audit | file/HTTP | đẩy SIEM (QRadar/Elastic) |

> **Ba hướng triển khai production:**
> - **Bare-metal:** MinIO cài thẳng trên server vật lý + NVMe JBOD — throughput/độ trễ tốt nhất, ít tầng trung gian.
> - **Ảo hóa (≥2 ESXi):** mỗi ESXi host gắn SSD/NVMe local làm datastore, tạo VM minio + virtual disk (VMDK) trên datastore đó, dùng **anti-affinity** để mỗi minio node nằm trên một host vật lý khác → mất 1 host vẫn còn quorum. Vừa giữ được **hạ tầng ảo hóa** (chạy thêm workload khác trên cụm ESXi) vừa có **cụm MinIO AIStor** trên storage SSD/NVMe. Lab này chính là bản thu nhỏ của hướng B — chỉ khác là dồn hết VM trên 1 host nên host là SPOF.
> - **Kubernetes (MinIO Operator):** khai báo tenant bằng CRD, Operator tự dựng StatefulSet + PVC (dùng **DirectPV / local NVMe PV** để bám ổ vật lý), hợp môi trường đã chuẩn hoá K8s, cần multi-tenant self-service + GitOps. **Đánh đổi:** thêm tầng trừu tượng (CNI, CSI, container) → **throughput/độ trễ thấp hơn bare-metal**, vận hành phức tạp hơn (cần cụm K8s khỏe, quản lý vòng đời Operator, CSI cho local disk). **Chỉ dùng khi thật sự cần** tính khai báo/đa tenant trên K8s — ưu tiên hiệu năng thuần thì chọn A hoặc B.

## Kết luận

Cluster MinIO AIStor Enterprise 4 node EC:4 đã chứng minh đủ năng lực enterprise cho khách tài chính: **AD làm IAM** phân quyền riêng từng bucket qua LDAPS, **mã hóa at-rest với khóa riêng từng bucket** qua Vault/KES, **versioning + Object Lock Compliance** chống ransomware ở mức root-proof. Vault ở lab unseal thủ công — bước tiếp theo khi lên production: bổ sung **auto-unseal** (transit/HSM/cloud KMS) cho Vault, tách **KES/Vault HA**, và mở rộng lên 8+ node (bare-metal NVMe hoặc VM trên cụm ESXi có datastore SSD/NVMe) — xem bảng [Lab → Production](#lab--production).

---

