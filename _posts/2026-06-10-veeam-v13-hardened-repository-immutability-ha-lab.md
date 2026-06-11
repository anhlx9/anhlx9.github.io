---
title: "Veeam VBR 13 — Linux Hardened Repository & Immutability"
categories:
- Windows
- Linux
- Veeam
- Backup
tags:
- veeam
- backup
- immutability
- hardened-repository
- disaster-recovery
- ubuntu
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Veeam VBR 13, Linux Hardened Repository XFS, Immutability 7 ngày, backup VMware + Agent, test 3 kịch bản restore
---

Backup là layer cuối cùng của defense — nhưng cũng là mục tiêu số một của kẻ tấn công: chiếm được backup server là xóa được tất cả bản sao và buộc nạn nhân trả tiền chuộc. Lab này mình dựng Veeam VBR 13 với 2 Linux Hardened Repository (XFS + Immutability 7 ngày) trên Ubuntu 24.04, backup primary vào VHR-01 rồi nhân bản ngay sang VHR-02 qua Backup Copy Job Immediate, sau đó test 3 kịch bản thực tế: mất storage, mất VM production, và mất chính Veeam server.

Mình dùng 2 phương pháp — Veeam Agent cài trực tiếp lên DC01 (giả lập máy chủ vật lý), VMware-native backup qua vCenter cho DC02 — cả 2 đều có bản sao trên 2 repo độc lập theo 3-2-1. Linux Hardened Repository hoàn toàn dùng được cho doanh nghiệp — chỉ cần VM thêm disk, không cần hạ tầng phụ; quy mô lớn mới cần nâng lên Ceph RadosGW hoặc MinIO với Object Lock — scale tốt hơn, tách khỏi filesystem và không bypass được kể cả bởi admin.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/00.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/topo.png)

## Mục lục
- [Mục lục](#mục-lục)
  - [1. Topology \& Lab Design](#1-topology--lab-design)
  - [2. Chuẩn bị Linux Hardened Repository](#2-chuẩn-bị-linux-hardened-repository)
  - [3. Cài đặt Veeam VBR 13](#3-cài-đặt-veeam-vbr-13)
  - [4. Add Repository \& Immutability](#4-add-repository--immutability)
  - [5. Add vCenter \& Protection Group](#5-add-vcenter--protection-group)
  - [6. Tạo Backup Jobs](#6-tạo-backup-jobs)
  - [7. Tạo Backup Copy Jobs (Immediate)](#7-tạo-backup-copy-jobs-immediate)
  - [8. Veeam Configuration Backup](#8-veeam-configuration-backup)
  - [9. Test 1 — Storage Failover](#9-test-1--storage-failover)
  - [10. Test 2 — VM Disaster Recovery](#10-test-2--vm-disaster-recovery)
  - [11. Test 3 — Veeam Server Failover](#11-test-3--veeam-server-failover)
  - [Lời kết](#lời-kết)

---

### 1. Topology & Lab Design

Mình dùng 6 VM trên ESXi, tất cả trong VLAN 200 (`10.10.200.0/24`).

| VM | Hostname | IP | OS | Role | vCPU | RAM | Disk |
|----|----------|----|----|------|------|-----|------|
| 1 | dc01 | 10.10.200.11 | Windows Server 2022 | Domain Controller — Agent backup target | 4 | 8 GB | C: 200 GB · E: 500 GB |
| 2 | dc02 | 10.10.200.12 | Windows Server 2022 | Domain Controller — VMware backup target | 4 | 8 GB | C: 200 GB · E: 500 GB |
| 3 | vbr-01 | 10.10.200.21 | Windows Server 2022 | Veeam VBR 13 — main server | 4 | 8 GB | C: 200 GB · E: 500 GB |
| 4 | vbr-02 | 10.10.200.22 | Windows Server 2022 | Veeam VBR 13 — standby | 4 | 8 GB | C: 200 GB · E: 500 GB |
| 5 | vhr-01 | 10.10.200.31 | Ubuntu 24.04 | Linux Hardened Repository 1 | 4 | 4 GB | OS: 100 GB · Data: 500 GB |
| 6 | vhr-02 | 10.10.200.32 | Ubuntu 24.04 | Linux Hardened Repository 2 | 4 | 4 GB | OS: 100 GB · Data: 500 GB |

```
Backup flow:
  DC01 (Agent)   ──[BKP-DC01-Agent]────► VHR-01 ──┐
  DC02 (VMware)  ──[BKP-DC02-VMware]──► VHR-01 ──┼──[COPY-ALL-TO-VHR02]──► VHR-02
  VBR-01 config  ──[Config Backup]────────────────────────────────────────► VHR-02

  Immutability:  VHR-01 = 7 ngày  |  VHR-02 = 7 ngày  (độc lập)
```

3 kịch bản test cuối bài:
1. **Storage Failover** — tắt VHR-01, restore DC02 từ bản copy trên VHR-02
2. **VM Disaster** — xóa DC02 khỏi vCenter, restore full VM từ Veeam
3. **Veeam Server Failover** — tắt VBR-01, cài Veeam lên VBR-02, import config, restore DC02

---

### 2. Chuẩn bị Linux Hardened Repository

Mình thực hiện các bước dưới trên **cả hai** vhr-01 và vhr-02 — thay IP tương ứng khi SSH vào từng máy. Mỗi VM đã có sẵn disk 500 GB gắn vào, chưa format.

**2.1 Format XFS và mount**

Sau khi boot, disk 500 GB cho Data backup xuất hiện là `/dev/sdb`.

```bash
# Verify disk mới
lsblk

# Format XFS — reflink=1 để Veeam dùng fast synthetic full
mkfs.xfs -b size=4096 -m reflink=1 /dev/sdb

# Tạo mount point
mkdir -p /mnt/backup

# Ghi vào fstab 
DISK_UUID=$(blkid -s UUID -o value /dev/sdb)
echo "UUID=${DISK_UUID}  /mnt/backup  xfs  defaults,nofail  0 0" | tee -a /etc/fstab

mount -a
df -h /mnt/backup
```

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/01.png)

**2.2 Tạo user cho Veeam**

Veeam Hardened Repository yêu cầu local user riêng, không dùng root. User này cần sudo trong lần đầu để Veeam deploy transport service — sau khi add repository xong, sudo sẽ được revoke.

```bash
useradd -m -s /bin/bash veeamrepo
passwd veeamrepo
# Nhập: <veeamrepo-password>

# Cấp sudo tạm thời cho lần setup ban đầu
usermod -aG sudo veeamrepo

# Gán ownership thư mục backup
chown veeamrepo:veeamrepo /mnt/backup
chmod 700 /mnt/backup
```

> Sau khi add repository thành công và switch sang single-use credentials, chạy `gpasswd -d veeamrepo sudo` để revoke sudo — đây là bước hoàn thiện chuẩn Hardened Repository.

**2.3 Mở port firewall**

```bash
ufw allow 22/tcp
ufw allow 2500:3300/tcp
ufw --force enable
ufw status
```

Port `2500–3300` là dải Veeam data channel cho backup và restore traffic.

---

### 3. Cài đặt Veeam VBR 13

Mình cài Veeam trên **vbr-01** trước. vbr-02 sẽ cài khi test failover ở Section 11.

**3.1 Mount ISO và cài đặt**

Copy file `VeeamBackup&Replication_13.0.1.1071_20251217.iso` vào vbr-01, mount rồi chạy `Setup.exe`.

Wizard cài đặt:
- **Install** → Veeam Backup & Replication
- Accept EULA
- **License**: Browse đến file `.lic` Trial 30 ngày (tải từ trang chủ Veeam khi đăng ký), hoặc bỏ qua để chạy Community Edition
- **PostgreSQL**: Veeam tự cài PostgreSQL 15 — không cần chuẩn bị trước
- Các mục còn lại giữ mặc định → **Install**
- Chờ khoảng 10–15 phút

> Veeam v13 dùng PostgreSQL thay SQL Server — không cần license SQL riêng, phù hợp cả cho SMB lẫn enterprise.

> **Lưu ý production:** Phiên bản `13.0.1.1071` (Dec 2025) có thể đã có patch mới hơn. Trên môi trường production, nên check Veeam Release Notes và update lên latest patch trước khi vận hành.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/02.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/03.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/04.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/05.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/06.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/07.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/08.png)

**3.2 Mở console**

Sau khi cài xong, mở **Veeam Backup & Replication Console** trên vbr-01. Vào **Main Menu (≡) → License** xác nhận license đang active.

Community Edition cho phép bảo vệ tối đa 10 workload (instances) — đủ cho lab này với 2 target (DC01, DC02). Mỗi VM hoặc máy chủ vật lý được backup tính là 1 instance.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/09.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/10.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/11.png)

---

### 4. Add Repository & Immutability

Mình add cả 2 Linux repo vào Veeam — thực hiện lần lượt VHR-01 rồi VHR-02 với các bước tương tự.

**4.1 Add Linux Hardened Repository**

**Backup Infrastructure** → **Backup Repositories** → **Add Repository** → **Linux** → **Hardened Repository**

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/12.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/13.png)

| Field | Giá trị |
|-------|---------|
| Name | `VHR-01` |
| DNS/IP | `10.10.200.31` |
| Port | 22 |
| Credentials | Chọn **Single-use SSH credentials** → **Add...** → username: `veeamrepo`, password: `<veeamrepo-password>` |

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/14.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/15.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/16.png)

Nhấn **Next** → Veeam SSH vào server, detect OS, cài transport service tự động.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/17.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/18.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/19.png)

Bước **Repository** → nhấn **Populate** → chọn `/mnt/backup` → ghi nhận capacity 500 GB hiển thị đúng.


**4.2 Bật Immutability**

Ở bước **Repository**, tích chọn:

- ✅ **Make recent backups immutable for:** `7` days

> 7 ngày là minimum thực tế. Production thường dùng 14–30 ngày tùy yêu cầu compliance (PCI-DSS, ISO 27001). Dữ liệu trong thời gian immutability không thể xóa kể cả khi có root access trực tiếp lên server Linux.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/20.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/21.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/22.png)
![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/23.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/24.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/25.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/26.png)

Sau khi add VHR-01 thành công, SSH vào vhr-01 và revoke sudo:

```bash
gpasswd -d veeamrepo sudo
```

Lặp lại các bước 4.1–4.2 cho VHR-02 với IP `10.10.200.32`, rồi revoke sudo trên vhr-02 tương tự. Kết quả mình có 2 Hardened Repository trong Backup Infrastructure:

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/27.png)

---

### 5. Add vCenter & Protection Group

**5.1 Add vCenter vào Veeam**

**Backup Infrastructure** → **Managed Servers** → **Add Server** → **VMware vSphere** → **vCenter Server**

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/28.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/29.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/30.png)

| Field | Giá trị |
|-------|---------|
| DNS/IP | `<vcenter-ip-hoac-fqdn>` |
| Credentials | Add → tài khoản vCenter admin |

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/31.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/32.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/33.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/34.png)

Sau khi add xong, vào **Inventory** → **Virtual Infrastructure** — sẽ thấy toàn bộ VM tree, bao gồm dc02.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/35.png)

**5.2 Tạo Protection Group cho DC01 (Agent)**

Protection Group là cơ chế Veeam dùng để quản lý các endpoint cần cài Agent — Veeam tự push Agent và theo dõi trạng thái.

**Inventory** → **Physical Infrastructure** → **Add** → **Protection Group**

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/36.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/37.png)

| Field | Giá trị |
|-------|---------|
| Name | `PG-Physical-Servers` |
| Type | **Individual computers** |
| Computer | `10.10.200.11` (dc01) |
| Credentials | Add → Windows → Local Administrator hoặc Domain Admin trên dc01 |

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/38.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/39.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/40.png)

Sau khi tạo, Veeam rescan group → kết nối dc01 qua WMI, push Veeam Agent for Windows tự động. Mình confirm dc01 xuất hiện với status **Managed** trong inventory.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/41.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/42.png)

---

### 6. Tạo Backup Jobs

**6.1 Job 1 — Agent Backup (DC01)**

Mô phỏng backup máy chủ vật lý hoặc VM không có quyền truy cập hypervisor.

**Home** → **Backup Job** → **Windows computer...**

| Bước | Cấu hình |
|------|---------|
| Name | `BKP-DC01-Agent` |
| Job Mode | **Managed by backup server** |
| Computers | Add → chọn từ Protection Group `PG-Physical-Servers` → `dc01` |
| Backup mode | **Entire computer** |
| Storage → Repository | `VHR-01` |
| Storage → Retention policy | `7` days |
| Guest Processing | ✅ Enable application-aware processing · ☐ Guest file system indexing |
| Schedule | Daily, 22:00 |

> **Entire computer** tạo bare-metal restore point — có thể restore cả OS lẫn data kể cả khi thay phần cứng hoàn toàn. Với máy chủ vật lý production, đây là mode chuẩn.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/4.3png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/44.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/45.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/46.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/47.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/48.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/49.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/50.png)

Chuột phải job → **Start** → verify **Success** và file backup xuất hiện trên VHR-01.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/51.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/52.png)

**6.2 Job 2 — VMware Backup (DC02)**

Mô phỏng backup máy ảo production thông qua vCenter — Veeam dùng VMware snapshot API, không cần agent bên trong VM.

**Home** → **Backup Job** → **Virtual machine...**

| Bước | Cấu hình |
|------|---------|
| Name | `BKP-DC02-VMware` |
| Virtual Machines | Add VM → chọn `dc02` |
| Storage → Repository | `VHR-01` |
| Storage → Retention policy | `7` days |
| Guest Processing | ✅ Enable application-aware processing · ☐ Guest file system indexing |
| Schedule | Daily, 22:00 |

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/53.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/54.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/55.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/56.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/57.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/58.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/59.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/60.png)

Sau khi tạo xong, chuột phải job → **Start** để chạy thử lần đầu. Verify status **Success** và có restore point trong **Home → Backups → Disk**.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/61.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/62.png)

---

### 7. Tạo Backup Copy Jobs (Immediate)

Backup Copy Job sao chép dữ liệu từ VHR-01 sang VHR-02 ngay khi có restore point mới — đảm bảo VHR-02 luôn có bản sao độc lập mà không cần chờ schedule riêng.

Trong Veeam v13, một Backup Copy Job có thể gom nhiều source job vào cùng một wizard — mình chỉ cần tạo **1 copy job** cho cả 2 backup.

**Home** → **Backup Copy** → **Image-level backup...**

| Field | Giá trị |
|-------|---------|
| Name | `COPY-ALL-TO-VHR02` |
| Copy mode | ✅ **Immediate copy (mirroring)** |
| Objects | Add → chọn `BKP-DC01-Agent` + `BKP-DC02-VMware` |
| Target Repository | `VHR-02` |
| Retention policy | `7` days |
| GFS (bắt buộc) | ✅ **Keep certain full backups** → `1 weekly` |

> **Lưu ý:** Backup Copy Job vào Hardened Repository (immutable) **bắt buộc bật GFS retention** — Veeam yêu cầu điều này để đảm bảo có ít nhất một full backup immutable trong chuỗi. Bỏ check GFS sẽ báo lỗi.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/63.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/64.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/65.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/66.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/67.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/68.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/69.png)

Sau khi 2 primary backup jobs chạy xong, 2 copy jobs tự kích hoạt. Mình verify trong **Home → Backups → Disk (Copy)** thấy cả dc01 lẫn dc02 có restore point trên VHR-02.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/70.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/71.png)

---

### 8. Veeam Configuration Backup

Config Backup lưu toàn bộ cấu hình VBR — jobs, credentials, repository connections, schedules — ra file `.bco`. Đây là thứ cần thiết để phục hồi Veeam server khi vbr-01 bị sự cố.

**Main Menu (≡) → Configuration Backup**

| Field | Giá trị |
|-------|---------|
| Enable configuration backup | ✅ |
| Backup repository | `VHR-02` |
| Keep configuration backup | `5` restore points |
| Encrypt configuration backup | ✅ → passphrase: `<config-backup-passphrase>` |
| Schedule | Daily, 23:30 |

Mình chọn VHR-02 thay vì VHR-01 vì kịch bản failover quan trọng nhất là cả vbr-01 lẫn VHR-01 cùng không đến được — lúc đó chỉ cần VHR-02 là đủ để rebuild toàn bộ.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/72.png)

Nhấn **Backup Now** để tạo bản config backup đầu tiên ngay lập tức. Mình SSH vào vhr-02 verify:

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/73.png)

---

### 9. Test 1 — Storage Failover

**Kịch bản:** VHR-01 bị sự cố. Yêu cầu: restore vẫn thực hiện được từ bản sao trên VHR-02.

**Bước 1 — Tắt VHR-01:**

Trong vSphere Client → vhr-01 → **Power Off**.

**Bước 2 — Chạy backup job:**

Chuột phải `BKP-DC02-VMware` → **Start**. Job báo lỗi không kết nối được VHR-01 — đây là expected behavior. Điểm quan trọng là bản sao trên VHR-02 từ lần chạy trước vẫn nguyên vẹn và đang trong trạng thái immutable.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/74.png)

**Bước 3 — Restore DC02 từ VHR-02:**

**Home** → **Backups** → **Disk (Copy)** → tìm `dc02` → chuột phải → **Restore entire VM...**

Veeam đọc trực tiếp từ VHR-02. Chọn restore point mới nhất → **Finish**.

Verify dc02 boot lại bình thường, đăng nhập được vào domain.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/75.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/76.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/76.png)

---

### 10. Test 2 — VM Disaster Recovery

**Kịch bản:** DC02 bị xóa khỏi vCenter (nhầm tay hoặc attacker). Yêu cầu: restore full VM từ Veeam.

Mình bật lại vhr-01 trước khi bắt đầu test này để dùng primary backup.

**Bước 1 — Xóa DC02:**

Trong vSphere Client → dc02 → chuột phải → **Delete from Disk** (xóa cả VM files trên datastore).

**Bước 2 — Restore full VM:**

**Home** → **Backups** → **Disk** → `dc02` → chuột phải → **Restore entire VM...**

| Bước wizard | Cấu hình |
|-------------|---------|
| Restore Point | Chọn bản mới nhất |
| Restore Mode | **Restore to original location** |
| Reason | `Accidental deletion test` |

Nhấn **Finish** → Veeam kết nối vCenter, tạo lại VM, restore VMDK từ backup, register VM vào inventory.

**Bước 3 — Verify AD:**

```powershell
# Verify replication với DC01
repadmin /replsummary

# Verify domain controller hoạt động
dcdiag /test:replications /v
```

DC01 và DC02 sync lại, domain hoạt động bình thường.

---

### 11. Test 3 — Veeam Server Failover

**Kịch bản:** VBR-01 bị sự cố toàn bộ. Yêu cầu: phục hồi Veeam trên VBR-02 từ config backup, sau đó restore DC02.

**Bước 1 — Tắt VBR-01:**

Trong vSphere Client → vbr-01 → **Power Off**.

**Bước 2 — Copy file config backup:**

SSH vào vhr-02, copy file `.bco` mới nhất ra một Windows network share hoặc trực tiếp sang vbr-02:

```bash
# Trên vhr-02: xem file bco mới nhất
ls -lt /mnt/backup/VBRConfig/
```

Copy file `.bco` sang `E:\` trên vbr-02 (qua shared folder hoặc SCP).

**Bước 3 — Cài Veeam trên VBR-02:**

Mount ISO `VeeamBackup&Replication_13.0.1.1071_20251217.iso` trên vbr-02, chạy `Setup.exe` với cấu hình mặc định — không cần add license hay cấu hình gì thêm, mình sẽ import config từ file `.bco`.

Cài đặt mất khoảng 10–15 phút.

**Bước 4 — Restore Veeam Configuration:**

Mở Veeam Console trên vbr-02 → **Main Menu (≡)** → **Configuration Restore...**

| Bước wizard | Cấu hình |
|-------------|---------|
| Restore from | **Local path** → trỏ đến file `.bco` tại `E:\` |
| Encryption passphrase | `<config-backup-passphrase>` |
| Restore options | ✅ **Restore jobs in disabled state** |

Chọn **Restore jobs in disabled state** để review trước khi cho phép jobs tự chạy — tránh trường hợp job ghi đè data trên primary repo khi chưa kiểm tra xong.

Nhấn **Restore** → Veeam import toàn bộ: repositories, credentials, jobs, schedules.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/77.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/78.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/79.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/80.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/81.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/82.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/83.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/84.png)


**Bước 5 — Verify kết nối repository:**

**Backup Infrastructure** → **Backup Repositories** → chuột phải **VHR-02** → **Rescan**. Repository kết nối thành công, Veeam đọc được danh sách backup files và restore points.

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/84.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/85.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/86.png)

![](/assets/img/2026-06-10-veeam-v13-hardened-repository-immutability-ha-lab/87.png)


**Bước 6 — Restore DC02 từ VBR-02:**

**Home** → **Backups** → **Disk (Copy)** → `dc02` → **Restore entire VM...**

Thực hiện tương tự Test 2. Verify DC02 boot lại, AD sync với DC01.

> Từ khi VBR-01 down đến khi restore DC02 thành công trong lab mất khoảng **20–25 phút** — 10–15 phút cài Veeam mới, 5 phút import config, còn lại là thời gian restore VM. RTO này hoàn toàn chấp nhận được cho phần lớn doanh nghiệp không phải critical 24/7.

---

### Lời kết

Lab này mình đi qua đủ vòng đời của một hạ tầng backup nghiêm túc — từ dựng Linux Hardened Repository với XFS Immutability, backup song song 2 phương pháp VMware và Agent, đến test cả 3 kịch bản thất bại: mất storage, mất VM, mất chính Veeam server. Điểm quan trọng nhất là Backup Copy Job (Immediate) kết hợp Config Backup lưu về repo độc lập VHR-02 tạo ra 2 lớp bảo vệ tách biệt hoàn toàn — kể cả khi site chính down, recovery vẫn thực hiện được từ một điểm duy nhất còn lại. Bước tiếp theo nếu muốn nâng thêm: thêm Capacity Tier đẩy cold backup lên S3-compatible object storage (MinIO) và bật Object Lock để có immutability layer thứ ba theo chuẩn 3-2-1-1-0.
