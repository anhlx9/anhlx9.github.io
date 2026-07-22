---
title: "Đồng bộ, Đối soát & Di trú dữ liệu: hệ hỗn hợp Database Oracle–MariaDB–PostgreSQL–MSSQL với Data Guard, GoldenGate & Veridata"
categories:
- OracleDB
- MariaDB
- PostgreSQL
- MSSQL
- Database
tags:
- data-guard
- goldengate
- veridata
- mssql
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Oracle lõi + 3 DB vệ tinh (MariaDB, PostgreSQL, MSSQL): đồng bộ Data Guard & GoldenGate, đối soát Veridata, di trú toàn hệ sang bộ VM mới
---

Một hệ tài chính lõi Oracle Database. Đội ứng dụng mới viết trên PostgreSQL, hệ đối tác chỉ nhận SQL Server, mấy dịch vụ nội bộ thì quen MariaDB — mỗi hệ DB có mặt vì mục đích khác nhau và phù hợp với từng loại nhu cầu khác nhau, ép tất cả về một thứ chỉ đổi lấy rắc rối khác. Nhưng cũng chẳng bên nào nên cắm thẳng vào lõi: một câu query lỡ tay bên báo cáo là cả luồng giao dịch chậm theo.

Lối ra quen thuộc là mô hình lõi–vệ tinh (Hub/Spoke). Oracle giữ nguồn dữ liệu duy nhất, dữ liệu chảy ra ba DB vệ tinh — MariaDB 10.6, PostgreSQL 13, SQL Server 2022 — mỗi đội đọc trên bản sao của mình. Dựng xong mô hình mới là nửa đầu câu chuyện. Nửa sau là ba việc đeo theo nó suốt vòng đời: dữ liệu chảy sang bằng đường nào và lõi chết thì lấy gì chạy tiếp, chảy rồi làm sao biết bản sao còn khớp từng row với bản gốc, tới ngày thay phần cứng thì bê cả cụm sang máy mới kiểu gì mà không phải downtime hệ thống.

Lab dựng trọn mạch đó trên 6 VM — **đồng bộ**, **đối soát**, **di trú**:

1. **Phần 1 — Dựng nền tảng:** cài Oracle 19c, MariaDB, PostgreSQL, SQL Server; tạo schema tài chính `FINACC` ở Oracle, schema đích rỗng ở 3 vệ tinh.
2. **Phần 2 — Đồng bộ:** Data Guard `OPRI`→`OSTBA` dựng bản sao vật lý cho lõi; GoldenGate Hub đẩy `FINACC` ra cả 3 vệ tinh bằng CDC.
3. **Phần 3 — Đối soát:** Veridata so từng row cặp Oracle↔MSSQL; rồi ghi 1 record vào primary và soi nó lan tới standby lẫn 3 vệ tinh.
4. **Phần 4 — Di trú:** dựng bộ VM mới (.31/.32 Oracle, .33 MSSQL), switchover lõi trong vài giây, kéo 3 vệ tinh theo, bỏ bộ cũ.

| Thành phần | Vai trò trong lab |
|---|---|
| **Data Guard** | Ship/apply redo giữa các node Oracle — dự phòng ở phần 2, đường di trú lõi ở phần 4 |
| **GoldenGate** | CDC heterogeneous: đọc redo Oracle, apply sang 3 họ DB khác nhau |
| **Veridata** | So dữ liệu ở mức row, chỉ dùng cho cặp nó hỗ trợ là Oracle↔MSSQL |
| **MariaDB / PostgreSQL / MSSQL** | 3 vệ tinh, mỗi cái một họ DB để lộ rõ khác biệt lúc map kiểu dữ liệu |

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/architecture.png"/>

## Mục lục

- [Phần 1 — Dựng nền tảng 4 DB](#phần-1--dựng-nền-tảng-4-db)
    - [1.1 — Chuẩn bị (.21/.22 Oracle Linux, .23 Windows)](#11--chuẩn-bị-2122-oracle-linux-23-windows)
    - [1.2 — MS SQL Server Developer (Windows, .23)](#12--ms-sql-server-developer-windows-23)
    - [1.3 — MariaDB 10.6 (Docker, .21)](#13--mariadb-106-docker-21)
    - [1.4 — PostgreSQL 13 (Docker, .22)](#14--postgresql-13-docker-22)
    - [1.5 — Cài Oracle 19c software-only (.21/.22)](#15--cài-oracle-19c-software-only-2122)
    - [1.6 — Tạo Primary OPRI + schema FINACC (.21)](#16--tạo-primary-opri--schema-finacc-21)
    - [1.7 — Listener + tnsnames (.21/.22)](#17--listener--tnsnames-2122)
- [Phần 2 — Đồng bộ dữ liệu](#phần-2--đồng-bộ-dữ-liệu)
    - [2.1 — Đồng bộ dự phòng: Oracle Data Guard OPRI→OSTBA](#21--đồng-bộ-dự-phòng-oracle-data-guard-opriostba)
    - [2.2 — Đồng bộ phục vụ khai thác: GoldenGate Hub → 3 vệ tinh](#22--đồng-bộ-phục-vụ-khai-thác-goldengate-hub--3-vệ-tinh)
      - [2.2.1 — OGG Hub + Extract (capture Oracle)](#221--ogg-hub--extract-capture-oracle)
      - [2.2.2 — Replicat → MariaDB](#222--replicat--mariadb)
      - [2.2.3 — Replicat → PostgreSQL](#223--replicat--postgresql)
      - [2.2.4 — Replicat → MS SQL Server (qua ODBC từ hub)](#224--replicat--ms-sql-server-qua-odbc-từ-hub)
      - [2.2.5 — Tổng kết deployment GoldenGate](#225--tổng-kết-deployment-goldengate)
- [Phần 3 — Đối soát dữ liệu](#phần-3--đối-soát-dữ-liệu)
    - [3.1 — Cài Veridata (.22) + 2 agent (Oracle, SQL Server)](#31--cài-veridata-22--2-agent-oracle-sql-server)
    - [3.2 — Veridata verify Oracle ↔ MSSQL](#32--veridata-verify-oracle--mssql)
    - [3.3 — Kiểm chứng end-to-end (ghi 1 record → soi cả hệ)](#33--kiểm-chứng-end-to-end-ghi-1-record--soi-cả-hệ)
- [Phần 4 — Di trú dữ liệu](#phần-4--di-trú-dữ-liệu)
    - [4.1 — Dựng bộ VM mới (.31/.32 Oracle, .33 MSSQL)](#41--dựng-bộ-vm-mới-3132-oracle-33-mssql)
    - [4.2 — Di trú Oracle: Data Guard mở rộng + switchover sang .31](#42--di-trú-oracle-data-guard-mở-rộng--switchover-sang-31)
    - [4.3 — Di trú vệ tinh: MariaDB→.31, PostgreSQL→.32, MSSQL→.33](#43--di-trú-vệ-tinh-mariadb31-postgresql32-mssql33)
    - [4.4 — Chuyển Hub + Veridata sang bộ mới, kích hoạt](#44--chuyển-hub--veridata-sang-bộ-mới-kích-hoạt)
      - [A. Cài 4 binary OGG + Service Manager + 4 deployment trên .31](#a-cài-4-binary-ogg--service-manager--4-deployment-trên-31)
      - [B. Extract `EXT_FIN` capture NPRI (Dep\_Hub 9010)](#b-extract-ext_fin-capture-npri-dep_hub-9010)
      - [C. 3 Distribution Path trên Dep\_Hub](#c-3-distribution-path-trên-dep_hub)
      - [D. Dọn checkpoint table cũ trên 3 vệ tinh mới](#d-dọn-checkpoint-table-cũ-trên-3-vệ-tinh-mới)
      - [E. 3 Replicat trỏ vệ tinh mới](#e-3-replicat-trỏ-vệ-tinh-mới)
      - [F. Mở ghi lại (downtime kết thúc) + test xuyên hệ mới](#f-mở-ghi-lại-downtime-kết-thúc--test-xuyên-hệ-mới)
      - [G. Veridata mới trên .32](#g-veridata-mới-trên-32)
    - [4.5 — Bỏ .21/.22/.23](#45--bỏ-212223)
  - [Kết luận](#kết-luận)

## Môi trường

**Bộ VM chạy (Phần 1–3):**

| VM | Hostname | IP | OS | Spec (vCPU / RAM / Disk) | Vai trò |
|----|----------|-----|-----|----|---------|
| VM1 | Oracle-pri-01 | 10.10.200.21 | Oracle Linux 8.10 | 8 / 12 GB / 200 GB | Oracle **Primary OPRI** + **MariaDB** (docker) + **OGG Hub** (Extract + 3 Replicat) |
| VM2 | Oracle-sby-01 | 10.10.200.22 | Oracle Linux 8.10 | 8 / 12 GB / 200 GB | Oracle **Standby OSTBA** (Data Guard) + **PostgreSQL** (docker) + **Veridata** |
| VM3 | MSSQL-01 | 10.10.200.23 | **Windows Server 2025** | 8 / 8 GB / 200 GB | **MS SQL Server 2022 Developer** |

**Bộ VM đích (Phần 4 — di trú):**

| VM | Hostname | IP | OS | Spec (vCPU / RAM / Disk) | Vai trò sau cutover |
|----|----------|-----|-----|----|---------------------|
| VM4 | Oracle-pri-02 | 10.10.200.31 | Oracle Linux 8.10 | 8 / 12 GB / 200 GB | Oracle **Primary** (mới) + MariaDB + OGG Hub |
| VM5 | Oracle-sby-02 | 10.10.200.32 | Oracle Linux 8.10 | 8 / 12 GB / 200 GB | Oracle **Standby** (mới) + PostgreSQL + Veridata |
| VM6 | MSSQL-02 | 10.10.200.33 | Windows Server 2025 | 8 / 8 GB / 200 GB | MS SQL Server (mới) |


Stack phần mềm:

| Thành phần | Version | Vai trò trong lab |
|-----------|---------|-------------------|
| Oracle Database | 19c (19.3) EE | DB lõi — nguồn dữ liệu `FINACC` |
| MariaDB | 10.6 (Docker) | Vệ tinh |
| PostgreSQL | 13 (Docker) | Vệ tinh |
| MS SQL Server | 2022 **Developer** | Vệ tinh — Express KHÔNG được OGG hỗ trợ |
| GoldenGate | 26ai (23.26) | CDC Oracle → 3 vệ tinh (build for Oracle / MySQL / PostgreSQL / SQL Server) |
| Veridata | 12.2.1.4 | Đối soát Oracle↔MSSQL (nền FMW Infrastructure 12.2.1.4 + JDK 8u491) |

> **Vì sao đối soát tự động chỉ cho Oracle↔MSSQL?** Veridata 12.2.1.4 hỗ trợ các datasource: Oracle, **SQL Server**, DB2, Sybase, Informix, Teradata, NSK, Hive — **KHÔNG có MySQL/MariaDB lẫn PostgreSQL**. Vì vậy MariaDB/PostgreSQL đối soát bằng SQL thủ công, còn MSSQL đi qua Veridata (SQL Server dùng tên 3 phần `db.schema.table` khớp cách Veridata sinh SQL). Đây phản ánh đúng thực tế: Veridata thường dùng cho Oracle↔Oracle và Oracle↔SQL Server.

## Chuẩn bị

Tải trước các gói (Oracle cần tài khoản OTN/edelivery miễn phí; license OTN dùng cho lab/nghiên cứu):

| Gói | Đặt ở VM | Link |
|-----|----------|------|
| Oracle Linux 8.10 ISO | .21/.22/.31/.32 | https://yum.oracle.com/oracle-linux-isos.html |
| Windows Server 2025 ISO (eval/lab) | .23/.33 | https://www.microsoft.com/evalcenter/evaluate-windows-server-2025 |
| Oracle Database 19c (19.3) EE — `LINUX.X64_193000_db_home.zip` — lấy link **"Home"**, KHÔNG phải RPM | Oracle VM | https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html |
| GoldenGate 26ai (23.26) — build **for Oracle / for MySQL / for PostgreSQL / for SQL Server** (Linux x86-64) | .21 (hub) | https://www.oracle.com/middleware/technologies/goldengate-downloads.html |
| **Microsoft ODBC Driver 18 for SQL Server** (Linux) — `msodbcsql18` (≥ 17.8.1), cài qua repo | .21 (hub) | https://packages.microsoft.com/config/rhel/8/prod.repo |
| **SQL Server 2022 Developer** (free non-prod — **KHÔNG Express**, dính `OGG-05311`) + SSMS | .23/.33 | https://www.microsoft.com/sql-server/sql-server-downloads |
| **FMW Infrastructure 12.2.1.4** + **Veridata 12.2.1.4** — hệ 12.2.1.4, KHÔNG trộn 14.1.2 | .22/.32 | https://edelivery.oracle.com |
| Oracle JDK 8 — `jdk-8u491-linux-x64.rpm` | .22/.32 | https://www.oracle.com/java/technologies/downloads/#java8 |
| MariaDB 10.6 / PostgreSQL 13 — Docker image `mariadb:10.6` / `postgres:13` (tự pull) | .21/.22 | — |

---

# Phần 1 — Dựng nền tảng 4 DB

Kết thúc phần này: Oracle Primary có schema `FINACC` (2 bảng `accounts`, `transactions`), và 3 DB vệ tinh (MariaDB/PostgreSQL/MSSQL) đã dựng với schema đích **rỗng** chờ GoldenGate đổ dữ liệu.

### 1.1 — Chuẩn bị (.21/.22 Oracle Linux, .23 Windows)

**Mục tiêu:** hostname, hosts, NTP, tắt firewall/SELinux (lab), cài gói preinstall của Oracle + Docker trên 2 node Linux; Windows .23 đặt IP tĩnh.

Chạy trên **.21 và .22** (đổi hostname tương ứng `Oracle-pri-01` / `Oracle-sby-01`):

```bash
# Đổi theo từng VM
hostnamectl set-hostname Oracle-pri-01

cat >> /etc/hosts << 'EOF'
10.10.200.21 Oracle-pri-01
10.10.200.22 Oracle-sby-01
10.10.200.23 MSSQL-01
10.10.200.31 Oracle-pri-02
10.10.200.32 Oracle-sby-02
10.10.200.33 MSSQL-02
EOF

# NTP
dnf install -y chrony
sed -i 's/^pool .*/pool vn.pool.ntp.org iburst/' /etc/chrony.conf
systemctl enable --now chronyd

# Lab: tắt firewalld + SELinux permissive
systemctl disable --now firewalld
sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
setenforce 0

# Gói preinstall 19c: tự tạo user oracle, group, kernel params, limits
dnf install -y oracle-database-preinstall-19c wget unzip tar nmap-ncat
echo 'oracle:Zxcasd123!@#' | chpasswd

# oracle sudo không cần password — các bước sau đổi qua lại oracle/root liên tục (Lab only)
echo 'oracle ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/oracle
chmod 440 /etc/sudoers.d/oracle
visudo -c

# Cây thư mục Oracle
mkdir -p /u01/app/oracle/product/19.3.0/dbhome_1 /u01/oradata /u01/fra /u01/soft
chown -R oracle:oinstall /u01
chmod -R 775 /u01

# Docker engine — chạy DB vệ tinh MariaDB/PostgreSQL dạng container
dnf install -y dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
systemctl enable --now docker
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/01.png"/>

Trên **.23 (Windows Server 2025)**: đặt IP tĩnh `10.10.200.23/24` gw `.1`, DNS `8.8.8.8`; đồng bộ giờ (`w32tm /config /manualpeerlist:vn.pool.ntp.org`).

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/02.png"/>

**Phân phối bộ cài về đúng VM.** Toàn bộ gói đã tải sẵn về `/home/anhlx` trên **.21**; từ đây đẩy vào `/u01/soft` của .21 và .22 — mỗi node chỉ nhận phần nó cần: **.21** làm hub OGG (Oracle DB + 4 build GoldenGate), **.22** chạy Veridata (Oracle DB + FMW Infrastructure + Veridata + JDK 8). Bằng **root** trên **.21**:

```bash
cd /home/anhlx

# .21 — hub: Oracle DB 19c + 4 build GoldenGate (Oracle/MySQL/PostgreSQL/SQLServer)
cp *db_home.zip *for_Oracle*.zip *for_MySQL*.zip \
   *for_PostgreSQL*.zip *for_SQLServer*.zip /u01/soft/

# .22 — standby + Veridata: Oracle DB 19c + FMW Infrastructure + Veridata + JDK 8
scp *db_home.zip *Infrastructure*.zip *Veridata*.zip \
    *jdk-8u491*.rpm root@10.10.200.22:/u01/soft/

# runInstaller chạy bằng user oracle → trả quyền
chown -R oracle:oinstall /u01/soft
ssh root@10.10.200.22 'chown -R oracle:oinstall /u01/soft'
```

**Kết quả mong đợi:**

```bash
ls -l /u01/soft/                                   # trên .21
ssh root@10.10.200.22 'ls -l /u01/soft/'           # trên .22
```

```
# .21
Oracle_Database_19c_for_Linux-LINUX.X64_193000_db_home.zip
Oracle_GoldenGate_23.26.1.0.0_on_Linux_x86-64_for_Oracle_for_Linux_x86-64-V1054774-01.zip
Oracle_GoldenGate_23.26.1.0.1_on_Linux_x86-64_for_MySQL-compatible_Databases_for_Linux_x86-64-V1054821-01.zip
Oracle_GoldenGate_23.26.1.0.1_on_Linux_x86-64_for_SQLServer_for_Linux_x86-64-V1054820-01.zip
Oracle_GoldenGate_23.26.1.0.2_on_Linux_x86-64_for_PostgreSQL_for_Linux_x86-64-V1054822-01.zip

# .22
Oracle_Database_19c_for_Linux-LINUX.X64_193000_db_home.zip
Oracle_Fusion_Middleware_12c_Infrastructure_12.2.1.4.0-V983368-01.zip
Oracle_GoldenGate_Veridata_12.2.1.4.0-V983619-01.zip
Oracle_JDK_8-jdk-8u491-linux-x64.rpm
```

> Bản GoldenGate **for MySQL-compatible Databases** dùng cho MariaDB — không có build riêng tên MariaDB. Các glob `*for_MySQL*`, `*for_SQLServer*`... khớp đúng 1 file mỗi loại nên dùng lại được ở mọi bước unzip phía sau.

---

### 1.2 — MS SQL Server Developer (Windows, .23)

**Mục tiêu:** dựng MS SQL Server 2022 trên Windows .23 làm DB vệ tinh, cấu hình để hub OGG (.21) và Veridata (.22) kết nối từ xa qua TCP/IP port 1433, mixed-mode auth, login `ggmssql`, database `FINACC` + 2 bảng đích rỗng.

> **Ghi chú: phải dùng Developer/Standard/Enterprise, KHÔNG dùng Express.** Oracle GoldenGate for SQL Server **không hỗ trợ Express Edition** (EngineEdition 4) — khi Replicat/Veridata login sẽ báo `OGG-05311 Oracle GoldenGate does not support SQL Server 2022 Engine 4 edition`. **Developer Edition** miễn phí (non-prod), engine = Enterprise (EngineEdition 3) → OGG chấp nhận. Dùng Developer.

**Bước 1 — Cài SQL Server 2022 Developer.** Tải **SQL Server 2022 Developer** (miễn phí) từ microsoft.com/sql-server/sql-server-downloads → chạy installer:
- **New SQL Server standalone installation** → Instance: đặt **default instance `MSSQLSERVER`** (nghe port 1433, khỏi named instance động).
- **Database Engine Configuration** → Authentication Mode: **Mixed Mode**, đặt mật khẩu `sa` = `Zxcasd123!@#`, **Add Current User** vào SQL admins.
- Cài thêm **SSMS** (SQL Server Management Studio) để chạy SQL.

> Nếu lỡ cài Express: **Edition Upgrade** giữ nguyên data — Installation Center → Maintenance → Edition Upgrade → chọn Developer. Kiểm: `SELECT SERVERPROPERTY('EngineEdition');` phải = **3** (không phải 4).

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/03.png"/>

**Bước 2 — Bật TCP/IP + cố định port 1433** (SQL Server Configuration Manager):
- SQL Server Network Configuration → Protocols for MSSQLSERVER → **TCP/IP** → **Enable**.
- Double-click TCP/IP → tab **IP Addresses** → **IPAll**: xóa trắng *TCP Dynamic Ports*, đặt **TCP Port = 1433**.
- SQL Server Services → **SQL Server (MSSQLSERVER)** → **Restart**.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/04.png"/>

**Bước 3 — Mở firewall** (PowerShell admin):
```powershell
New-NetFirewallRule -DisplayName "SQL Server 1433" -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/05.png"/>

**Bước 4 — Tạo database + login + 2 bảng đích rỗng** (SSMS → New Query → Execute):
```sql
CREATE DATABASE FINACC;
GO
USE FINACC;
GO
CREATE TABLE dbo.accounts (
  id       BIGINT       PRIMARY KEY,
  acct_no  VARCHAR(20)  UNIQUE,
  owner    VARCHAR(80),
  balance  DECIMAL(18,2)
);
CREATE TABLE dbo.transactions (
  id        BIGINT       PRIMARY KEY,
  acct_id   BIGINT,
  amount    DECIMAL(18,2),
  txn_type  VARCHAR(10),
  created   DATETIME
);
GO
CREATE LOGIN ggmssql WITH PASSWORD = 'Zxcasd123!@#', CHECK_POLICY = OFF;
GO
USE FINACC;
CREATE USER ggmssql FOR LOGIN ggmssql;
ALTER ROLE db_owner ADD MEMBER ggmssql;
GO
```

**Bước 5 — Query kiểm tra instance + FINACC** (SSMS → New Query → Execute):
```sql
-- 1) Thông tin instance hiện tại
USE master;
GO
SELECT
  @@SERVERNAME                                AS server_name,
  SERVERPROPERTY('ProductVersion')            AS product_version,
  SERVERPROPERTY('Edition')                   AS edition,
  SERVERPROPERTY('EngineEdition')             AS engine_edition,          -- phải = 3
  SERVERPROPERTY('IsIntegratedSecurityOnly')  AS windows_auth_only;       -- phải = 0 (Mixed Mode)
GO

-- 2) Thuộc tính database FINACC
SELECT name, database_id, recovery_model_desc, state_desc, is_cdc_enabled
FROM sys.databases
WHERE name = 'FINACC';
GO

-- 3) Bảng đích + số row hiện có
USE FINACC;
GO
SELECT s.name AS [schema], t.name AS [table], p.rows AS row_count
FROM sys.tables t
JOIN sys.schemas s    ON s.schema_id = t.schema_id
JOIN sys.partitions p ON p.object_id = t.object_id AND p.index_id IN (0,1)
ORDER BY t.name;
GO

-- 4) User ggmssql + role trong FINACC
SELECT dp.name AS db_user, dp.type_desc, r.name AS role_name
FROM sys.database_principals dp
LEFT JOIN sys.database_role_members drm ON drm.member_principal_id = dp.principal_id
LEFT JOIN sys.database_principals r     ON r.principal_id = drm.role_principal_id
WHERE dp.name = 'ggmssql';
GO
```

`engine_edition` = 3 (không phải 4 = Express), `windows_auth_only` = 0 (SQL login `ggmssql` login được), 2 bảng đích rỗng chờ Replicat apply. Từ .21: `nc -vz 10.10.200.23 1433` → `succeeded`.

> **Bảng SQL Server dùng tên 3 phần** `FINACC.dbo.accounts` (database.schema.table) — chính là lý do Veridata so được MSSQL (Veridata sinh SQL 3 phần khớp SQL Server, khác MySQL chỉ 2 phần `db.table`).

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/06.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/07.png"/>

---

### 1.3 — MariaDB 10.6 (Docker, .21)

**Mục tiêu:** dựng nhanh MariaDB dạng container nhận dữ liệu Oracle qua GoldenGate; bật binlog ROW để Phần 4 (di trú) replicate tiếp sang .31. DB vệ tinh lab nên chỉ cần container, không cài native. Bằng **root** trên **.21**:

```bash
mkdir -p /opt/mariadb/conf /opt/mariadb/data
cat > /opt/mariadb/conf/lab.cnf << 'EOF'
[mysqld]
server_id                = 21
log_bin                  = /var/lib/mysql/mariadb-bin
binlog_format            = ROW
expire_logs_days         = 7
innodb_buffer_pool_size  = 512M
EOF

cat > /opt/mariadb/docker-compose.yml << 'EOF'
services:
  mariadb:
    image: mariadb:10.6
    container_name: mariadb
    restart: unless-stopped
    ports: ["3306:3306"]
    environment:
      MARIADB_ROOT_PASSWORD: 'Zxcasd123!@#'
    volumes:
      - ./conf/lab.cnf:/etc/mysql/conf.d/lab.cnf:ro
      - ./data:/var/lib/mysql
EOF

docker compose -f /opt/mariadb/docker-compose.yml up -d
```

Tạo schema đích **rỗng** khớp cấu trúc Oracle + user cho OGG apply + user replication (phần di trú). Vì OGG/Replicat kết nối qua TCP nên user để `'%'` (không phải `localhost`):

```bash
docker exec -i mariadb mariadb -uroot -p'Zxcasd123!@#' << 'EOF'
CREATE DATABASE finacc;
CREATE TABLE finacc.accounts (
  id BIGINT PRIMARY KEY, acct_no VARCHAR(20) UNIQUE,
  owner VARCHAR(80), balance DECIMAL(18,2));
CREATE TABLE finacc.transactions (
  id BIGINT PRIMARY KEY, acct_id BIGINT,
  amount DECIMAL(18,2), txn_type VARCHAR(10), created DATETIME);

CREATE USER 'ggmysql'@'%' IDENTIFIED BY 'Zxcasd123!@#';
GRANT ALL PRIVILEGES ON finacc.* TO 'ggmysql'@'%';
GRANT SELECT ON *.* TO 'ggmysql'@'%';
CREATE USER 'repl'@'10.10.200.%' IDENTIFIED BY 'Zxcasd123!@#';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.10.200.%';
FLUSH PRIVILEGES;

SHOW TABLES FROM finacc;
SELECT * FROM finacc.accounts;
SELECT * FROM finacc.transactions;
EOF
```

Kiểm binlog đã bật (nguồn cho replication sang .31 ở Phần 4):

```bash
docker exec mariadb mariadb -uroot -p'Zxcasd123!@#' -e "SHOW MASTER STATUS;"
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/08.png"/>

### 1.4 — PostgreSQL 13 (Docker, .22)

**Mục tiêu:** dựng PostgreSQL container nhận dữ liệu Oracle qua GoldenGate; bật `wal_level=logical` mở đường cho cả GoldenGate lẫn streaming replication native sang .32 ở Phần 4. Bằng **root** trên **.22**:

```bash
mkdir -p /opt/pg/data
cat > /opt/pg/docker-compose.yml << 'EOF'
services:
  postgres:
    image: postgres:13
    container_name: postgres
    restart: unless-stopped
    ports: ["5432:5432"]
    environment:
      POSTGRES_PASSWORD: 'Zxcasd123!@#'
    volumes:
      - ./data:/var/lib/postgresql/data
    command:
      - "-c"
      - "listen_addresses=*"
      - "-c"
      - "wal_level=logical"
      - "-c"
      - "max_wal_senders=10"
      - "-c"
      - "max_replication_slots=10"
      - "-c"
      - "wal_keep_size=1GB"
      - "-c"
      - "shared_buffers=512MB"
      - "-c"
      - "password_encryption=scram-sha-256"
EOF

docker compose -f /opt/pg/docker-compose.yml up -d

# Mở pg_hba cho OGG apply + replication rồi reload
docker exec -i postgres bash -c "cat >> /var/lib/postgresql/data/pg_hba.conf" << 'EOF'
host    all             ggpg        10.10.200.0/24        scram-sha-256
host    replication     repl        10.10.200.0/24        scram-sha-256
EOF
docker exec postgres psql -U postgres -c "SELECT pg_reload_conf();"
```

> ⚠ **`password_encryption=scram-sha-256` là bắt buộc, không phải tùy chọn.** PostgreSQL 13 mặc định băm mật khẩu bằng **md5** (từ PG14 mới đổi sang scram), trong khi 2 dòng `pg_hba` vừa thêm đều đòi **scram-sha-256** — role tạo ra sẽ không có SCRAM verifier để bắt tay, server trả `password authentication failed` dù mật khẩu gõ đúng. Nếu đã lỡ tạo role trước khi bật tham số này, xem cách vá ở cuối mục.

Tạo schema đích rỗng + user cho OGG apply + user replication:

```bash
docker exec -i postgres psql -U postgres << 'EOF'
CREATE ROLE ggpg LOGIN PASSWORD 'Zxcasd123!@#';
CREATE ROLE repl REPLICATION LOGIN PASSWORD 'Zxcasd123!@#';
CREATE DATABASE finacc OWNER ggpg;
\c finacc
CREATE TABLE accounts (id BIGINT PRIMARY KEY, acct_no VARCHAR(20) UNIQUE,
  owner VARCHAR(80), balance NUMERIC(18,2));
CREATE TABLE transactions (id BIGINT PRIMARY KEY, acct_id BIGINT,
  amount NUMERIC(18,2), txn_type VARCHAR(10), created TIMESTAMP);
GRANT ALL ON ALL TABLES IN SCHEMA public TO ggpg;

\dt
SELECT * FROM accounts;
SELECT * FROM transactions;
EOF
```

**Kết quả mong đợi:** 2 bảng đã tạo, dữ liệu **rỗng** (chờ Replicat apply):

```
        List of relations
 Schema |     Name     | Type  |  Owner
--------+--------------+-------+----------
 public | accounts     | table | postgres
 public | transactions | table | postgres
(2 rows)

 id | acct_no | owner | balance
----+---------+-------+---------
(0 rows)

 id | acct_id | amount | txn_type | created
----+---------+--------+----------+---------
(0 rows)
```

Kiểm `wal_level` (nguồn cho cả GoldenGate lẫn streaming replication sang .32 ở Phần 4) và kiểu băm mật khẩu của 2 role vừa tạo:

```bash
docker exec postgres psql -U postgres -c "SHOW wal_level;"
docker exec postgres psql -U postgres -Atc \
  "SELECT rolname, left(rolpassword,14) FROM pg_authid WHERE rolname IN ('ggpg','repl');"
```

```
 wal_level
-----------
 logical

ggpg|SCRAM-SHA-256$
repl|SCRAM-SHA-256$
```

Ra `md5...` thay vì `SCRAM-SHA-256$` nghĩa là container đang chạy không có `password_encryption=scram-sha-256` — băm lại, không cần restart vì verifier được đọc trực tiếp lúc auth:

```bash
docker exec -i postgres psql -U postgres << 'EOF'
SET password_encryption = 'scram-sha-256';
ALTER ROLE ggpg PASSWORD 'Zxcasd123!@#';
ALTER ROLE repl PASSWORD 'Zxcasd123!@#';
EOF
```

> Cái bẫy ở chỗ lỗi này **không lộ ra ngay**. `docker-entrypoint` tự chèn dòng catch-all `host all all all md5` vào `pg_hba.conf` lúc khởi tạo, nằm **trên** 2 dòng ta append — mà pg_hba xét từ trên xuống nên `ggpg` khớp md5 và chạy ngon suốt Phần 2. Riêng kết nối **replication** thì catch-all `host all all` không match (record replication chỉ khớp dòng có chữ `replication` ở cột database), nên `pg_basebackup` ở [4.3](#43--di-trú-vệ-tinh-mariadb31-postgresql32-mssql33) là thứ đầu tiên thực sự đi vào dòng scram — và vỡ ở tận đó.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/09.png"/>

---

### 1.5 — Cài Oracle 19c software-only (.21/.22)

**Mục tiêu:** giải nén binary 19c và chạy installer silent (chưa tạo DB). Bằng user **oracle** trên **.21 và .22**.

> ⚠ **`ORACLE_SID` phải khác nhau giữa 2 node** — `.21` là `OPRI`, `.22` là `OSTBA`.

Trên **.21**:

```bash
su - oracle

cat >> ~/.bash_profile << 'EOF'
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.3.0/dbhome_1
export ORACLE_SID=OPRI
export PATH=$ORACLE_HOME/bin:$PATH
EOF
source ~/.bash_profile
```

Trên **.22** — y hệt, chỉ đổi `ORACLE_SID`:

```bash
su - oracle

cat >> ~/.bash_profile << 'EOF'
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.3.0/dbhome_1
export ORACLE_SID=OSTBA
export PATH=$ORACLE_HOME/bin:$PATH
EOF
source ~/.bash_profile
```

Xác nhận trước khi đi tiếp (chạy trên từng node, phải ra đúng SID của node đó):

```bash
echo $ORACLE_SID          # .21 -> OPRI ; .22 -> OSTBA
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/10.png"/>

Phần còn lại giống nhau trên **cả 2 node**:

```bash
unzip -q /u01/soft/*db_home.zip -d $ORACLE_HOME

# OL8 chưa có trong danh sách distro của installer 19.3 -> ép nhận diện
export CV_ASSUME_DISTID=OEL7.8

cd $ORACLE_HOME
./runInstaller -ignorePrereq -waitforcompletion -silent \
  oracle.install.option=INSTALL_DB_SWONLY \
  ORACLE_HOSTNAME=$(hostname) \
  UNIX_GROUP_NAME=oinstall \
  INVENTORY_LOCATION=/u01/app/oraInventory \
  ORACLE_HOME=$ORACLE_HOME ORACLE_BASE=$ORACLE_BASE \
  oracle.install.db.InstallEdition=EE \
  oracle.install.db.OSDBA_GROUP=dba oracle.install.db.OSOPER_GROUP=oper \
  oracle.install.db.OSBACKUPDBA_GROUP=backupdba oracle.install.db.OSDGDBA_GROUP=dgdba \
  oracle.install.db.OSKMDBA_GROUP=kmdba oracle.install.db.OSRACDBA_GROUP=racdba \
  SECURITY_UPDATES_VIA_MYORACLESUPPORT=false DECLINE_SECURITY_UPDATES=true
```

Sau `Successfully Setup Software`, chạy 2 script bằng **root**:

```bash
sudo /u01/app/oraInventory/orainstRoot.sh
sudo /u01/app/oracle/product/19.3.0/dbhome_1/root.sh
```

**Kết quả mong đợi:** `sqlplus -v` → `SQL*Plus: Release 19.0.0.0.0`; `echo $ORACLE_SID` ra `OPRI` trên .21 và `OSTBA` trên .22.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/11.png"/>

### 1.6 — Tạo Primary OPRI + schema FINACC (.21)

**Mục tiêu:** tạo DB `OPRI` bằng `dbca` silent, bật archivelog + force logging + standby redo log (điều kiện bắt buộc của Data Guard). SGA bóp ~1.5 GB để 12 GB RAM còn chỗ cho MariaDB + GoldenGate. User **oracle** trên **.21**:

```bash
dbca -silent -createDatabase \
  -templateName General_Purpose.dbc \
  -gdbname OPRI -sid OPRI -createAsContainerDatabase false \
  -sysPassword 'Zxcasd123!@#' -systemPassword 'Zxcasd123!@#' \
  -storageType FS -datafileDestination /u01/oradata \
  -recoveryAreaDestination /u01/fra -recoveryAreaSize 20480 \
  -characterSet AL32UTF8 -totalMemory 1536 -emConfiguration NONE
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/12.png"/>

Bật archivelog, force logging, thêm standby redo log (n+1 = 4 group):

```sql
sqlplus / as sysdba

SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
ALTER DATABASE FORCE LOGGING;
ALTER SYSTEM SWITCH LOGFILE;

ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 11 ('/u01/oradata/OPRI/srl11.log') SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 12 ('/u01/oradata/OPRI/srl12.log') SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 13 ('/u01/oradata/OPRI/srl13.log') SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 GROUP 14 ('/u01/oradata/OPRI/srl14.log') SIZE 200M;

ALTER SYSTEM SET log_archive_config='DG_CONFIG=(OPRI,OSTBA)' SCOPE=BOTH;
ALTER SYSTEM SET standby_file_management=AUTO SCOPE=BOTH;
EXIT;
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/13.png"/>

Tạo **schema tài chính `FINACC`** — nguồn dữ liệu chảy xuyên suốt lab:

```sql
sqlplus / as sysdba

CREATE USER finacc IDENTIFIED BY "Zxcasd123!@#" QUOTA UNLIMITED ON users;
GRANT connect, resource TO finacc;

CREATE TABLE finacc.accounts (
  id       NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  acct_no  VARCHAR2(20) UNIQUE,
  owner    VARCHAR2(80),
  balance  NUMBER(18,2) DEFAULT 0
);
CREATE TABLE finacc.transactions (
  id        NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  acct_id   NUMBER REFERENCES finacc.accounts(id),
  amount    NUMBER(18,2),
  txn_type  VARCHAR2(10),          -- CREDIT / DEBIT
  created   DATE DEFAULT SYSDATE
);

INSERT INTO finacc.accounts (acct_no, owner, balance) VALUES ('ACC-0001', 'Nguyen Van A', 1000000);
INSERT INTO finacc.accounts (acct_no, owner, balance) VALUES ('ACC-0002', 'Tran Thi B', 500000);
INSERT INTO finacc.transactions (acct_id, amount, txn_type) VALUES (1, 250000, 'CREDIT');
COMMIT;
EXIT;
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/14.png"/>

Verify **cấu hình DG-ready + schema FINACC** (user **oracle** trên **.21**):

```sql
sqlplus / as sysdba

-- Chế độ DB + điều kiện Data Guard
SELECT name, open_mode, database_role FROM v$database;   -- OPRI / READ WRITE / PRIMARY
SELECT log_mode, force_logging FROM v$database;          -- ARCHIVELOG / YES
SELECT COUNT(*) AS srl FROM v$standby_log;               -- 4
SHOW PARAMETER log_archive_config                        -- DG_CONFIG=(OPRI,OSTBA)
SHOW PARAMETER standby_file_management                   -- AUTO

-- Schema nguồn + dữ liệu seed
SELECT username, account_status FROM dba_users WHERE username='FINACC';   -- FINACC / OPEN
SELECT table_name FROM dba_tables WHERE owner='FINACC' ORDER BY 1;        -- ACCOUNTS / TRANSACTIONS
SELECT 'accounts' AS tbl, COUNT(*) FROM finacc.accounts
UNION ALL
SELECT 'transactions', COUNT(*) FROM finacc.transactions;                 -- 2 / 1

-- Xem toàn bộ record vừa insert
SET LINESIZE 200
COLUMN owner FORMAT A20
SELECT * FROM finacc.accounts ORDER BY id;
SELECT * FROM finacc.transactions ORDER BY id;
EXIT;
```

**Kết quả mong đợi:** DB `PRIMARY` / `READ WRITE`; `log_mode = ARCHIVELOG`, `force_logging = YES`, `v$standby_log = 4`; schema `FINACC` mở, có 2 bảng với `accounts = 2`, `transactions = 1`.

> Nếu `force_logging = NO` → chạy lại `ALTER DATABASE FORCE LOGGING;`. Nếu `srl < 4` → thêm standby redo log cho đủ 4 group. Hai điều kiện này **bắt buộc** trước khi Data Guard duplicate ở Phần 2.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/15.png"/>

### 1.7 — Listener + tnsnames (.21/.22)

**Mục tiêu:** listener có static entry và tnsnames đủ 2 node. Listener chỉ bind được vào IP **của chính host đó** — mỗi node phải để đúng IP của mình, nếu để nhầm IP node kia sẽ bị `Linux Error: 99: Cannot assign requested address`.

Mỗi node khai **2 static entry**, mục đích khác nhau:

| Entry | Dùng khi |
|---|---|
| `GLOBAL_DBNAME = <SID>` | RMAN nối vào instance đang NOMOUNT lúc duplicate ([2.1](#21--đồng-bộ-dự-phòng-oracle-data-guard-opriostba)) |
| `GLOBAL_DBNAME = <SID>_DGMGRL` | Broker khởi động lại instance từ xa sau `SWITCHOVER` ([4.2](#42--di-trú-oracle-data-guard-mở-rộng--switchover-sang-31)) |

Cả hai đều phải **static**: instance lúc đó đang down hoặc chưa mount nên không tự đăng ký service động được. Thiếu entry `_DGMGRL` thì switchover vẫn đổi vai trò thành công, nhưng broker không dựng lại được primary cũ và báo `ORA-12514: TNS:listener does not currently know of service requested`.

Trên **.21** (user **oracle**):

```bash
cat > $ORACLE_HOME/network/admin/listener.ora << 'EOF'
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.200.21)(PORT = 1521))))

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = OPRI)
      (ORACLE_HOME = /u01/app/oracle/product/19.3.0/dbhome_1)
      (SID_NAME = OPRI))
    (SID_DESC =
      (GLOBAL_DBNAME = OPRI_DGMGRL)
      (ORACLE_HOME = /u01/app/oracle/product/19.3.0/dbhome_1)
      (SID_NAME = OPRI)))
EOF
```

Trên **.22** — đổi HOST thành `10.10.200.22`, GLOBAL_DBNAME/SID_NAME thành `OSTBA`:

```bash
cat > $ORACLE_HOME/network/admin/listener.ora << 'EOF'
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.200.22)(PORT = 1521))))

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = OSTBA)
      (ORACLE_HOME = /u01/app/oracle/product/19.3.0/dbhome_1)
      (SID_NAME = OSTBA))
    (SID_DESC =
      (GLOBAL_DBNAME = OSTBA_DGMGRL)
      (ORACLE_HOME = /u01/app/oracle/product/19.3.0/dbhome_1)
      (SID_NAME = OSTBA)))
EOF
```

`tnsnames.ora` giống hệt nhau trên **cả 2 node**:

```bash
cat > $ORACLE_HOME/network/admin/tnsnames.ora << 'EOF'
OPRI =
  (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.200.21)(PORT = 1521))
    (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = OPRI)))
OSTBA =
  (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.200.22)(PORT = 1521))
    (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = OSTBA)))
EOF
```

**`sqlnet.ora` — nới inbound timeout, cũng giống nhau trên cả 2 node:**

```bash
cat > $ORACLE_HOME/network/admin/sqlnet.ora << 'EOF'
SQLNET.INBOUND_CONNECT_TIMEOUT=180
EOF

cat >> $ORACLE_HOME/network/admin/listener.ora << 'EOF'

INBOUND_CONNECT_TIMEOUT_LISTENER=180
EOF

lsnrctl stop; lsnrctl start
```

> Mặc định Oracle cho phép 60s để hoàn tất bắt tay xác thực. Khi standby bận apply redo, kết nối redo transport từ primary dễ vượt ngưỡng này → alert log đầy `ORA-3136: inbound connection timed out` + `TNS-12535`, và primary thấy destination lỗi. Nới lên 180s là khuyến nghị chuẩn cho Data Guard, đặt ngay từ đầu cho đỡ phải debug sau.

**Kết quả mong đợi:** từ .21 `tnsping OSTBA` OK và ngược lại `tnsping OPRI` từ .22 OK.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/16.png"/>

---

# Phần 2 — Đồng bộ dữ liệu

Kết thúc phần này: dữ liệu chảy ra khỏi lõi theo **hai đường song song, giải hai bài toán khác nhau**. Data Guard nhân bản nguyên cả DB `OPRI` ở tầng block — dùng khi .21 chết, không ai đọc nó lúc bình thường. GoldenGate chỉ rót thay đổi của schema `FINACC` sang 3 vệ tinh — để các đội đọc hằng ngày mà không đụng vào lõi. Hai đường không thay thế nhau: mất Data Guard là mất dự phòng, mất GoldenGate là các đội mất bản sao để khai thác.

### 2.1 — Đồng bộ dự phòng: Oracle Data Guard OPRI→OSTBA

**Mục tiêu:** nhân bản DB lõi từ primary qua network (`FROM ACTIVE DATABASE`), giao việc ship/apply redo cho DG Broker. Đây là lớp **dự phòng/HA** cho Oracle lõi — physical standby giống hệt ở tầng block.

**Dựng standby bằng RMAN duplicate.** Trên **.21**, copy password file sang .22 (phải giống hệt giữa các node DG):

```bash
su - oracle
# đích dùng đường dẫn TUYỆT ĐỐI — $ORACLE_HOME ở vế remote do shell non-login của scp
# diễn giải, thường rỗng, dễ ghi lạc chỗ mà vẫn báo "100%"
scp $ORACLE_HOME/dbs/orapwOPRI \
    oracle@10.10.200.22:/u01/app/oracle/product/19.3.0/dbhome_1/dbs/orapwOSTBA
```

Verify 2 file **giống hệt nhau** (mọi node Data Guard phải chung một password file, lệch là hỏng xác thực redo transport):

```bash
# .21
md5sum $ORACLE_HOME/dbs/orapwOPRI
# .22
md5sum /u01/app/oracle/product/19.3.0/dbhome_1/dbs/orapwOSTBA
```

Trên **.22**, tạo pfile tối thiểu + start NOMOUNT, rồi duplicate. Phải `export ORACLE_SID=OSTBA` trước khi `STARTUP NOMOUNT` — `sqlplus / as sysdba` đặt tên vùng shared memory theo `$ORACLE_SID`, không theo tên pfile; nếu SID còn là `OPRI` thì RMAN nối auxiliary qua `@OSTBA` (listener trỏ `SID_NAME=OSTBA`) sẽ không tìm thấy realm và báo `ORA-27101 shared memory realm does not exist`:

```bash
su - oracle
export ORACLE_SID=OSTBA
mkdir -p /u01/oradata/OSTBA /u01/fra $ORACLE_BASE/admin/OSTBA/adump

cat > $ORACLE_HOME/dbs/initOSTBA.ora << 'EOF'
db_name=OPRI
db_unique_name=OSTBA
EOF

sqlplus / as sysdba << 'EOF'
STARTUP NOMOUNT PFILE='/u01/app/oracle/product/19.3.0/dbhome_1/dbs/initOSTBA.ora';
EXIT;
EOF

rman << 'EOF'
CONNECT TARGET sys/"Zxcasd123!@#"@OPRI;
CONNECT AUXILIARY sys/"Zxcasd123!@#"@OSTBA;
DUPLICATE TARGET DATABASE FOR STANDBY FROM ACTIVE DATABASE
  SPFILE
    SET db_unique_name='OSTBA'
    SET log_archive_config='DG_CONFIG=(OPRI,OSTBA)'
    SET standby_file_management='AUTO'
    SET db_file_name_convert='/OPRI/','/OSTBA/'
    SET log_file_name_convert='/OPRI/','/OSTBA/'
    SET db_recovery_file_dest='/u01/fra'
    SET audit_file_dest='/u01/app/oracle/admin/OSTBA/adump'
  NOFILENAMECHECK;
EOF
```

RMAN kết thúc `Finished Duplicate Db`. **Dọn pfile tạm + xác nhận standby đang chạy bằng SPFILE** — bước nhỏ nhưng bỏ qua là dính chuỗi lỗi nặng về sau:

```bash
# pfile tối giản chỉ dùng để NOMOUNT lúc duplicate; để lại thì lần restart sau
# instance có thể bám vào nó (chỉ 2 dòng, KHÔNG có control_files) -> ORA-00205
mv $ORACLE_HOME/dbs/initOSTBA.ora /tmp/initOSTBA.ora.bak
```

```sql
sqlplus / as sysdba

SELECT instance_name, status FROM v$instance;        -- OSTBA / MOUNTED
SELECT database_role, db_unique_name FROM v$database; -- PHYSICAL STANDBY / OSTBA
SHOW PARAMETER spfile                                 -- PHẢI ra .../dbs/spfileOSTBA.ora
EXIT;
```

> **`SHOW PARAMETER spfile` rỗng = standby đang chạy PFILE** → sau này mọi `ALTER SYSTEM ... SCOPE=BOTH` sẽ báo `ORA-32001`. Đừng chữa bằng `CREATE SPFILE FROM MEMORY` khi instance chỉ đang NOMOUNT — lúc đó memory không có `control_files`, SPFILE sinh ra sẽ thiếu tham số này và lần mount kế tiếp dính `ORA-00205: error in identifying control file`. Nếu lỡ mất SPFILE, dựng lại từ pfile đầy đủ (`CREATE SPFILE FROM PFILE='...'`) có khai rõ `control_files`.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/17.png"/>

**Bật Data Guard Broker.** Trên **cả .21 và .22**:

```sql
sqlplus / as sysdba
ALTER SYSTEM SET dg_broker_start=TRUE SCOPE=BOTH;
SHOW PARAMETER dg_broker_start;
EXIT;
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/18.png"/>

Trên **.21**, tạo configuration + enable:

```bash
dgmgrl << 'EOF'
CONNECT sys/"Zxcasd123!@#"@OPRI
CREATE CONFIGURATION labdg AS PRIMARY DATABASE IS OPRI CONNECT IDENTIFIER IS OPRI;
ADD DATABASE OSTBA AS CONNECT IDENTIFIER IS OSTBA MAINTAINED AS PHYSICAL;
ENABLE CONFIGURATION;
SHOW CONFIGURATION LAG;
EOF
```

(Mới enable có thể báo `ORA-16610`/`ORA-16810` vài chục giây — chờ rồi `SHOW CONFIGURATION` lại tới khi `SUCCESS`.)

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/19.png"/>

**Verify 3 tầng — đừng chỉ nhìn `SHOW CONFIGURATION`.** Ba lớp kiểm tra chạy ở 3 chỗ khác nhau, mỗi lớp bắt một loại lỗi âm thầm riêng:

| # | Chạy trên | Bằng công cụ | Kiểm tra điều gì |
|---|-----------|--------------|------------------|
| 1 | **.21** (primary) | `sqlplus / as sysdba` | Redo **gửi đi** có tới standby không |
| 2 | **.22** (standby) | `sqlplus / as sysdba` | Redo nhận được có **được apply** không |
| 3 | **.22** (standby) | `dgmgrl` | Password file khớp **2 chiều** giữa 2 node |

**(1) Trên `.21`** (user **oracle**) — kiểm tra redo transport:

```sql
sqlplus / as sysdba

SELECT dest_id, status, error FROM v$archive_dest WHERE dest_id=2;
EXIT;
```

Kỳ vọng: `status = VALID`, cột `error` **rỗng**. Nếu thấy `ORA-01033` / `ORA-16778` → transport đang đứt, standby chưa mount hoặc listener/tnsnames sai.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/20.png"/>

**(2) Trên `.22`** (user **oracle**) — kiểm tra tiến trình apply:

```sql
sqlplus / as sysdba

SELECT process, status, sequence# FROM v$managed_standby WHERE process='MRP0';
EXIT;
```

Kỳ vọng: có **1 dòng** `MRP0` ở `APPLYING_LOG` hoặc `WAIT_FOR_LOG`. Query trả về **0 dòng** = MRP không chạy → redo có về nhưng không ai apply, standby đứng yên trong khi `SHOW CONFIGURATION` vẫn có thể báo đẹp.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/21.png"/>

**(3) Trên `.22`** (user **oracle**) — xác thực password file cả 2 chiều:

```bash
dgmgrl << 'EOF'
CONNECT sys/"Zxcasd123!@#"@OSTBA
CONNECT sys/"Zxcasd123!@#"@OPRI
EOF
```

**Kết quả mong đợi:** `SHOW CONFIGURATION` → `SUCCESS`, `Transport Lag: 0 seconds` + `Apply Lag: 0 seconds`.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/22.png"/>

### 2.2 — Đồng bộ phục vụ khai thác: GoldenGate Hub → 3 vệ tinh

OGG Hub trên .21 = một điểm tập trung chạy Extract (capture Oracle) + 3 Replicat đẩy sang 3 vệ tinh. Đây là lớp **khai thác**: biến MariaDB/PostgreSQL/MSSQL thành bản sao dữ liệu tài chính để tách tải khỏi Oracle lõi.

#### 2.2.1 — OGG Hub + Extract (capture Oracle)

**Mục tiêu:** cài OGG Microservices for Oracle trên .21, tạo Service Manager + Deployment `Dep_Hub`, rồi Extract đẩy thay đổi `FINACC` ra trail. Chuẩn bị DB cho CDC (user **oracle**, sqlplus trên .21):

```sql
su - oracle
sqlplus / as sysdba
ALTER SYSTEM SET enable_goldengate_replication=TRUE SCOPE=BOTH;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;

CREATE USER oggadmin IDENTIFIED BY "Zxcasd123!@#" QUOTA UNLIMITED ON users;
GRANT connect, resource, dba TO oggadmin;
EXEC dbms_goldengate_auth.grant_admin_privilege('OGGADMIN');
EXIT;
```

Cài binary OGG MA for Oracle + tạo Service Manager/Deployment `Dep_Hub` (SM port 9001):

```bash
mkdir -p /u01/soft/ogg && cd /u01/soft/ogg
unzip -q /u01/soft/*for_Oracle*.zip
INST=$(find /u01/soft/ogg -name runInstaller -path '*Disk1*' | head -1)
# INSTALL_OPTION bắt buộc với 26ai/23.26 (thiếu -> INS-75022). ORA21c dùng chung cho 19c/21c/23ai.
"$INST" -silent -waitforcompletion \
  INSTALL_OPTION=ORA21c \
  SOFTWARE_LOCATION=/u01/app/oracle/product/ogg_ora \
  ORACLE_BASE=/u01/app/oracle UNIX_GROUP_NAME=oinstall

cat > /tmp/oggca.rsp << 'EOF'
oracle.install.responseFileVersion=/oracle/install/rspfmt_oggca_response_schema_v23c_1_0
CONFIGURATION_OPTION=ADD
DEPLOYMENT_NAME=Dep_Hub
ADMINISTRATOR_USER=oggadmin
ADMINISTRATOR_PASSWORD=Zxcasd123!@#
SERVICEMANAGER_DEPLOYMENT_HOME=/u01/app/oracle/deployments/ServiceManager
HOST_SERVICEMANAGER=10.10.200.21
PORT_SERVICEMANAGER=9001
SECURITY_ENABLED=false
STRONG_PWD_POLICY_ENABLED=false
CREATE_NEW_SERVICEMANAGER=true
REGISTER_SERVICEMANAGER_AS_A_SERVICE=true
OGG_SOFTWARE_HOME=/u01/app/oracle/product/ogg_ora
OGG_DEPLOYMENT_HOME=/u01/app/oracle/deployments/Dep_Hub
ENV_TNS_ADMIN=/u01/app/oracle/product/19.3.0/dbhome_1/network/admin
PORT_ADMINSRVR=9010
PORT_DISTSRVR=9011
PORT_RCVRSRVR=9012
PORT_PMSRVR=9013
UDP_PORT_PMSRVR=9014
METRICS_SERVER_ENABLED=true
OGG_SCHEMA=oggadmin
EOF

/u01/app/oracle/product/ogg_ora/bin/oggca.sh -silent -responseFile /tmp/oggca.rsp
sudo /u01/app/oracle/deployments/ServiceManager/bin/registerServiceManager.sh
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/23.png"/>

OGG Microservices tách nhiều service, mỗi cái một port: **9001 ServiceManager** (tổng quan), **9010 adminsrvr** (nơi làm việc chính: Extract/Replicat/Credential/TRANDATA), 9011 distsrvr (đẩy trail cross-host — dùng ở Phần 4), 9012 recvsrvr, 9013 pmsrvr (metrics). Web GUI `http://10.10.200.21:9001` (oggadmin / Zxcasd123!@#), nhưng 26ai gọn nhất qua **adminclient** (CLI).

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/24.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/25.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/26.png"/>

Trên .21, user `oracle`:

Param file cho Extract:

```bash
cat > /u01/app/oracle/deployments/Dep_Hub/etc/conf/ogg/EXT_FIN.prm << 'EOF'
EXTRACT EXT_FIN
USERIDALIAS cred_opri
EXTTRAIL ea
TABLE FINACC.*;
EOF
```

```bash
export OGG_HOME=/u01/app/oracle/product/ogg_ora
export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.21:9010 DEPLOYMENT Dep_Hub AS oggadmin PASSWORD Zxcasd123!@# !

-- (1) Credential login vào Oracle đọc redo
ALTER CREDENTIALSTORE ADD USER oggadmin@10.10.200.21:1521/OPRI PASSWORD Zxcasd123!@# ALIAS cred_opri
DBLOGIN USERIDALIAS cred_opri

-- (2) SCHEMATRANDATA: supplemental logging cho FINACC
ADD SCHEMATRANDATA FINACC ALLCOLS
INFO SCHEMATRANDATA FINACC

-- (3) Integrated Extract EXT_FIN capture FINACC.* -> trail ea
ADD EXTRACT EXT_FIN, INTEGRATED TRANLOG, BEGIN NOW
REGISTER EXTRACT EXT_FIN DATABASE
ADD EXTTRAIL ea, EXTRACT EXT_FIN

START EXTRACT EXT_FIN
INFO EXTRACT EXT_FIN
```

**Kết quả mong đợi:** `INFO EXTRACT EXT_FIN` → `Status RUNNING`, `Log Read Checkpoint` = `Oracle Integrated Redo Logs`.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/27.png"/>

#### 2.2.2 — Replicat → MariaDB

**Mục tiêu:** cài OGG for MySQL, nhận trail của Extract Oracle rồi Replicat áp vào MariaDB. User **oracle** trên .21 — cài binary + deployment `Dep_Maria` (thêm vào Service Manager 9001 đang chạy):

```bash
mkdir -p /u01/soft/ogg_mysql && cd /u01/soft/ogg_mysql
unzip -q /u01/soft/*for_MySQL*.zip
chmod -R u+x /u01/soft/ogg_mysql/ggs_Linux_x64_MySQL_services_shiphome/Disk1
INST=$(find /u01/soft/ogg_mysql -name runInstaller -path '*Disk1*' | head -1)
"$INST" -silent -waitforcompletion \
  INSTALL_OPTION=MYSQL \
  SOFTWARE_LOCATION=/u01/app/oracle/product/ogg_mysql \
  ORACLE_BASE=/u01/app/oracle UNIX_GROUP_NAME=oinstall

unset OGG_HOME
cat > /tmp/oggca_maria.rsp << 'EOF'
oracle.install.responseFileVersion=/oracle/install/rspfmt_oggca_response_schema_v23c_1_0
CONFIGURATION_OPTION=ADD
DEPLOYMENT_NAME=Dep_Maria
ADMINISTRATOR_USER=oggadmin
ADMINISTRATOR_PASSWORD=Zxcasd123!@#
SERVICEMANAGER_DEPLOYMENT_HOME=/u01/app/oracle/deployments/ServiceManager
HOST_SERVICEMANAGER=10.10.200.21
PORT_SERVICEMANAGER=9001
SECURITY_ENABLED=false
STRONG_PWD_POLICY_ENABLED=false
CREATE_NEW_SERVICEMANAGER=false
REGISTER_SERVICEMANAGER_AS_A_SERVICE=false
OGG_SOFTWARE_HOME=/u01/app/oracle/product/ogg_mysql
OGG_DEPLOYMENT_HOME=/u01/app/oracle/deployments/Dep_Maria
PORT_ADMINSRVR=9110
PORT_DISTSRVR=9111
PORT_RCVRSRVR=9112
PORT_PMSRVR=9113
UDP_PORT_PMSRVR=9114
METRICS_SERVER_ENABLED=true
OGG_SCHEMA=oggadmin
EOF
/u01/app/oracle/product/ogg_mysql/bin/oggca.sh -silent -responseFile /tmp/oggca_maria.rsp
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/28.png"/>

Có 2 bộ binary (`ogg_ora` capture Oracle, `ogg_mysql` apply MySQL) — chú ý set đúng `OGG_HOME` mỗi phần.

**Bước 1 — Credential `cred_maria`** (trên `Dep_Maria`, 9110):
```bash
export OGG_HOME=/u01/app/oracle/product/ogg_mysql; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.21:9110 DEPLOYMENT Dep_Maria AS oggadmin PASSWORD Zxcasd123!@# !
ALTER CREDENTIALSTORE ADD USER ggmysql@127.0.0.1:3306/finacc PASSWORD Zxcasd123!@# ALIAS cred_maria
DBLOGIN USERIDALIAS cred_maria
```
> OGG for MySQL bắt buộc USERID format `user@host:port/db` — thiếu `/finacc` báo `OGG-05005`.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/29.png"/>

**Bước 2 — Distribution Path `ea`→`eb`** (trên `Dep_Hub`, 9010) — chuyển trail từ deployment Extract sang Receiver của `Dep_Maria`:

```bash
export OGG_HOME=/u01/app/oracle/product/ogg_ora; export PATH=$OGG_HOME/bin:$PATH
adminclient
```
```sql
CONNECT http://10.10.200.21:9010 DEPLOYMENT Dep_Hub AS oggadmin PASSWORD Zxcasd123!@# !

-- credential domain Network (thứ tự: ALIAS trước DOMAIN trước PASSWORD)
ALTER CREDENTIALSTORE ADD USER oggadmin ALIAS path_maria DOMAIN Network PASSWORD Zxcasd123!@#

-- clause AUTHENTICATION đặt SAU target URI, thiếu -> OGG-12143
ADD DISTPATH p_ea_eb SOURCE trail://10.10.200.21:9011/services/v2/sources?trail=ea TARGET ws://10.10.200.21:9112/services/v2/targets?trail=eb AUTHENTICATION USERIDALIAS path_maria DOMAIN Network
START DISTPATH p_ea_eb
INFO DISTPATH p_ea_eb
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/30.png"/>

**Bước 3 — Replicat `RMAR`** (trên `Dep_Maria`, 9110):

```bash
cat > /u01/app/oracle/deployments/Dep_Maria/etc/conf/ogg/RMAR.prm << 'EOF'
REPLICAT RMAR
USERIDALIAS cred_maria
MAP FINACC.ACCOUNTS,     TARGET finacc.accounts;
MAP FINACC.TRANSACTIONS, TARGET finacc.transactions;
EOF
```

```bash
export OGG_HOME=/u01/app/oracle/product/ogg_mysql; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.21:9110 DEPLOYMENT Dep_Maria AS oggadmin PASSWORD Zxcasd123!@# !
DBLOGIN USERIDALIAS cred_maria
ADD CHECKPOINTTABLE finacc.oggchkpt
ADD REPLICAT RMAR, EXTTRAIL eb, CHECKPOINTTABLE finacc.oggchkpt

START REPLICAT RMAR
INFO REPLICAT RMAR
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/31.png"/>

**Kết quả CDC:** `INFO DISTPATH p_ea_eb` + `INFO REPLICAT RMAR` đều `RUNNING`; insert bên Oracle tự xuất hiện trong `docker exec mariadb mariadb -uroot -p'Zxcasd123!@#' -e "SELECT * FROM finacc.transactions;"`. Nhưng bảng vẫn thiếu dữ liệu có từ trước — đó là việc của bước tiếp theo.

**Initial Load — đồng bộ dữ liệu cũ.** Extract `EXT_FIN` tạo với `BEGIN NOW` nên dữ liệu seed từ [1.6](#16--tạo-primary-opri--schema-finacc-21) (2 account + 1 transaction) chưa sang. Dùng **initial-load Extract `SOURCEISTABLE`** (đọc thẳng snapshot bảng nguồn ra EXTFILE) + Replicat `HANDLECOLLISIONS`. Vì `Dep_Hub` và `Dep_Maria` cùng host .21 nên chỉ cần `cp` file:

```bash
# INITMA (SOURCEISTABLE) trên Dep_Hub
su - oracle
cat > /u01/app/oracle/deployments/Dep_Hub/etc/conf/ogg/INITMA.prm << 'EOF'
EXTRACT INITMA
USERIDALIAS cred_opri
EXTFILE ia PURGE
TABLE FINACC.*;
EOF
export OGG_HOME=/u01/app/oracle/product/ogg_ora; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.21:9010 DEPLOYMENT Dep_Hub AS oggadmin PASSWORD Zxcasd123!@# !
DBLOGIN USERIDALIAS cred_opri
ADD EXTRACT INITMA, SOURCEISTABLE
START EXTRACT INITMA
EXIT
```

```bash
# copy EXTFILE sang Dep_Maria (cùng host)
cp /u01/app/oracle/deployments/Dep_Hub/var/lib/data/ia* \
   /u01/app/oracle/deployments/Dep_Maria/var/lib/data/

# RINIT trên Dep_Maria
cat > /u01/app/oracle/deployments/Dep_Maria/etc/conf/ogg/RINIT.prm << 'EOF'
REPLICAT RINIT
USERIDALIAS cred_maria
HANDLECOLLISIONS
MAP FINACC.ACCOUNTS,     TARGET finacc.accounts;
MAP FINACC.TRANSACTIONS, TARGET finacc.transactions;
EOF
export OGG_HOME=/u01/app/oracle/product/ogg_mysql; export PATH=$OGG_HOME/bin:$PATH
adminclient
```
```sql
CONNECT http://10.10.200.21:9110 DEPLOYMENT Dep_Maria AS oggadmin PASSWORD Zxcasd123!@# !
DBLOGIN USERIDALIAS cred_maria
ADD REPLICAT RINIT, EXTFILE ia, CHECKPOINTTABLE finacc.oggchkpt
STOP REPLICAT RINIT
ALTER REPLICAT RINIT, EXTSEQNO 0, EXTRBA 0
START REPLICAT RINIT
STATS REPLICAT RINIT
INFO REPLICAT RINIT
```

```bash
docker exec mariadb mariadb -uroot -p'Zxcasd123!@#' -e \
  "SELECT '--- ACCOUNTS ---' AS ''; SELECT * FROM finacc.accounts;
   SELECT '--- TRANSACTIONS ---' AS ''; SELECT * FROM finacc.transactions;"
```

**Kết quả sau initial load:** MariaDB có đủ 2 account + 1 transaction seed từ [1.6](#16--tạo-primary-opri--schema-finacc-21), cộng thêm mọi thay đổi CDC phát sinh sau đó — từ đây `RMAR` một mình gánh tiếp, `RINIT` không cần chạy lại.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/32.png"/>

#### 2.2.3 — Replicat → PostgreSQL

**Mục tiêu:** đẩy Oracle → PostgreSQL vẫn theo **kiến trúc hub** — cài thêm binary `ogg_pg` trên .21, deployment `Dep_PG` connect PostgreSQL remote `10.10.200.22:5432`.

> **PG khác MySQL ở tầng kết nối:** OGG for PostgreSQL đi qua **ODBC/DataDirect** (`ggpsql25.so`), không native — `oggca` cần khai driver ODBC.

**Cài binary + `Dep_PG`** (ports 9210–9213). Mấu chốt PG: response file bắt buộc `ENV_POSTGRESQL_ODBCINST` (thiếu → `FATAL INS-85077`):

```bash
mkdir -p /u01/soft/ogg_pg && cd /u01/soft/ogg_pg
unzip -q /u01/soft/*for_PostgreSQL*.zip
chmod -R u+x /u01/soft/ogg_pg/*PostgreSQL*shiphome/Disk1
INST=$(find /u01/soft/ogg_pg -name runInstaller -path '*Disk1*' | head -1)
"$INST" -silent -waitforcompletion \
  INSTALL_OPTION=PostgreSQL \
  SOFTWARE_LOCATION=/u01/app/oracle/product/ogg_pg \
  ORACLE_BASE=/u01/app/oracle UNIX_GROUP_NAME=oinstall

cat > /tmp/oggca_pg.rsp << 'EOF'
oracle.install.responseFileVersion=/oracle/install/rspfmt_oggca_response_schema_v23c_1_0
CONFIGURATION_OPTION=ADD
DEPLOYMENT_NAME=Dep_PG
ADMINISTRATOR_USER=oggadmin
ADMINISTRATOR_PASSWORD=Zxcasd123!@#
SERVICEMANAGER_DEPLOYMENT_HOME=/u01/app/oracle/deployments/ServiceManager
HOST_SERVICEMANAGER=10.10.200.21
PORT_SERVICEMANAGER=9001
SECURITY_ENABLED=false
STRONG_PWD_POLICY_ENABLED=false
CREATE_NEW_SERVICEMANAGER=false
REGISTER_SERVICEMANAGER_AS_A_SERVICE=false
OGG_SOFTWARE_HOME=/u01/app/oracle/product/ogg_pg
OGG_DEPLOYMENT_HOME=/u01/app/oracle/deployments/Dep_PG
ENV_POSTGRESQL_ODBCINST=/u01/app/oracle/product/ogg_pg/datadirect/odbcinst.ini
PORT_ADMINSRVR=9210
PORT_DISTSRVR=9211
PORT_RCVRSRVR=9212
PORT_PMSRVR=9213
UDP_PORT_PMSRVR=9214
METRICS_SERVER_ENABLED=true
OGG_SCHEMA=oggadmin
EOF
export OGG_HOME=/u01/app/oracle/product/ogg_pg
$OGG_HOME/bin/oggca.sh -silent -responseFile /tmp/oggca_pg.rsp
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/33.png"/>

**Credential `cred_pg`** (DSN-less, format `user@host:port/db`, host là PG remote .22):

```bash
export OGG_HOME=/u01/app/oracle/product/ogg_pg; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.21:9210 DEPLOYMENT Dep_PG AS oggadmin PASSWORD Zxcasd123!@# !
ALTER CREDENTIALSTORE ADD USER ggpg@10.10.200.22:5432/finacc PASSWORD Zxcasd123!@# ALIAS cred_pg
DBLOGIN USERIDALIAS cred_pg
LIST TABLES public.*
```
> Bảng PG ở schema `public` → target Replicat là `public.*` (khác MariaDB `finacc.*`).

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/34.png"/>

**Distribution Path `ea`→`ec`** (trên `Dep_Hub`, port 9212, trail `ec`, alias `path_pg`):
```bash
export OGG_HOME=/u01/app/oracle/product/ogg_ora; export PATH=$OGG_HOME/bin:$PATH
adminclient
```
```sql
CONNECT http://10.10.200.21:9010 DEPLOYMENT Dep_Hub AS oggadmin PASSWORD Zxcasd123!@# !
ALTER CREDENTIALSTORE ADD USER oggadmin ALIAS path_pg DOMAIN Network PASSWORD Zxcasd123!@#
ADD DISTPATH p_ea_ec SOURCE trail://10.10.200.21:9011/services/v2/sources?trail=ea TARGET ws://10.10.200.21:9212/services/v2/targets?trail=ec AUTHENTICATION USERIDALIAS path_pg DOMAIN Network
START DISTPATH p_ea_ec
INFO DISTPATH p_ea_ec
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/35.png"/>

**Replicat `RPPG` (CDC)** trên `Dep_PG`, MAP target `public.*`:
```bash
cat > /u01/app/oracle/deployments/Dep_PG/etc/conf/ogg/RPPG.prm << 'EOF'
REPLICAT RPPG
USERIDALIAS cred_pg
MAP FINACC.ACCOUNTS,     TARGET public.accounts;
MAP FINACC.TRANSACTIONS, TARGET public.transactions;
EOF
export OGG_HOME=/u01/app/oracle/product/ogg_pg; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.21:9210 DEPLOYMENT Dep_PG AS oggadmin PASSWORD Zxcasd123!@# !
DBLOGIN USERIDALIAS cred_pg
ADD CHECKPOINTTABLE public.oggchkpt
ADD REPLICAT RPPG, EXTTRAIL ec, CHECKPOINTTABLE public.oggchkpt
START REPLICAT RPPG
INFO REPLICAT RPPG
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/36.png"/>

**Initial Load — đồng bộ dữ liệu cũ.** Giống MariaDB nhưng EXTFILE là `ip` và target `public.*`. Extract `INITPG` (`SOURCEISTABLE`) trên `Dep_Hub` đọc snapshot bảng nguồn ra EXTFILE:

```bash
# INITPG (SOURCEISTABLE) trên Dep_Hub
su - oracle
cat > /u01/app/oracle/deployments/Dep_Hub/etc/conf/ogg/INITPG.prm << 'EOF'
EXTRACT INITPG
USERIDALIAS cred_opri
EXTFILE ip PURGE
TABLE FINACC.*;
EOF
export OGG_HOME=/u01/app/oracle/product/ogg_ora; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.21:9010 DEPLOYMENT Dep_Hub AS oggadmin PASSWORD Zxcasd123!@# !
DBLOGIN USERIDALIAS cred_opri
ADD EXTRACT INITPG, SOURCEISTABLE
START EXTRACT INITPG
EXIT
```

```bash
# copy EXTFILE sang Dep_PG (cùng host .21)
cp /u01/app/oracle/deployments/Dep_Hub/var/lib/data/ip* \
   /u01/app/oracle/deployments/Dep_PG/var/lib/data/

# RINITPG trên Dep_PG — target public.*, HANDLECOLLISIONS chống trùng với CDC
cat > /u01/app/oracle/deployments/Dep_PG/etc/conf/ogg/RINITPG.prm << 'EOF'
REPLICAT RINITPG
USERIDALIAS cred_pg
HANDLECOLLISIONS
MAP FINACC.ACCOUNTS,     TARGET public.accounts;
MAP FINACC.TRANSACTIONS, TARGET public.transactions;
EOF
export OGG_HOME=/u01/app/oracle/product/ogg_pg; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.21:9210 DEPLOYMENT Dep_PG AS oggadmin PASSWORD Zxcasd123!@# !
DBLOGIN USERIDALIAS cred_pg
ADD REPLICAT RINITPG, EXTFILE ip, CHECKPOINTTABLE public.oggchkpt
STOP REPLICAT RINITPG
ALTER REPLICAT RINITPG, EXTSEQNO 0, EXTRBA 0
START REPLICAT RINITPG
STATS REPLICAT RINITPG
INFO REPLICAT RINITPG
```

**Kết quả mong đợi:** `INFO REPLICAT RPPG` → `RUNNING`; PostgreSQL là bản sao đầy đủ. So sánh Oracle ↔ PostgreSQL:

```bash
# Nguồn Oracle (.21)
export ORACLE_SID=OPRI
sqlplus -s / as sysdba << 'EOF'
SET LINESIZE 200 PAGESIZE 100
SELECT id, acct_no, owner, balance FROM FINACC.ACCOUNTS ORDER BY id;
SELECT id, acct_id, amount, txn_type, TO_CHAR(created,'YYYY-MM-DD HH24:MI:SS') created
  FROM FINACC.TRANSACTIONS ORDER BY id;
EXIT;
EOF
```

```bash
# Đích PostgreSQL (SSH từ .21 sang .22)
ssh root@10.10.200.22 "docker exec postgres psql -U postgres -d finacc \
  -c 'SELECT * FROM accounts ORDER BY id;' \
  -c 'SELECT * FROM transactions ORDER BY id;'"
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/37.png"/>

#### 2.2.4 — Replicat → MS SQL Server (qua ODBC từ hub)

**Mục tiêu:** hub .21 (Linux) deliver Oracle → MS SQL Server remote (.23) qua **Microsoft ODBC Driver**. OGG for SQL Server chạy được trên Linux, apply remote qua ODBC — giữ mô hình hub tập trung.

**A. Cài Microsoft ODBC Driver 18 trên hub .21** (bằng root):

```bash
curl -s https://packages.microsoft.com/config/rhel/8/prod.repo | tee /etc/yum.repos.d/mssql-release.repo
ACCEPT_EULA=Y dnf install -y msodbcsql18 mssql-tools18 unixODBC unixODBC-devel
odbcinst -q -d                      # -> [ODBC Driver 18 for SQL Server]
# test từ .21 vào MSSQL (-C = trust cert)
export PATH=$PATH:/opt/mssql-tools18/bin
sqlcmd -S 10.10.200.23,1433 -U ggmssql -P 'Zxcasd123!@#' -d FINACC -C -Q "SELECT name FROM sys.tables;"
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/38.png"/>

**B. Cài binary OGG for SQL Server** (`INSTALL_OPTION=MSSQL`, bằng oracle):
```bash
su - oracle
mkdir -p /u01/soft/ogg_mssql && cd /u01/soft/ogg_mssql
unzip -q /u01/soft/*for_SQLServer*.zip
chmod -R u+x /u01/soft/ogg_mssql/*shiphome/Disk1
INST=$(find /u01/soft/ogg_mssql -name runInstaller -path '*Disk1*' | head -1)
"$INST" -silent -waitforcompletion \
  INSTALL_OPTION=MSSQL \
  SOFTWARE_LOCATION=/u01/app/oracle/product/ogg_mssql \
  ORACLE_BASE=/u01/app/oracle UNIX_GROUP_NAME=oinstall
```
> `INSTALL_OPTION=MSSQL` (không phải `SQLSERVER` — sẽ `INS-75022`; tên lấy theo shiphome `ggs_Linux_x64_MSSQL_...`).

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/39.png"/>

**C. Khai DSN ODBC + tạo deployment `Dep_MSSQL`.** SQL Server đi qua **DSN** (khác PG DSN-less) → cần cả `odbc.ini` (DSN) lẫn `odbcinst.ini` (driver). **Đặt DSN ở `/etc/odbc.ini`** (đường mặc định mọi tiến trình đọc được):
```bash
# DSN vào /etc/odbc.ini (bằng root)
cat >> /etc/odbc.ini << 'EOF'

[mssql_finacc]
Driver      = ODBC Driver 18 for SQL Server
Server      = 10.10.200.23,1433
Database    = FINACC
Encrypt     = no
EOF
```

```bash
su - oracle
# oggca tạo Dep_MSSQL (ports 9310-9313, join SM 9001).
export OGG_HOME=/u01/app/oracle/product/ogg_mssql
cat > /tmp/oggca_mssql.rsp << 'EOF'
oracle.install.responseFileVersion=/oracle/install/rspfmt_oggca_response_schema_v23c_1_0
CONFIGURATION_OPTION=ADD
DEPLOYMENT_NAME=Dep_MSSQL
ADMINISTRATOR_USER=oggadmin
ADMINISTRATOR_PASSWORD=Zxcasd123!@#
SERVICEMANAGER_DEPLOYMENT_HOME=/u01/app/oracle/deployments/ServiceManager
HOST_SERVICEMANAGER=10.10.200.21
PORT_SERVICEMANAGER=9001
SECURITY_ENABLED=false
STRONG_PWD_POLICY_ENABLED=false
CREATE_NEW_SERVICEMANAGER=false
REGISTER_SERVICEMANAGER_AS_A_SERVICE=false
OGG_SOFTWARE_HOME=/u01/app/oracle/product/ogg_mssql
OGG_DEPLOYMENT_HOME=/u01/app/oracle/deployments/Dep_MSSQL
ENV_MSSQL_ODBCINI=/etc/odbc.ini
ENV_MSSQL_ODBCINST=/etc
PORT_ADMINSRVR=9310
PORT_DISTSRVR=9311
PORT_RCVRSRVR=9312
PORT_PMSRVR=9313
UDP_PORT_PMSRVR=9314
METRICS_SERVER_ENABLED=true
OGG_SCHEMA=oggadmin
EOF
$OGG_HOME/bin/oggca.sh -silent -responseFile /tmp/oggca_mssql.rsp
```
> Kiểm env đúng: `PID=$(ss -tlnp|grep :9310|grep -oP 'pid=\K[0-9]+'); tr '\0' '\n' </proc/$PID/environ | grep ODBC` → `ODBCSYSINI=/etc`. Nếu lỡ tạo sai và cần xóa deployment: `oggca DROP` schema khó, dùng **REST**: `curl -sk -u oggadmin:'Zxcasd123!@#' -X DELETE http://10.10.200.21:9001/services/v2/deployments/Dep_MSSQL` rồi `rm -rf` deployment home.

**D. Credential + Replicat `RMSSQL` (CDC)** trên `Dep_MSSQL` (9310):
```bash
export OGG_HOME=/u01/app/oracle/product/ogg_mssql; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.21:9310 DEPLOYMENT Dep_MSSQL AS oggadmin PASSWORD Zxcasd123!@# !
ALTER CREDENTIALSTORE ADD USER ggmssql@mssql_finacc PASSWORD Zxcasd123!@# ALIAS cred_mssql
DBLOGIN USERIDALIAS cred_mssql
LIST TABLES dbo.*
ADD CHECKPOINTTABLE dbo.oggchkpt
ADD REPLICAT RMSSQL, EXTTRAIL ed, CHECKPOINTTABLE dbo.oggchkpt
EXIT
```

```bash
cat > /u01/app/oracle/deployments/Dep_MSSQL/etc/conf/ogg/RMSSQL.prm << 'EOF'
REPLICAT RMSSQL
USERIDALIAS cred_mssql
MAP FINACC.ACCOUNTS,     TARGET dbo.accounts;
MAP FINACC.TRANSACTIONS, TARGET dbo.transactions;
EOF
```

**E. Distribution Path `ea`→`ed`** trên hub `Dep_Hub` (9010):
```bash
export OGG_HOME=/u01/app/oracle/product/ogg_ora; export PATH=$OGG_HOME/bin:$PATH
adminclient
```
```sql
CONNECT http://10.10.200.21:9010 DEPLOYMENT Dep_Hub AS oggadmin PASSWORD Zxcasd123!@# !
ALTER CREDENTIALSTORE ADD USER oggadmin ALIAS path_mssql DOMAIN Network PASSWORD Zxcasd123!@#
ADD DISTPATH p_ea_ed SOURCE trail://10.10.200.21:9011/services/v2/sources?trail=ea TARGET ws://10.10.200.21:9312/services/v2/targets?trail=ed AUTHENTICATION USERIDALIAS path_mssql DOMAIN Network
START DISTPATH p_ea_ed
INFO DISTPATH p_ea_ed
```
Rồi về `Dep_MSSQL` (9310): `START REPLICAT RMSSQL` → `INFO REPLICAT RMSSQL` = RUNNING.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/40.png"/>

**F. Initial Load — đồng bộ dữ liệu cũ.** Giống Maria/PG, EXTFILE là `ie` và target `dbo.*`. Lưu ý MSSQL: **tên group ≤ 8 ký tự** → dùng `INITMS`/`RINITMS`. Extract `INITMS` (`SOURCEISTABLE`) trên `Dep_Hub` đọc snapshot bảng nguồn ra EXTFILE:

```bash
# INITMS (SOURCEISTABLE) trên Dep_Hub
su - oracle
cat > /u01/app/oracle/deployments/Dep_Hub/etc/conf/ogg/INITMS.prm << 'EOF'
EXTRACT INITMS
USERIDALIAS cred_opri
EXTFILE ie PURGE
TABLE FINACC.*;
EOF
export OGG_HOME=/u01/app/oracle/product/ogg_ora; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.21:9010 DEPLOYMENT Dep_Hub AS oggadmin PASSWORD Zxcasd123!@# !
DBLOGIN USERIDALIAS cred_opri
ADD EXTRACT INITMS, SOURCEISTABLE
START EXTRACT INITMS
EXIT
```

```bash
# copy EXTFILE sang Dep_MSSQL (cùng host .21)
cp /u01/app/oracle/deployments/Dep_Hub/var/lib/data/ie* \
   /u01/app/oracle/deployments/Dep_MSSQL/var/lib/data/

# RINITMS trên Dep_MSSQL — target dbo.*, HANDLECOLLISIONS chống trùng với CDC
cat > /u01/app/oracle/deployments/Dep_MSSQL/etc/conf/ogg/RINITMS.prm << 'EOF'
REPLICAT RINITMS
USERIDALIAS cred_mssql
HANDLECOLLISIONS
MAP FINACC.ACCOUNTS,     TARGET dbo.accounts;
MAP FINACC.TRANSACTIONS, TARGET dbo.transactions;
EOF
export OGG_HOME=/u01/app/oracle/product/ogg_mssql; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.21:9310 DEPLOYMENT Dep_MSSQL AS oggadmin PASSWORD Zxcasd123!@# !
DBLOGIN USERIDALIAS cred_mssql
ADD REPLICAT RINITMS, EXTFILE ie, CHECKPOINTTABLE dbo.oggchkpt
STOP REPLICAT RINITMS
ALTER REPLICAT RINITMS, EXTSEQNO 0, EXTRBA 0
START REPLICAT RINITMS
STATS REPLICAT RINITMS
INFO REPLICAT RINITMS
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/41.png"/>

**Kết quả mong đợi:** insert bên Oracle `FINACC.TRANSACTIONS` → tự xuất hiện trong MSSQL `FINACC.dbo.transactions`; sau initial load, `STATS RINITMS` báo `accounts: 2 inserts`, `transactions: 1 insert` — đúng bằng dữ liệu seed ở [1.6](#16--tạo-primary-opri--schema-finacc-21).

```sql
USE FINACC;
GO

SELECT * FROM dbo.accounts;
SELECT * FROM dbo.transactions;
GO
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/42.png"/>

#### 2.2.5 — Tổng kết deployment GoldenGate

Sau 3 nhánh replicat, ServiceManager `9001` trên .21 quản lý **4 deployment** — mỗi deployment gắn với một build binary riêng vì OGG tách binary theo loại DB đích:

| Deployment | Binary (`OGG_HOME`) | Ports | DB đích | Credential |
|---|---|---|---|---|
| ServiceManager | — | 9001 | — | — |
| `Dep_Hub` | `ogg_ora` | 9010–9013 | Oracle `OPRI` (.21) — nguồn | `cred_opri` |
| `Dep_Maria` | `ogg_mysql` | 9110–9113 | MariaDB `finacc` (.21, docker) | `cred_maria` |
| `Dep_PG` | `ogg_pg` | 9210–9213 | PostgreSQL `finacc` (.22, ODBC) | `cred_pg` |
| `Dep_MSSQL` | `ogg_mssql` | 9310–9313 | MSSQL `FINACC` (.23, ODBC DSN) | `cred_mssql` |

Luồng CDC và initial load của từng nhánh:

| Nhánh | Extract (Hub) | Trail | DISTPATH | Replicat | Initial load |
|---|---|---|---|---|---|
| → MariaDB | `EXT_FIN` | `ea` → `eb` | `p_ea_eb` | `RMAR` | `INITMA` → `ia` → `RINIT` |
| → PostgreSQL | `EXT_FIN` | `ea` → `ec` | `p_ea_ec` | `RPPG` | `INITPG` → `ip` → `RINITPG` |
| → MSSQL | `EXT_FIN` | `ea` → `ed` | `p_ea_ed` | `RMSSQL` | `INITMS` → `ie` → `RINITMS` |

Chỉ **một** Extract `EXT_FIN` capture redo của Oracle ra trail `ea`; Distribution Service fan-out `ea` sang 3 trail đích `eb`/`ec`/`ed`. Nhờ vậy thêm một DB vệ tinh chỉ cần cài binary + tạo deployment + thêm distpath — **không đụng gì tới Extract ở nguồn**, tải trên DB nguồn không tăng theo số đích.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/43.png"/>

---

# Phần 3 — Đối soát dữ liệu

Replicat báo `RUNNING` chỉ nói rằng tiến trình còn sống, không nói bản sao còn khớp. Một row bị apply lệch kiểu dữ liệu, một lần ai đó sửa tay bên vệ tinh, một transaction rơi giữa lúc restart — tất cả đều xảy ra mà `INFO REPLICAT` vẫn xanh. Kết thúc phần này: có **bằng chứng ở mức row** rằng lõi và bản sao khớp nhau, Veridata lo cặp Oracle↔MSSQL, MariaDB/PostgreSQL soi tay bằng SQL.

### 3.1 — Cài Veridata (.22) + 2 agent (Oracle, SQL Server)

**Mục tiêu:** dựng Veridata Server + 2 agent, tất cả trên **.22**. Agent đọc DB qua JDBC/OCI nên kết nối Oracle (.21) và MSSQL (.23) qua mạng.

**A. Cài nền tảng** (bằng root cài JDK, rồi **oracle** cho FMW/Veridata/RCU — OUI từ chối chạy bằng root):

```bash
dnf install -y /u01/soft/*jdk-8u491*.rpm        # bằng root
su - oracle
# FMW Infrastructure
cat > /tmp/fmw.rsp << 'EOF'
[ENGINE]
Response File Version=1.0.0.0.0
[GENERIC]
ORACLE_HOME=/u01/app/oracle/product/fmw12c
INSTALL_TYPE=Fusion Middleware Infrastructure
DECLINE_AUTO_UPDATES=true
EOF
cat > /tmp/oraInst.loc << 'EOF'
inventory_loc=/u01/app/oraInventory
inst_group=oinstall
EOF
cd /u01/soft && unzip -o -q *Infrastructure*.zip
java -jar /u01/soft/fmw_12.2.1.4.0_infrastructure.jar -silent -responseFile /tmp/fmw.rsp -invPtrLoc /tmp/oraInst.loc
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/44.png"/>

```bash
# Veridata Server — jar tên fmw_12.2.1.4.0_ogg.jar, INSTALL_TYPE riêng (nhãn từ Disk1/stage/distributions/GG_Veridata_*.xml)
cd /u01/soft && unzip -o -q *Veridata*.zip
cat > /tmp/veridata.rsp << 'EOF'
[ENGINE]
Response File Version=1.0.0.0.0
[GENERIC]
ORACLE_HOME=/u01/app/oracle/product/fmw12c
INSTALL_TYPE=GoldenGate Veridata Web Server Installation
DECLINE_AUTO_UPDATES=true
EOF
java -jar /u01/soft/fmw_12.2.1.4.0_ogg.jar -silent -responseFile /tmp/veridata.rsp -invPtrLoc /tmp/oraInst.loc
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/45.png"/>

```bash
{ echo 'Zxcasd123!@#'; for i in $(seq 10); do echo 'Zxcasd123#'; done; } | \
/u01/app/oracle/product/fmw12c/oracle_common/bin/rcu -silent -createRepository \
  -databaseType ORACLE -connectString 10.10.200.21:1521:OPRI \
  -dbUser sys -dbRole sysdba -schemaPrefix LABVD \
  -component STB -component OPSS -component IAU -component IAU_APPEND \
  -component IAU_VIEWER -component VERIDATA -f
```

> Dòng đầu là mật khẩu `sys` (`Zxcasd123!@#`), các dòng sau là mật khẩu đặt cho schema RCU — `seq 10` chỉ để dư, RCU đọc bao nhiêu schema thì lấy bấy nhiêu dòng. Schema **cố ý dùng `Zxcasd123#`, bỏ `!@`**: mật khẩu này sẽ nằm trong JDBC URL và connect string của WLST ở các bước sau, mà `@` ở đó bị hiểu là dấu ngăn giữa credential và host — đúng cái bẫy `ORA-12154` gặp lại ở [G.3](#g-veridata-mới-trên-32).

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/46.png"/>

**B. Tạo domain WebLogic bằng WLST** (CLI, không cần GUI) + cấp quyền:

```bash
su - oracle
cat > /tmp/veridata_domain.py << 'EOF'
oh = '/u01/app/oracle/product/fmw12c'
readTemplate(oh + '/wlserver/common/templates/wls/wls.jar')
addTemplate(oh + '/oracle_common/common/templates/wls/oracle.jrf_template.jar')
addTemplate(oh + '/veridata/common/templates/wls/veridata_web_template.jar')
setOption('DomainName', 'veridata_domain')
setOption('ServerStartMode', 'prod')
setOption('JavaHome', '/usr/java/default')
cd('/Security/base_domain/User/weblogic')
cmo.setPassword('Zxcasd123!@#')
cd('/JDBCSystemResource/LocalSvcTblDataSource/JdbcResource/LocalSvcTblDataSource/JDBCDriverParams/NO_NAME_0')
set('DriverName','oracle.jdbc.OracleDriver')
set('URL','jdbc:oracle:thin:@10.10.200.21:1521/OPRI')
set('PasswordEncrypted','Zxcasd123#')
cd('Properties/NO_NAME_0/Property/user')
set('Value','LABVD_STB')
cd('/'); getDatabaseDefaults()
cd('/Servers/AdminServer'); set('ListenAddress','10.10.200.22'); set('ListenPort',7001)
cd('/Servers/VERIDATA_server1'); set('ListenAddress','10.10.200.22'); set('ListenPort',8830)
writeDomain('/u01/app/oracle/config/domains/veridata_domain')
closeTemplate(); exit()
EOF
/u01/app/oracle/product/fmw12c/oracle_common/common/bin/wlst.sh /tmp/veridata_domain.py

# boot.properties + start (AdminServer trước, đợi RUNNING mới start managed server)
DOMAIN=/u01/app/oracle/config/domains/veridata_domain
for S in AdminServer VERIDATA_server1; do
  mkdir -p $DOMAIN/servers/$S/security
  printf 'username=weblogic\npassword=Zxcasd123!@#\n' > $DOMAIN/servers/$S/security/boot.properties
done
nohup $DOMAIN/bin/startWebLogic.sh > /tmp/adminserver.log 2>&1 &
until grep -q 'Server state changed to RUNNING' /tmp/adminserver.log; do sleep 5; done
nohup $DOMAIN/bin/startManagedWebLogic.sh VERIDATA_server1 t3://10.10.200.22:7001 > /tmp/veridata.log 2>&1 &

# Cấp quyền: app Veridata map role Administrator -> group veridataAdministrator (không phải Administrators)
cat > /tmp/vd_grant.py << 'EOF'
connect('weblogic','Zxcasd123!@#','t3://10.10.200.22:7001')
atn = cmo.getSecurityConfiguration().getDefaultRealm().lookupAuthenticationProvider('DefaultAuthenticator')
if not atn.groupExists('veridataAdministrator'):
    atn.createGroup('veridataAdministrator', 'Veridata Administrator role')
atn.addMemberToGroup('veridataAdministrator', 'weblogic')
disconnect(); exit()
EOF
/u01/app/oracle/product/fmw12c/oracle_common/common/bin/wlst.sh /tmp/vd_grant.py
```
> Console: `http://10.10.200.22:8830/veridata`, login `weblogic`/`Zxcasd123!@#` (session mới sau khi cấp group).

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/47.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/48.png"/>

**C. Cài Agent + tạo 2 deployment (Oracle 7850, SQL Server 7852).** Web Server không kèm agent — cài Agent install type vào home riêng:

**Cài binary Veridata Agent** (một Oracle Home `vdagent`, dùng chung cho cả 2 agent). Trên .22, user `oracle`:
```bash
su - oracle
cat > /tmp/vdagent.rsp << 'EOF'
[ENGINE]
Response File Version=1.0.0.0.0
[GENERIC]
ORACLE_HOME=/u01/app/oracle/product/vdagent
INSTALL_TYPE=GoldenGate Veridata Agent Installation
DECLINE_AUTO_UPDATES=true
EOF
java -jar /u01/soft/fmw_12.2.1.4.0_ogg.jar -silent -responseFile /tmp/vdagent.rsp -invPtrLoc /tmp/oraInst.loc
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/49.png"/>

**Agent Oracle (7850)** → OPRI .21, driver `ojdbc8.jar` (copy từ Oracle DB home 19.3). Tạo deployment `vdagent_ora` rồi start. Trên .22:
```bash
export JAVA_HOME=/usr/java/default
A=/u01/app/oracle/product/vdagent/veridata/agent

ODEP=/u01/app/oracle/vdagent_ora
$A/agent_config.sh $ODEP
cp /u01/app/oracle/product/19.3.0/dbhome_1/jdbc/lib/ojdbc8.jar $ODEP/drivers/
cat > $ODEP/agent.properties << 'EOF'
server.port=7850
database.url=jdbc:oracle:thin:@10.10.200.21:1521/OPRI
server.jdbcDriver=ojdbc8.jar
server.driversLocation=drivers
server.useSsl=false
EOF
$ODEP/agent.sh start
sudo ss -tlnp | grep 7850
```

**Agent SQL Server (7852)** → MSSQL .23, driver DataDirect `wlsqlserver.jar` (bundle sẵn trong agent). Trên .22:
```bash
export JAVA_HOME=/usr/java/default
A=/u01/app/oracle/product/vdagent/veridata/agent
SDEP=/u01/app/oracle/vdagent_mssql
$A/agent_config.sh $SDEP
cp /u01/app/oracle/product/vdagent/oracle_common/modules/datadirect/wlsqlserver.jar $SDEP/drivers/
cat > $SDEP/agent.properties << 'EOF'
server.port=7852
database.url=jdbc:weblogic:sqlserver://10.10.200.23:1433;databaseName=FINACC
server.jdbcDriver=wlsqlserver.jar
server.driversLocation=drivers
server.useSsl=false
server.use2WaySsl=false
server.allowTrustedExpiredCertificates=true
server.identitystore.path=./config/certs/serverIdentity.jks
server.truststore.path=./config/certs/serverTrust.jks
server.identitystore.type=JKS
server.truststore.type=JKS
server.identitystore.keyfactory.alg.name=SunX509
server.truststore.keyfactory.alg.name=SunX509
server.ssl.algorithm.name=TLS
EOF
$SDEP/agent.sh start
sudo ss -tlnp | grep 7852
```

### 3.2 — Veridata verify Oracle ↔ MSSQL

**Mục tiêu:** khai 2 connection qua 2 agent, ghép cặp bảng Oracle↔MSSQL rồi chạy job so từng row. Toàn bộ làm trên console `http://10.10.200.22:8830/veridata` (`weblogic` / `Zxcasd123!@#`), vào **Configuration**.

**A. Connections** (Connection Configuration → New, wizard 3 bước):

| Connection | Agent host:port | Datasource Type | User / pass |
|---|---|---|---|
| `conn_ora` | `10.10.200.22:7850` | `Oracle` | `oggadmin` / `Zxcasd123!@#` |
| `conn_mssql` | `10.10.200.22:7852` | **`SQL Server`** | `ggmssql` / `Zxcasd123!@#` |

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/50.png"/>
<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/51.png"/>
<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/52.png"/>

Với `conn_mssql`: Task 2 chọn **`SQL Server`** → **Verify** ("Datasource type verified"); Task 3 user `ggmssql` → **Test Connection** ("Datasource connection was successful").

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/53.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/54.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/55.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/56.png"/>

**B. Group + Compare Pair** (Group Configuration → New): Group `grp_ora_mssql`, Source `conn_ora`, Target `conn_mssql`.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/57.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/58.png"/>

Vào **Compare Pair Configuration → Manual Mapping**: Source Schema `FINACC`; Target **Catalog `FINACC`, Schema `dbo`**. Chọn từng cặp → **Generate Compare Pair**:

| Source (Oracle) | Target (SQL Server) |
|---|---|
| `FINACC.ACCOUNTS` | `FINACC.dbo.accounts` |
| `FINACC.TRANSACTIONS` | `FINACC.dbo.transactions` |

(Bảng `oggchkpt`/`oggchkpt_lox` là checkpoint của OGG — bỏ qua.)

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/59.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/60.png"/>

**C. Job** (Job Configuration → New): tạo `job_ora_mssql` chứa group `grp_ora_mssql` → **Run / Execute Job**. Xem kết quả ở **Finished Jobs / Reports**.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/61.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/62.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/63.png"/>

**Kết quả mong đợi:** report của `job_ora_mssql` → mỗi bảng **`tables in sync`, `Rows out-of-sync: 0`** (accounts 2 row, transactions đủ). Report ghi rõ 2 agent: source `jdbc:oracle:thin:@10.10.200.21:1521/OPRI`, target `jdbc:weblogic:sqlserver://10.10.200.23:1433;databaseName=FINACC`.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/64.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/65.png"/>

### 3.3 — Kiểm chứng end-to-end (ghi 1 record → soi cả hệ)

**Mục tiêu:** một phép thử duy nhất chứng minh cả hệ đang chảy — ghi **1 record vào Oracle primary (.21)** rồi soi nó lan tới cả 4 đích.

```bash
# .21 — insert vào nguồn Oracle
export ORACLE_SID=OPRI
sqlplus -s / as sysdba << 'EOF'
INSERT INTO FINACC.TRANSACTIONS (acct_id, amount, txn_type, created) VALUES (1, 999, 'E2E', SYSDATE);
COMMIT;
EOF

sqlplus -s / as sysdba << 'EOF'
SET LINESIZE 200 PAGESIZE 100
SELECT id, acct_no, owner, balance FROM FINACC.ACCOUNTS ORDER BY id;
SELECT id, acct_id, amount, txn_type, TO_CHAR(created,'YYYY-MM-DD HH24:MI:SS') created
  FROM FINACC.TRANSACTIONS ORDER BY id;
EXIT;
EOF
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/66.png"/>

- **Thủ công — Data Guard (OSTBA .22):**

```bash
dgmgrl << 'EOF'
CONNECT sys/"Zxcasd123!@#"@OSTBA
SHOW CONFIGURATION LAG
EOF
```

`Apply Lag: 0 seconds` → record đã redo-apply sang standby. Muốn nhìn tận mắt dòng data thì mở read-only rồi `SELECT ... WHERE txn_type='E2E';`.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/67.png"/>

- **Thủ công — MariaDB (.21):**

```bash
docker exec mariadb mariadb -uroot -p'Zxcasd123!@#' -e \
  "SELECT * FROM finacc.accounts; SELECT * FROM finacc.transactions;"
```

- **Thủ công — PostgreSQL (.22):**

```bash
ssh root@10.10.200.22 "docker exec postgres psql -U postgres -d finacc \
  -c 'SELECT * FROM accounts;' -c 'SELECT * FROM transactions;'"
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/68.png"/>

- **Tự động — MSSQL (Veridata):** chạy lại `job_ora_mssql` → vẫn **0 out-of-sync** (record `E2E` khớp cả 2 bên).

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/69.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/70.png"/>

---

# Phần 4 — Di trú dữ liệu

Mục tiêu: chuyển toàn bộ hệ (Oracle lõi + 3 vệ tinh + OGG Hub + Veridata) sang bộ VM mới (.31/.32/.33) với downtime tối thiểu, rồi bỏ bộ cũ. Nguyên tắc: **dựng bản sao song song trước, cutover sau** — mỗi tầng dùng đúng công cụ của nó:

| Tầng | Cách di trú | Downtime |
|------|-------------|----------|
| Oracle lõi | Data Guard mở rộng 4 node + `SWITCHOVER TO NPRI` | vài giây |
| MariaDB | native replication .21→.31, promote | ~0 |
| PostgreSQL | streaming replication .22→.32, promote | ~0 |
| MSSQL | backup/restore .23→.33 | theo cỡ file backup |
| OGG Hub + Veridata | dựng lại trên .31/.32, trỏ node mới | — |

**Trình tự cutover** (quan trọng — tránh hổng CDC):

1. Dựng .31/.32/.33 + đưa .31/.32 vào Data Guard làm standby ([4.1](#41--dựng-bộ-vm-mới-3132-oracle-33-mssql), [4.2](#42--di-trú-oracle-data-guard-mở-rộng--switchover-sang-31)).
2. **Ngừng ghi** vào `FINACC` (downtime bắt đầu) → `SWITCHOVER TO NPRI` — từ đây bộ cũ **đóng băng**, không phát sinh thay đổi mới.
3. Di trú 3 vệ tinh từ bản đóng băng ([4.3](#43--di-trú-vệ-tinh-mariadb31-postgresql32-mssql33)).
4. Dựng hub OGG mới trên .31: Extract `BEGIN NOW` trên NPRI, 3 Replicat trỏ vệ tinh mới ([4.4](#44--chuyển-hub--veridata-sang-bộ-mới-kích-hoạt)) → **mở ghi lại** (downtime kết thúc).
5. Verify bằng Veridata mới trên .32, rồi bỏ bộ cũ ([4.5](#45--bỏ-212223)).

> Thực tế production: bước replication vệ tinh chạy **từ trước** switchover để đuổi kịp dần (downtime chỉ còn switchover). Trong lab nguồn đã đóng băng nên catch-up tức thời — cùng một quy trình.

### 4.1 — Dựng bộ VM mới (.31/.32 Oracle, .33 MSSQL)

**Mục tiêu:** dựng 3 VM đích tới trạng thái "sẵn sàng nhận dữ liệu" — OS, Oracle binary, listener, tnsnames 4 alias, SQL Server nghe 1433. Chưa tạo database nào ở bước này; data sẽ do Data Guard và backup/restore mang sang.

**A. Chuẩn bị OS .31/.32.** Bằng **root** trên **cả .31 và .32** (đổi hostname tương ứng `Oracle-pri-02` / `Oracle-sby-02`):

```bash
# .31 = Oracle-pri-02, .32 = Oracle-sby-02
hostnamectl set-hostname Oracle-pri-02

cat >> /etc/hosts << 'EOF'
10.10.200.21 Oracle-pri-01
10.10.200.22 Oracle-sby-01
10.10.200.23 MSSQL-01
10.10.200.31 Oracle-pri-02
10.10.200.32 Oracle-sby-02
10.10.200.33 MSSQL-02
EOF

# NTP
dnf install -y chrony
sed -i 's/^pool .*/pool vn.pool.ntp.org iburst/' /etc/chrony.conf
systemctl enable --now chronyd

# Lab: tắt firewalld + SELinux permissive
systemctl disable --now firewalld
sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
setenforce 0

# Gói preinstall 19c: tự tạo user oracle, group, kernel params, limits
dnf install -y oracle-database-preinstall-19c wget unzip tar nmap-ncat
echo 'oracle:Zxcasd123!@#' | chpasswd

# oracle sudo không cần password — các bước sau đổi qua lại oracle/root liên tục (Lab only)
echo 'oracle ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/oracle
chmod 440 /etc/sudoers.d/oracle
visudo -c

# Cây thư mục Oracle
mkdir -p /u01/app/oracle/product/19.3.0/dbhome_1 /u01/oradata /u01/fra /u01/soft
chown -R oracle:oinstall /u01
chmod -R 775 /u01

# Docker: .31 chạy MariaDB mới, .32 chạy PostgreSQL mới
dnf install -y dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
systemctl enable --now docker
```

Copy bộ cài từ node cũ sang .31, .32 (user `oracle`):

```bash
# .21 → .31/.32: binary Oracle 19c; riêng .31 (hub mới) thêm 4 gói OGG
scp /u01/soft/*db_home.zip oracle@10.10.200.31:/u01/soft/
scp /u01/soft/*db_home.zip oracle@10.10.200.32:/u01/soft/
scp /u01/soft/*for_Oracle*.zip /u01/soft/*for_MySQL*.zip \
    /u01/soft/*for_PostgreSQL*.zip /u01/soft/*for_SQLServer*.zip oracle@10.10.200.31:/u01/soft/

# .22 → .32: bộ Veridata (FMW Infrastructure + Veridata + JDK 8)
scp /u01/soft/*Infrastructure*.zip /u01/soft/*Veridata*.zip \
    /u01/soft/*jdk-8u491*.rpm oracle@10.10.200.32:/u01/soft/
```

**B. Cài Oracle 19c software-only trên .31/.32.** User **oracle**, y như node cũ — khác duy nhất `ORACLE_SID`.

Trên **.31**:

```bash
su - oracle

cat >> ~/.bash_profile << 'EOF'
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.3.0/dbhome_1
export ORACLE_SID=NPRI
export PATH=$ORACLE_HOME/bin:$PATH
EOF
source ~/.bash_profile
```

Trên **.32** — y hệt, chỉ đổi `ORACLE_SID`:

```bash
su - oracle

cat >> ~/.bash_profile << 'EOF'
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.3.0/dbhome_1
export ORACLE_SID=NSTB
export PATH=$ORACLE_HOME/bin:$PATH
EOF
source ~/.bash_profile
```

Xác nhận trước khi đi tiếp (chạy trên từng node, phải ra đúng SID của node đó):

```bash
echo $ORACLE_SID          # .31 -> NPRI ; .32 -> NSTB
```

Phần còn lại giống nhau trên **cả 2 node**:

```bash
unzip -q /u01/soft/*db_home.zip -d $ORACLE_HOME

# OL8 chưa có trong danh sách distro của installer 19.3 -> ép nhận diện
export CV_ASSUME_DISTID=OEL7.8

cd $ORACLE_HOME
./runInstaller -ignorePrereq -waitforcompletion -silent \
  oracle.install.option=INSTALL_DB_SWONLY \
  ORACLE_HOSTNAME=$(hostname) \
  UNIX_GROUP_NAME=oinstall \
  INVENTORY_LOCATION=/u01/app/oraInventory \
  ORACLE_HOME=$ORACLE_HOME ORACLE_BASE=$ORACLE_BASE \
  oracle.install.db.InstallEdition=EE \
  oracle.install.db.OSDBA_GROUP=dba oracle.install.db.OSOPER_GROUP=oper \
  oracle.install.db.OSBACKUPDBA_GROUP=backupdba oracle.install.db.OSDGDBA_GROUP=dgdba \
  oracle.install.db.OSKMDBA_GROUP=kmdba oracle.install.db.OSRACDBA_GROUP=racdba \
  SECURITY_UPDATES_VIA_MYORACLESUPPORT=false DECLINE_SECURITY_UPDATES=true
```

Sau `Successfully Setup Software`, chạy 2 script bằng **root**:

```bash
sudo /u01/app/oraInventory/orainstRoot.sh
sudo /u01/app/oracle/product/19.3.0/dbhome_1/root.sh
sqlplus -v
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/71.png"/>

**C. Static listener trên .31/.32** — 2 entry mỗi node y như [1.7](#17--listener--tnsnames-2122): `<SID>` cho RMAN duplicate, `<SID>_DGMGRL` cho broker khởi động lại instance sau switchover. Listener chỉ bind được IP của chính host đó. Trên **.31** (user `oracle`):

```bash
cat > $ORACLE_HOME/network/admin/listener.ora << 'EOF'
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.200.31)(PORT = 1521))))

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = NPRI)
      (ORACLE_HOME = /u01/app/oracle/product/19.3.0/dbhome_1)
      (SID_NAME = NPRI))
    (SID_DESC =
      (GLOBAL_DBNAME = NPRI_DGMGRL)
      (ORACLE_HOME = /u01/app/oracle/product/19.3.0/dbhome_1)
      (SID_NAME = NPRI)))
EOF
lsnrctl start
```

Trên **.32**:

```bash
cat > $ORACLE_HOME/network/admin/listener.ora << 'EOF'
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.200.32)(PORT = 1521))))

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = NSTB)
      (ORACLE_HOME = /u01/app/oracle/product/19.3.0/dbhome_1)
      (SID_NAME = NSTB))
    (SID_DESC =
      (GLOBAL_DBNAME = NSTB_DGMGRL)
      (ORACLE_HOME = /u01/app/oracle/product/19.3.0/dbhome_1)
      (SID_NAME = NSTB)))
EOF
lsnrctl start
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/72.png"/>

**D. tnsnames đủ 4 alias trên TẤT CẢ 4 node Linux** (.21/.22/.31/.32 — ghi đè cùng một file):

```bash
cat > $ORACLE_HOME/network/admin/tnsnames.ora << 'EOF'
OPRI =
  (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.200.21)(PORT = 1521))
    (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = OPRI)))
OSTBA =
  (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.200.22)(PORT = 1521))
    (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = OSTBA)))
NPRI =
  (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.200.31)(PORT = 1521))
    (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = NPRI)))
NSTB =
  (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.200.32)(PORT = 1521))
    (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = NSTB)))
EOF
```

Verify tnsping xuyên cả 4 node — chạy trên **bất kỳ node nào** (ví dụ .21), phải ping được cả 3 alias còn lại:

```bash
tnsping OSTBA
tnsping NPRI
tnsping NSTB
```

Mỗi lệnh phải ra `OK (xx msec)`. Lỗi `TNS-03505: Failed to resolve name` → tnsnames.ora chưa ghi đúng trên node đang đứng; lỗi `TNS-12541: TNS:no listener` → listener bên node đích (.31/.32) chưa `lsnrctl start` (quay lại bước C).

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/73.png"/>

**E. .33 (`MSSQL-02`) — Windows + SQL Server 2022 Developer.** Các bước như [1.2](#12--ms-sql-server-developer-windows-23) nhưng **chỉ tới hết phần network** (KHÔNG tạo database):

1. Windows Server 2025, IP tĩnh `10.10.200.33/24` gw `.1`, DNS `8.8.8.8`; giờ: `w32tm /config /manualpeerlist:vn.pool.ntp.org`.
2. Cài **SQL Server 2022 Developer** (KHÔNG Express — dính `OGG-05311`): New standalone installation → default instance `MSSQLSERVER` → Authentication **Mixed Mode**, `sa` = `Zxcasd123!@#`, Add Current User; cài thêm **SSMS**.
3. Configuration Manager: Protocols for MSSQLSERVER → **TCP/IP = Enabled**; tab IP Addresses → IPAll: xóa trắng *TCP Dynamic Ports*, đặt **TCP Port = 1433**; restart service `SQL Server (MSSQLSERVER)`.
4. Mở firewall (PowerShell admin):

```powershell
New-NetFirewallRule -DisplayName "SQL Server 1433" -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/74.png"/>

5. **Query kiểm tra instance** (SSMS → New Query → Execute) — xác nhận đúng edition và đã nghe TCP 1433 trước khi restore:

```sql
USE master;
GO
SELECT
  @@SERVERNAME                                AS server_name,
  SERVERPROPERTY('ProductVersion')            AS product_version,
  SERVERPROPERTY('Edition')                   AS edition,
  SERVERPROPERTY('EngineEdition')             AS engine_edition,          -- phải = 3
  SERVERPROPERTY('IsIntegratedSecurityOnly')  AS windows_auth_only;       -- phải = 0 (Mixed Mode)
GO

-- Instance đang nghe port nào (bước 3 đã ăn chưa)
SELECT port, state_desc, type_desc
FROM sys.dm_tcp_listener_states
WHERE type_desc = 'TSQL';
GO

-- Chưa có FINACC là ĐÚNG — chỉ 4 system database
SELECT name, database_id, state_desc FROM sys.databases ORDER BY database_id;
GO
```

> **KHÔNG tạo database/bảng/login** — DB `FINACC` (kèm bảng + user `ggmssql` bên trong) sẽ do `RESTORE` từ backup của .23 tạo ra ở [4.3](#43--di-trú-vệ-tinh-mariadb31-postgresql32-mssql33).

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/75.png"/>

**Kết quả mong đợi:** từ .21 `tnsping NPRI` và `tnsping NSTB` OK; `nc -vz 10.10.200.33 1433` → `succeeded`; trên .31/.32 `sqlplus -v` ra 19.0.

### 4.2 — Di trú Oracle: Data Guard mở rộng + switchover sang .31

**Mục tiêu:** thêm .31 (`NPRI`) / .32 (`NSTB`) làm physical standby của OPRI bằng RMAN duplicate (đúng pattern [2.1](#21--đồng-bộ-dự-phòng-oracle-data-guard-opriostba)), rồi `SWITCHOVER` — primary dời sang bộ mới, downtime chỉ vài giây.

**A. Mở DG_CONFIG 4 node** — chạy trên **cả .21 (OPRI) lẫn .22 (OSTBA)** (động, không restart):

```sql
su - oracle
sqlplus / as sysdba
ALTER SYSTEM SET log_archive_config='DG_CONFIG=(OPRI,OSTBA,NPRI,NSTB)' SCOPE=BOTH;
EXIT;
```

`log_archive_config` là tham số **instance-level, không tự đồng bộ giữa các node** — mỗi DB giữ SPFILE riêng nên phải set trên cả 2. Standby cần biết `NPRI`/`NSTB` tồn tại để nhận redo đúng sau switchover.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/76.png"/>


**B. Copy password file** từ .21 sang 2 node mới (phải giống hệt giữa các node DG):

```bash
# .21, user oracle
scp $ORACLE_HOME/dbs/orapwOPRI oracle@10.10.200.31:$ORACLE_HOME/dbs/orapwNPRI
scp $ORACLE_HOME/dbs/orapwOPRI oracle@10.10.200.32:$ORACLE_HOME/dbs/orapwNSTB
```

**C. RMAN duplicate .31 → NPRI.** Trên **.31** (user `oracle`) :

```bash
su - oracle
export ORACLE_SID=NPRI
mkdir -p /u01/oradata/NPRI /u01/fra $ORACLE_BASE/admin/NPRI/adump

cat > $ORACLE_HOME/dbs/initNPRI.ora << 'EOF'
db_name=OPRI
db_unique_name=NPRI
EOF

sqlplus / as sysdba << 'EOF'
STARTUP NOMOUNT PFILE='/u01/app/oracle/product/19.3.0/dbhome_1/dbs/initNPRI.ora';
EXIT;
EOF

rman << 'EOF'
CONNECT TARGET sys/"Zxcasd123!@#"@OPRI;
CONNECT AUXILIARY sys/"Zxcasd123!@#"@NPRI;
DUPLICATE TARGET DATABASE FOR STANDBY FROM ACTIVE DATABASE
  SPFILE
    SET db_unique_name='NPRI'
    SET log_archive_config='DG_CONFIG=(OPRI,OSTBA,NPRI,NSTB)'
    SET standby_file_management='AUTO'
    SET db_file_name_convert='/OPRI/','/NPRI/'
    SET log_file_name_convert='/OPRI/','/NPRI/'
    SET db_recovery_file_dest='/u01/fra'
    SET audit_file_dest='/u01/app/oracle/admin/NPRI/adump'
  NOFILENAMECHECK;
EOF
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/77.png"/>

**D. RMAN duplicate .32 → NSTB.** Trên **.32** (user `oracle`) — cùng pattern, mọi chỗ `NPRI` thành `NSTB`:

```bash
su - oracle
export ORACLE_SID=NSTB
mkdir -p /u01/oradata/NSTB /u01/fra $ORACLE_BASE/admin/NSTB/adump

cat > $ORACLE_HOME/dbs/initNSTB.ora << 'EOF'
db_name=OPRI
db_unique_name=NSTB
EOF

sqlplus / as sysdba << 'EOF'
STARTUP NOMOUNT PFILE='/u01/app/oracle/product/19.3.0/dbhome_1/dbs/initNSTB.ora';
EXIT;
EOF

rman << 'EOF'
CONNECT TARGET sys/"Zxcasd123!@#"@OPRI;
CONNECT AUXILIARY sys/"Zxcasd123!@#"@NSTB;
DUPLICATE TARGET DATABASE FOR STANDBY FROM ACTIVE DATABASE
  SPFILE
    SET db_unique_name='NSTB'
    SET log_archive_config='DG_CONFIG=(OPRI,OSTBA,NPRI,NSTB)'
    SET standby_file_management='AUTO'
    SET db_file_name_convert='/OPRI/','/NSTB/'
    SET log_file_name_convert='/OPRI/','/NSTB/'
    SET db_recovery_file_dest='/u01/fra'
    SET audit_file_dest='/u01/app/oracle/admin/NSTB/adump'
  NOFILENAMECHECK;
EOF
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/78.png"/>

Cả 2 lần duplicate kết thúc `Finished Duplicate Db`; kiểm tra trên .31/.32: `SELECT database_role FROM v$database;` → `PHYSICAL STANDBY`.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/79.png"/>

**E. Dọn `log_archive_dest_n` kế thừa trên .31/.32.** `DUPLICATE ... SPFILE` copy nguyên spfile của OPRI, mà OPRI đang do broker quản lý nên trong đó có sẵn `log_archive_dest_2 = 'service="ostba" ...'` — dòng này do broker tự sinh ở [2.1](#21--đồng-bộ-dự-phòng-oracle-data-guard-opriostba). Node mới thừa hưởng nó như tham số **thủ công**, broker không nhận là của mình, nên `ADD DATABASE` sẽ chết ngay với `ORA-16698: member has a LOG_ARCHIVE_DEST_n parameter with SERVICE attribute set`. Xóa trước trên **cả .31 và .32**:

```bash
sqlplus -s / as sysdba << 'EOF'
SET LINESIZE 200
COL name FOR a25
COL value FOR a90
SELECT name, value FROM v$parameter
 WHERE name LIKE 'log_archive_dest_%'
   AND value IS NOT NULL AND UPPER(value) LIKE '%SERVICE%';
EOF
```

Có `log_archive_dest_2` (đôi khi cả `_3`) thì xóa trắng — giữ nguyên `log_archive_dest_1` vì nó là `LOCATION=`, không chứa attribute `SERVICE`:

```bash
sqlplus -s / as sysdba << 'EOF'
ALTER SYSTEM SET log_archive_dest_2='' SCOPE=BOTH;
EOF
```

Broker sẽ tự dựng lại dest_2 đúng hướng sau khi add thành công.

**F. Thêm vào Broker, đợi lag 0, switchover.** Bật broker trên **.31 và .32**:

```sql
sqlplus / as sysdba
ALTER SYSTEM SET dg_broker_start=TRUE SCOPE=BOTH;
EXIT;
```

Trên **.21**, add 2 DB mới vào configuration `labdg` có sẵn rồi đợi đồng bộ.

```bash
dgmgrl << 'EOF'
CONNECT sys/"Zxcasd123!@#"@OPRI
ADD DATABASE NPRI AS CONNECT IDENTIFIER IS NPRI MAINTAINED AS PHYSICAL;
ADD DATABASE NSTB AS CONNECT IDENTIFIER IS NSTB MAINTAINED AS PHYSICAL;
ENABLE CONFIGURATION;
SHOW CONFIGURATION LAG;
EOF
```

`SHOW CONFIGURATION LAG` phải ra đủ **4 member** với `Configuration Status: SUCCESS` và lag của NPRI/NSTB về `0 seconds`. Chưa đạt thì lặp lại lệnh này, **đừng switchover vội** — đây là lý do tách ra 2 block: để chung một heredoc thì `SWITCHOVER` chạy bất chấp lag còn bao nhiêu.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/80.png"/>


Đạt rồi mới switchover. **Từ lệnh `SWITCHOVER` là bắt đầu downtime — ngừng ghi vào `FINACC` trước đó.**

`VALIDATE DATABASE NPRI` phải ghi `Ready for Switchover: Yes` — nếu `No` thì dừng, xử lý phần nó báo thiếu rồi chạy lại.

```bash
dgmgrl << 'EOF'
CONNECT sys/"Zxcasd123!@#"@OPRI
VALIDATE DATABASE NPRI;
SWITCHOVER TO NPRI;
SHOW CONFIGURATION;
EOF
```

Sau `New primary database "npri" is opening...` là vai trò đã đổi xong. Broker tiếp đó tự mount lại OPRI cũ thành standby — nếu listener .21 thiếu entry `OPRI_DGMGRL` ([1.7](#17--listener--tnsnames-2122)) thì bước này sẽ báo `ORA-12514` kèm `Please complete the following steps to finish switchover`. Switchover **vẫn thành công**, chỉ cần dựng tay trên **.21**:

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/81.png"/>

```bash
export ORACLE_SID=OPRI
sqlplus -s / as sysdba << 'EOF'
STARTUP MOUNT;
SELECT database_role, db_unique_name, open_mode FROM v$database;
EOF
```

Ra `PHYSICAL STANDBY / OPRI / MOUNTED` là xong. Apply đứng (`ORA-16810`) thì bật lại từ dgmgrl: `EDIT DATABASE opri SET STATE='APPLY-ON';`.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/82.png"/>

**Kết quả mong đợi:** `SHOW CONFIGURATION` (connect vào `@NPRI` — primary mới) → `NPRI - Primary database`, OPRI/OSTBA/NSTB đều standby, `SUCCESS`; trên .31 `SELECT database_role, open_mode FROM v$database;` → `PRIMARY / READ WRITE`.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/83.png"/>

> Sau switchover, Extract `EXT_FIN` cũ trên hub .21 sẽ **ABEND** (OPRI giờ là standby, không capture được) — kệ nó, toàn bộ hub cũ sẽ bỏ ở [4.5](#45--bỏ-212223). **Chưa ghi gì vào NPRI** cho tới khi hub mới chạy ([4.4](#44--chuyển-hub--veridata-sang-bộ-mới-kích-hoạt)) — giữ nguyên tắc không hổng CDC.

### 4.3 — Di trú vệ tinh: MariaDB→.31, PostgreSQL→.32, MSSQL→.33

**Mục tiêu:** chuyển data 3 vệ tinh sang bộ mới, mỗi họ DB dùng đúng cơ chế native của nó. Bộ cũ đã đóng băng từ lúc switchover → dump/backup bây giờ là **bản final**, không sợ lệch.

**A. MariaDB .21 → .31 (native replication).** Trên **.31** (root) dựng container như [1.3](#13--mariadb-106-docker-21), chỉ đổi `server_id`:

```bash
mkdir -p /opt/mariadb/conf /opt/mariadb/data
cat > /opt/mariadb/conf/lab.cnf << 'EOF'
[mysqld]
server_id                = 31
log_bin                  = /var/lib/mysql/mariadb-bin
binlog_format            = ROW
expire_logs_days         = 7
innodb_buffer_pool_size  = 512M
EOF

cat > /opt/mariadb/docker-compose.yml << 'EOF'
services:
  mariadb:
    image: mariadb:10.6
    container_name: mariadb
    restart: unless-stopped
    ports: ["3306:3306"]
    environment:
      MARIADB_ROOT_PASSWORD: 'Zxcasd123!@#'
    volumes:
      - ./conf/lab.cnf:/etc/mysql/conf.d/lab.cnf:ro
      - ./data:/var/lib/mysql
EOF

docker compose -f /opt/mariadb/docker-compose.yml up -d

# User cho OGG apply (hub mới) — user nằm ngoài dump finacc nên tạo lại
docker exec -i mariadb mariadb -uroot -p'Zxcasd123!@#' << 'EOF'
CREATE USER 'ggmysql'@'%' IDENTIFIED BY 'Zxcasd123!@#';
GRANT ALL PRIVILEGES ON finacc.* TO 'ggmysql'@'%';
GRANT SELECT ON *.* TO 'ggmysql'@'%';
FLUSH PRIVILEGES;
EOF
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/84.png"/>

Dump từ .21 (kèm tọa độ binlog) rồi nạp vào .31:

```bash
# .21 (root): dump + vị trí binlog (--master-data=2 ghi CHANGE MASTER dạng comment)
docker exec mariadb mariadb-dump -uroot -p'Zxcasd123!@#' \
  --single-transaction --master-data=2 --databases finacc > /tmp/finacc.sql
scp /tmp/finacc.sql root@10.10.200.31:/tmp/

# .31 (root): nạp + lấy tọa độ từ dump
docker exec -i mariadb mariadb -uroot -p'Zxcasd123!@#' < /tmp/finacc.sql
grep -m1 "CHANGE MASTER" /tmp/finacc.sql
# -- CHANGE MASTER TO MASTER_LOG_FILE='mariadb-bin.00000x', MASTER_LOG_POS=yyyy;
```

Trỏ .31 làm replica của .21 (điền FILE/POS vừa grep được) → đợi đuổi kịp → promote:

```bash
docker exec -i mariadb mariadb -uroot -p'Zxcasd123!@#' << 'EOF'
CHANGE MASTER TO
  MASTER_HOST='10.10.200.21', MASTER_PORT=3306,
  MASTER_USER='repl', MASTER_PASSWORD='Zxcasd123!@#',
  MASTER_LOG_FILE='<mariadb-bin.00000x>', MASTER_LOG_POS=<yyyy>;
START SLAVE;
EOF
docker exec mariadb mariadb -uroot -p'Zxcasd123!@#' -e "SHOW SLAVE STATUS\G" | grep -E 'Running|Behind'
# Slave_IO_Running: Yes / Slave_SQL_Running: Yes / Seconds_Behind_Master: 0

# Promote: cắt khỏi .21, thành master độc lập
docker exec mariadb mariadb -uroot -p'Zxcasd123!@#' -e "STOP SLAVE; RESET SLAVE ALL;"
```

Sau `RESET SLAVE ALL`, `SHOW SLAVE STATUS` trả về *Empty set* — đúng như mong đợi nhưng nhìn rỗng thì khó khẳng định. Xác nhận bằng dấu hiệu dương:

```bash
docker exec mariadb mariadb -uroot -p'Zxcasd123!@#' -e "
SELECT @@server_id AS server_id, @@read_only AS read_only, @@log_bin AS log_bin;
SHOW ALL SLAVES STATUS\G
SHOW MASTER STATUS;
SELECT COUNT(*) AS rows_tx FROM finacc.transactions;"
```

| Dòng | Phải ra | Nghĩa |
|---|---|---|
| `server_id` | `31` | đúng node mới |
| `read_only` | `0` | ghi được — không còn là replica |
| `log_bin` | `1` | tự sinh binlog, sẵn sàng làm nguồn cho hub mới ở [4.4](#44--chuyển-hub--veridata-sang-bộ-mới-kích-hoạt) |
| `SHOW ALL SLAVES STATUS` | *Empty set* | không còn master nào để bám |
| `SHOW MASTER STATUS` | `mariadb-bin.00000x` + pos | binlog **của chính .31** |
| `rows_tx` | khớp số row bên .21 | data sang đủ |

Dùng `SHOW ALL SLAVES STATUS` (cú pháp riêng của MariaDB) thay vì `SHOW SLAVE STATUS` vì nó liệt kê **mọi** multi-source connection — rỗng nghĩa là không sót connection ẩn nào.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/85.png"/>

**B. PostgreSQL .22 → .32 (streaming replication).** `pg_hba` trên .22 đã mở `replication` cho `repl` từ [1.4](#14--postgresql-13-docker-22). Trên **.32** (root): basebackup bằng container tạm (cờ `-R` tự sinh `standby.signal` + `primary_conninfo`), rồi start container **cùng compose như 1.4** — nó tự chạy ở chế độ standby:

```bash
mkdir -p /opt/pg/data
docker run --rm -e PGPASSWORD='Zxcasd123!@#' -v /opt/pg/data:/target \
  postgres:13 pg_basebackup -h 10.10.200.22 -U repl -D /target -R -X stream -P
chown -R 999:999 /opt/pg/data && chmod 700 /opt/pg/data   # uid postgres trong image = 999

# docker-compose.yml giống hệt 1.4 (listen_addresses, wal_level=logical, ...)
scp root@10.10.200.22:/opt/pg/docker-compose.yml /opt/pg/
docker compose -f /opt/pg/docker-compose.yml up -d

docker exec postgres psql -U postgres -c "SELECT pg_is_in_recovery();"   # t = đang standby
```

Promote thành primary độc lập:

```bash
docker exec postgres psql -U postgres -c "SELECT pg_promote();"
docker exec postgres psql -U postgres -c "SELECT pg_is_in_recovery();"   # f = đã promote
```

Xác nhận data đã sang đủ và node ghi được:

```bash
docker exec postgres psql -U postgres -d finacc \
  -c 'SELECT * FROM accounts;' -c 'SELECT * FROM transactions;'
docker exec postgres psql -U postgres -c "SHOW transaction_read_only;"   # off = ghi được
```

2 bảng phải khớp số row với .22, và `standby.signal` trong `/opt/pg/data` đã tự biến mất sau `pg_promote()` — đó là dấu hiệu .32 thành primary độc lập, không còn bám WAL của .22.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/86.png"/>

**C. MSSQL .23 → .33 (backup/restore).** Trên **.23** (SSMS hoặc sqlcmd):

```sql
BACKUP DATABASE FINACC TO DISK='C:\bak\finacc.bak' WITH INIT;
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/87.png"/>

Copy file sang .33 (PowerShell admin trên .23, qua admin share):

```powershell
New-Item -ItemType Directory \\10.10.200.33\c$\bak -Force
Copy-Item C:\bak\finacc.bak \\10.10.200.33\c$\bak\
```

Trên **.33** (SSMS): xem logical name rồi restore:

```sql
RESTORE FILELISTONLY FROM DISK='C:\bak\finacc.bak';
-- logical name thường là FINACC / FINACC_log

RESTORE DATABASE FINACC FROM DISK='C:\bak\finacc.bak'
  WITH MOVE 'FINACC'     TO 'C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\DATA\FINACC.mdf',
       MOVE 'FINACC_log' TO 'C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\DATA\FINACC_log.ldf';
```

Login `ggmssql` nằm ở tầng instance nên **không theo file backup** — tạo lại rồi map vào user (đã có sẵn trong DB restore, đang orphan):

```sql
CREATE LOGIN ggmssql WITH PASSWORD = 'Zxcasd123!@#', CHECK_POLICY = OFF;
GO
USE FINACC;
ALTER USER ggmssql WITH LOGIN = ggmssql;   -- fix orphaned user
GO
```

**Kết quả mong đợi:** cả 3 vệ tinh mới có **đủ data như bản cũ** — MariaDB .31 / PostgreSQL .32 / MSSQL .33 đều ra đúng số dòng `accounts`, `transactions` (kể cả record `E2E` của [3.3](#33--kiểm-chứng-end-to-end-ghi-1-record--soi-cả-hệ)).

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/88.png"/>

### 4.4 — Chuyển Hub + Veridata sang bộ mới, kích hoạt

**Mục tiêu:** dựng lại tầng công cụ trên bộ mới rồi mở ghi lại — đây là bước kết thúc downtime. NPRI là bản physical của OPRI nên mọi thứ nằm **trong DB** đã sang sẵn — `enable_goldengate_replication=TRUE` (spfile duplicate mang theo), supplemental logging, user `oggadmin`, `SCHEMATRANDATA FINACC`, kể cả RCU schema `LABVD*` của Veridata. Phần phải làm lại là những gì nằm **ngoài DB**: binary OGG, deployment, credential, trail, agent.

#### A. Cài 4 binary OGG + Service Manager + 4 deployment trên .31

**A.1 — OGG for Oracle + `Dep_Hub` (9010–9014) + Service Manager 9001.** User **oracle** trên **.31**:

```bash
mkdir -p /u01/soft/ogg && cd /u01/soft/ogg
unzip -q /u01/soft/*for_Oracle*.zip
INST=$(find /u01/soft/ogg -name runInstaller -path '*Disk1*' | head -1)
"$INST" -silent -waitforcompletion \
  INSTALL_OPTION=ORA21c \
  SOFTWARE_LOCATION=/u01/app/oracle/product/ogg_ora \
  ORACLE_BASE=/u01/app/oracle UNIX_GROUP_NAME=oinstall

cat > /tmp/oggca.rsp << 'EOF'
oracle.install.responseFileVersion=/oracle/install/rspfmt_oggca_response_schema_v23c_1_0
CONFIGURATION_OPTION=ADD
DEPLOYMENT_NAME=Dep_Hub
ADMINISTRATOR_USER=oggadmin
ADMINISTRATOR_PASSWORD=Zxcasd123!@#
SERVICEMANAGER_DEPLOYMENT_HOME=/u01/app/oracle/deployments/ServiceManager
HOST_SERVICEMANAGER=10.10.200.31
PORT_SERVICEMANAGER=9001
SECURITY_ENABLED=false
STRONG_PWD_POLICY_ENABLED=false
CREATE_NEW_SERVICEMANAGER=true
REGISTER_SERVICEMANAGER_AS_A_SERVICE=true
OGG_SOFTWARE_HOME=/u01/app/oracle/product/ogg_ora
OGG_DEPLOYMENT_HOME=/u01/app/oracle/deployments/Dep_Hub
ENV_TNS_ADMIN=/u01/app/oracle/product/19.3.0/dbhome_1/network/admin
PORT_ADMINSRVR=9010
PORT_DISTSRVR=9011
PORT_RCVRSRVR=9012
PORT_PMSRVR=9013
UDP_PORT_PMSRVR=9014
METRICS_SERVER_ENABLED=true
OGG_SCHEMA=oggadmin
EOF
/u01/app/oracle/product/ogg_ora/bin/oggca.sh -silent -responseFile /tmp/oggca.rsp

sudo /u01/app/oracle/deployments/ServiceManager/bin/registerServiceManager.sh
```

**A.2 — OGG for MySQL + `Dep_Maria` (9110–9114):**

```bash
mkdir -p /u01/soft/ogg_mysql && cd /u01/soft/ogg_mysql
unzip -q /u01/soft/*for_MySQL*.zip
chmod -R u+x /u01/soft/ogg_mysql/ggs_Linux_x64_MySQL_services_shiphome/Disk1
INST=$(find /u01/soft/ogg_mysql -name runInstaller -path '*Disk1*' | head -1)
"$INST" -silent -waitforcompletion \
  INSTALL_OPTION=MYSQL \
  SOFTWARE_LOCATION=/u01/app/oracle/product/ogg_mysql \
  ORACLE_BASE=/u01/app/oracle UNIX_GROUP_NAME=oinstall

unset OGG_HOME
cat > /tmp/oggca_maria.rsp << 'EOF'
oracle.install.responseFileVersion=/oracle/install/rspfmt_oggca_response_schema_v23c_1_0
CONFIGURATION_OPTION=ADD
DEPLOYMENT_NAME=Dep_Maria
ADMINISTRATOR_USER=oggadmin
ADMINISTRATOR_PASSWORD=Zxcasd123!@#
SERVICEMANAGER_DEPLOYMENT_HOME=/u01/app/oracle/deployments/ServiceManager
HOST_SERVICEMANAGER=10.10.200.31
PORT_SERVICEMANAGER=9001
SECURITY_ENABLED=false
STRONG_PWD_POLICY_ENABLED=false
CREATE_NEW_SERVICEMANAGER=false
REGISTER_SERVICEMANAGER_AS_A_SERVICE=false
OGG_SOFTWARE_HOME=/u01/app/oracle/product/ogg_mysql
OGG_DEPLOYMENT_HOME=/u01/app/oracle/deployments/Dep_Maria
PORT_ADMINSRVR=9110
PORT_DISTSRVR=9111
PORT_RCVRSRVR=9112
PORT_PMSRVR=9113
UDP_PORT_PMSRVR=9114
METRICS_SERVER_ENABLED=true
OGG_SCHEMA=oggadmin
EOF
/u01/app/oracle/product/ogg_mysql/bin/oggca.sh -silent -responseFile /tmp/oggca_maria.rsp
```

**A.3 — OGG for PostgreSQL + `Dep_PG` (9210–9214)** — nhớ `ENV_POSTGRESQL_ODBCINST` (thiếu → `FATAL INS-85077`):

```bash
mkdir -p /u01/soft/ogg_pg && cd /u01/soft/ogg_pg
unzip -q /u01/soft/*for_PostgreSQL*.zip
chmod -R u+x /u01/soft/ogg_pg/*PostgreSQL*shiphome/Disk1
INST=$(find /u01/soft/ogg_pg -name runInstaller -path '*Disk1*' | head -1)
"$INST" -silent -waitforcompletion \
  INSTALL_OPTION=PostgreSQL \
  SOFTWARE_LOCATION=/u01/app/oracle/product/ogg_pg \
  ORACLE_BASE=/u01/app/oracle UNIX_GROUP_NAME=oinstall

cat > /tmp/oggca_pg.rsp << 'EOF'
oracle.install.responseFileVersion=/oracle/install/rspfmt_oggca_response_schema_v23c_1_0
CONFIGURATION_OPTION=ADD
DEPLOYMENT_NAME=Dep_PG
ADMINISTRATOR_USER=oggadmin
ADMINISTRATOR_PASSWORD=Zxcasd123!@#
SERVICEMANAGER_DEPLOYMENT_HOME=/u01/app/oracle/deployments/ServiceManager
HOST_SERVICEMANAGER=10.10.200.31
PORT_SERVICEMANAGER=9001
SECURITY_ENABLED=false
STRONG_PWD_POLICY_ENABLED=false
CREATE_NEW_SERVICEMANAGER=false
REGISTER_SERVICEMANAGER_AS_A_SERVICE=false
OGG_SOFTWARE_HOME=/u01/app/oracle/product/ogg_pg
OGG_DEPLOYMENT_HOME=/u01/app/oracle/deployments/Dep_PG
ENV_POSTGRESQL_ODBCINST=/u01/app/oracle/product/ogg_pg/datadirect/odbcinst.ini
PORT_ADMINSRVR=9210
PORT_DISTSRVR=9211
PORT_RCVRSRVR=9212
PORT_PMSRVR=9213
UDP_PORT_PMSRVR=9214
METRICS_SERVER_ENABLED=true
OGG_SCHEMA=oggadmin
EOF
export OGG_HOME=/u01/app/oracle/product/ogg_pg
$OGG_HOME/bin/oggca.sh -silent -responseFile /tmp/oggca_pg.rsp
```

**A.4 — ODBC + OGG for SQL Server + `Dep_MSSQL` (9310–9314).** Bằng **root**: cài driver Microsoft + khai DSN trỏ **.33**:

```bash
curl -s https://packages.microsoft.com/config/rhel/8/prod.repo | tee /etc/yum.repos.d/mssql-release.repo
ACCEPT_EULA=Y dnf install -y msodbcsql18 mssql-tools18 unixODBC unixODBC-devel

cat >> /etc/odbc.ini << 'EOF'

[mssql_finacc]
Driver      = ODBC Driver 18 for SQL Server
Server      = 10.10.200.33,1433
Database    = FINACC
Encrypt     = no
EOF

# test từ .31 tới MSSQL mới (-C = trust cert)
export PATH=$PATH:/opt/mssql-tools18/bin
sqlcmd -S 10.10.200.33,1433 -U ggmssql -P 'Zxcasd123!@#' -d FINACC -C -Q "SELECT name FROM sys.tables;"
```

> Lệnh test trên chạy **sau khi** đã restore FINACC ở [4.3](#43--di-trú-vệ-tinh-mariadb31-postgresql32-mssql33) — phải thấy `accounts`, `transactions`, `oggchkpt`.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/89.png"/>

Quay lại user **oracle**, cài binary + deployment:

```bash
su - oracle
mkdir -p /u01/soft/ogg_mssql && cd /u01/soft/ogg_mssql
unzip -q /u01/soft/*for_SQLServer*.zip
chmod -R u+x /u01/soft/ogg_mssql/*MSSQL*shiphome/Disk1
INST=$(find /u01/soft/ogg_mssql -name runInstaller -path '*Disk1*' | head -1)
"$INST" -silent -waitforcompletion \
  INSTALL_OPTION=MSSQL \
  SOFTWARE_LOCATION=/u01/app/oracle/product/ogg_mssql \
  ORACLE_BASE=/u01/app/oracle UNIX_GROUP_NAME=oinstall

export OGG_HOME=/u01/app/oracle/product/ogg_mssql
cat > /tmp/oggca_mssql.rsp << 'EOF'
oracle.install.responseFileVersion=/oracle/install/rspfmt_oggca_response_schema_v23c_1_0
CONFIGURATION_OPTION=ADD
DEPLOYMENT_NAME=Dep_MSSQL
ADMINISTRATOR_USER=oggadmin
ADMINISTRATOR_PASSWORD=Zxcasd123!@#
SERVICEMANAGER_DEPLOYMENT_HOME=/u01/app/oracle/deployments/ServiceManager
HOST_SERVICEMANAGER=10.10.200.31
PORT_SERVICEMANAGER=9001
SECURITY_ENABLED=false
STRONG_PWD_POLICY_ENABLED=false
CREATE_NEW_SERVICEMANAGER=false
REGISTER_SERVICEMANAGER_AS_A_SERVICE=false
OGG_SOFTWARE_HOME=/u01/app/oracle/product/ogg_mssql
OGG_DEPLOYMENT_HOME=/u01/app/oracle/deployments/Dep_MSSQL
ENV_MSSQL_ODBCINI=/etc/odbc.ini
ENV_MSSQL_ODBCINST=/etc
PORT_ADMINSRVR=9310
PORT_DISTSRVR=9311
PORT_RCVRSRVR=9312
PORT_PMSRVR=9313
UDP_PORT_PMSRVR=9314
METRICS_SERVER_ENABLED=true
OGG_SCHEMA=oggadmin
EOF
$OGG_HOME/bin/oggca.sh -silent -responseFile /tmp/oggca_mssql.rsp
```

Kiểm nhanh cả 4 deployment đã join Service Manager và đang chạy (`grep -o` thay vì `python3 -m json.tool` — OL8 minimal không có sẵn `python3`):

```bash
curl -s -u oggadmin:'Zxcasd123!@#' http://10.10.200.31:9001/services/v2/deployments \
  | grep -o '"name":"[^"]*","status":"[^"]*"'
ss -ltn | grep -E ':90[01][0-9]|:91[01][0-9]|:92[01][0-9]|:93[01][0-9]'
```

```
"name":"Dep_Hub","status":"running"
"name":"Dep_MSSQL","status":"running"
"name":"Dep_Maria","status":"running"
"name":"Dep_PG","status":"running"
"name":"ServiceManager","status":"running"
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/90.png"/>

#### B. Extract `EXT_FIN` capture NPRI (Dep_Hub 9010)

Param file trước, rồi adminclient:

```bash
cat > /u01/app/oracle/deployments/Dep_Hub/etc/conf/ogg/EXT_FIN.prm << 'EOF'
EXTRACT EXT_FIN
USERIDALIAS cred_opri
EXTTRAIL ea
TABLE FINACC.*;
EOF
export OGG_HOME=/u01/app/oracle/product/ogg_ora; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

> **Dọn capture kế thừa từ OPRI trước — nếu không sẽ dính `OGG-08241`.** `REGISTER EXTRACT` ở [2.2.1](#221--ogg-hub--extract-capture-oracle) ghi thông tin capture **vào trong database** (`DBA_CAPTURE`), không phải vào file config của OGG. NPRI là bản sao khối của OPRI nên nó thừa hưởng luôn đăng ký đó, trong khi `Dep_Hub` trên .31 thì trắng tinh — hai bên lệch nhau, `ADD EXTRACT` sẽ bị DB chặn với `OGG-08241: This Extract group EXT_FIN is already registered with the database`.

Kiểm rồi gỡ trên **.31** (user `oracle`):

```bash
sqlplus -s / as sysdba << 'EOF'
SET LINESIZE 200
COL capture_name FOR a20
SELECT capture_name, status, capture_user FROM dba_capture;
EOF
```

Ra `OGG$CAP_EXT_FIN | DISABLED | OGGADMIN` thì drop — API của Oracle, không gỡ được từ phía OGG vì `UNREGISTER EXTRACT` đòi group phải tồn tại ở local mà `Dep_Hub` mới thì chưa có:

```bash
sqlplus -s / as sysdba << 'EOF'
BEGIN
  DBMS_CAPTURE_ADM.DROP_CAPTURE('OGG$CAP_EXT_FIN', TRUE);
END;
/
SELECT capture_name FROM dba_capture;
EOF
```

Phải ra `no rows selected`. Đừng bỏ qua bước này rồi né bằng cách đặt tên Extract khác: capture dù `DISABLED` vẫn giữ một `required_checkpoint_scn`, Oracle sẽ **không xoá archive log** từ điểm đó trở đi để dành cho một Extract không bao giờ quay lại — FRA đầy dần mà không rõ vì sao.

Sạch rồi mới vào adminclient:

```sql
CONNECT http://10.10.200.31:9010 DEPLOYMENT Dep_Hub AS oggadmin PASSWORD Zxcasd123!@# !
ALTER CREDENTIALSTORE ADD USER oggadmin@10.10.200.31:1521/NPRI PASSWORD Zxcasd123!@# ALIAS cred_opri
DBLOGIN USERIDALIAS cred_opri

-- supplemental logging + SCHEMATRANDATA đã theo Data Guard sang NPRI, chỉ cần verify:
INFO SCHEMATRANDATA FINACC

ADD EXTRACT EXT_FIN, INTEGRATED TRANLOG, BEGIN NOW
REGISTER EXTRACT EXT_FIN DATABASE
ADD EXTTRAIL ea, EXTRACT EXT_FIN
START EXTRACT EXT_FIN
INFO EXTRACT EXT_FIN
```

#### C. 3 Distribution Path trên Dep_Hub

Vẫn trong adminclient `Dep_Hub` — 3 alias domain `Network` + 3 path `ea`→`eb|ec|ed` về Receiver 9112/9212/9312 (cả 4 deployment cùng nằm trên .31):

```sql
ALTER CREDENTIALSTORE ADD USER oggadmin ALIAS path_maria DOMAIN Network PASSWORD Zxcasd123!@#
ALTER CREDENTIALSTORE ADD USER oggadmin ALIAS path_pg DOMAIN Network PASSWORD Zxcasd123!@#
ALTER CREDENTIALSTORE ADD USER oggadmin ALIAS path_mssql DOMAIN Network PASSWORD Zxcasd123!@#

ADD DISTPATH p_ea_eb SOURCE trail://10.10.200.31:9011/services/v2/sources?trail=ea TARGET ws://10.10.200.31:9112/services/v2/targets?trail=eb AUTHENTICATION USERIDALIAS path_maria DOMAIN Network
ADD DISTPATH p_ea_ec SOURCE trail://10.10.200.31:9011/services/v2/sources?trail=ea TARGET ws://10.10.200.31:9212/services/v2/targets?trail=ec AUTHENTICATION USERIDALIAS path_pg DOMAIN Network
ADD DISTPATH p_ea_ed SOURCE trail://10.10.200.31:9011/services/v2/sources?trail=ea TARGET ws://10.10.200.31:9312/services/v2/targets?trail=ed AUTHENTICATION USERIDALIAS path_mssql DOMAIN Network

START DISTPATH p_ea_eb
START DISTPATH p_ea_ec
START DISTPATH p_ea_ed

INFO DISTPATH p_ea_eb
INFO DISTPATH p_ea_ec
INFO DISTPATH p_ea_ed
EXIT
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/91.png"/>

#### D. Dọn checkpoint table cũ trên 3 vệ tinh mới

Bảng `oggchkpt` **đi theo data lúc di trú** nhưng chứa checkpoint của deployment cũ — drop để tạo lại sạch:

```bash
export PATH=$PATH:/opt/mssql-tools18/bin

# MariaDB .31
sudo docker exec mariadb mariadb -uroot -p'Zxcasd123!@#' -e \
  "DROP TABLE IF EXISTS finacc.oggchkpt, finacc.oggchkpt_lox;"
# PostgreSQL .32
ssh root@10.10.200.32 "docker exec postgres psql -U postgres -d finacc -c \
  'DROP TABLE IF EXISTS public.oggchkpt, public.oggchkpt_lox;'"
# MSSQL .33
sqlcmd -S 10.10.200.33,1433 -U ggmssql -P 'Zxcasd123!@#' -d FINACC -C -Q \
  "DROP TABLE IF EXISTS dbo.oggchkpt, dbo.oggchkpt_lox;"
```

Kiểm lại: mỗi vệ tinh chỉ còn đúng **2 bảng data**, không còn bảng `oggchkpt*` nào:

```bash
# MariaDB .31
sudo docker exec mariadb mariadb -uroot -p'Zxcasd123!@#' -e "SHOW TABLES IN finacc;"
# PostgreSQL .32
ssh root@10.10.200.32 "docker exec postgres psql -U postgres -d finacc -c '\dt'"
# MSSQL .33
sqlcmd -S 10.10.200.33,1433 -U ggmssql -P 'Zxcasd123!@#' -d FINACC -C -Q \
  "SELECT name FROM sys.tables ORDER BY name;"
```

Cả 3 chỉ ra `accounts` và `transactions`. Còn sót `oggchkpt` thì Replicat ở mục E sẽ `ADD REPLICAT ... CHECKPOINTTABLE` vào bảng cũ mang checkpoint của hub đã bỏ — apply lệch vị trí trail ngay từ lần start đầu.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/92.png"/>

#### E. 3 Replicat trỏ vệ tinh mới

**KHÔNG cần Initial Load lần này:** vệ tinh mới đã đủ data từ [4.3](#43--di-trú-vệ-tinh-mariadb31-postgresql32-mssql33), NPRI chưa nhận ghi mới nào từ lúc switchover, Extract `BEGIN NOW` phủ từ hiện tại → không có khoảng hổng.

**E.1 — `RMAR` (Dep_Maria 9110, MariaDB cùng host .31):**

```bash
su - oracle
cat > /u01/app/oracle/deployments/Dep_Maria/etc/conf/ogg/RMAR.prm << 'EOF'
REPLICAT RMAR
USERIDALIAS cred_maria
MAP FINACC.ACCOUNTS,     TARGET finacc.accounts;
MAP FINACC.TRANSACTIONS, TARGET finacc.transactions;
EOF
export OGG_HOME=/u01/app/oracle/product/ogg_mysql; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.31:9110 DEPLOYMENT Dep_Maria AS oggadmin PASSWORD Zxcasd123!@# !
ALTER CREDENTIALSTORE ADD USER ggmysql@127.0.0.1:3306/finacc PASSWORD Zxcasd123!@# ALIAS cred_maria
DBLOGIN USERIDALIAS cred_maria
ADD CHECKPOINTTABLE finacc.oggchkpt
ADD REPLICAT RMAR, EXTTRAIL eb, CHECKPOINTTABLE finacc.oggchkpt
START REPLICAT RMAR
INFO REPLICAT RMAR
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/93.png"/>

**E.2 — `RPPG` (Dep_PG 9210, PostgreSQL remote .32):**

```bash
cat > /u01/app/oracle/deployments/Dep_PG/etc/conf/ogg/RPPG.prm << 'EOF'
REPLICAT RPPG
USERIDALIAS cred_pg
MAP FINACC.ACCOUNTS,     TARGET public.accounts;
MAP FINACC.TRANSACTIONS, TARGET public.transactions;
EOF
export OGG_HOME=/u01/app/oracle/product/ogg_pg; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.31:9210 DEPLOYMENT Dep_PG AS oggadmin PASSWORD Zxcasd123!@# !
ALTER CREDENTIALSTORE ADD USER ggpg@10.10.200.32:5432/finacc PASSWORD Zxcasd123!@# ALIAS cred_pg
DBLOGIN USERIDALIAS cred_pg
ADD CHECKPOINTTABLE public.oggchkpt
ADD REPLICAT RPPG, EXTTRAIL ec, CHECKPOINTTABLE public.oggchkpt
START REPLICAT RPPG
INFO REPLICAT RPPG
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/94.png"/>

**E.3 — `RMSSQL` (Dep_MSSQL 9310, MSSQL remote .33 qua DSN `mssql_finacc`):**

```bash
cat > /u01/app/oracle/deployments/Dep_MSSQL/etc/conf/ogg/RMSSQL.prm << 'EOF'
REPLICAT RMSSQL
USERIDALIAS cred_mssql
MAP FINACC.ACCOUNTS,     TARGET dbo.accounts;
MAP FINACC.TRANSACTIONS, TARGET dbo.transactions;
EOF
export OGG_HOME=/u01/app/oracle/product/ogg_mssql; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```sql
CONNECT http://10.10.200.31:9310 DEPLOYMENT Dep_MSSQL AS oggadmin PASSWORD Zxcasd123!@# !
ALTER CREDENTIALSTORE ADD USER ggmssql@mssql_finacc PASSWORD Zxcasd123!@# ALIAS cred_mssql
DBLOGIN USERIDALIAS cred_mssql
ADD CHECKPOINTTABLE dbo.oggchkpt
ADD REPLICAT RMSSQL, EXTTRAIL ed, CHECKPOINTTABLE dbo.oggchkpt
START REPLICAT RMSSQL
INFO REPLICAT RMSSQL
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/95.png"/>

#### F. Mở ghi lại (downtime kết thúc) + test xuyên hệ mới

```bash
# .31 — insert vào NPRI
export ORACLE_SID=NPRI
sqlplus -s / as sysdba << 'EOF'
INSERT INTO FINACC.TRANSACTIONS (acct_id, amount, txn_type, created) VALUES (1, 777, 'MIG', SYSDATE);
COMMIT;
EOF

# soi 3 vệ tinh mới — cả 2 bảng
sudo docker exec mariadb mariadb -uroot -p'Zxcasd123!@#' -e \
  "SELECT * FROM finacc.accounts; SELECT * FROM finacc.transactions;"

ssh root@10.10.200.32 "docker exec postgres psql -U postgres -d finacc \
  -c 'SELECT * FROM accounts;' -c 'SELECT * FROM transactions;'"

sqlcmd -S 10.10.200.33,1433 -U ggmssql -P 'Zxcasd123!@#' -d FINACC -C -Q \
  "SELECT * FROM dbo.accounts; SELECT * FROM dbo.transactions;"
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/96.png"/>

#### G. Veridata mới trên .32

**G.1 — Tắt Veridata cũ (.22) trước.** RCU schema `LABVD*` nằm **trong DB Oracle** — đã theo Data Guard sang NPRI, hai domain không được giành chung schema:

```bash
# .22, user oracle
su - oracle
export JAVA_HOME=/usr/java/default
DOMAIN=/u01/app/oracle/config/domains/veridata_domain
$DOMAIN/bin/stopManagedWebLogic.sh VERIDATA_server1 t3://10.10.200.22:7001 weblogic 'Zxcasd123!@#'
$DOMAIN/bin/stopWebLogic.sh weblogic 'Zxcasd123!@#' t3://10.10.200.22:7001
/u01/app/oracle/vdagent_ora/agent.sh stop
/u01/app/oracle/vdagent_mssql/agent.sh stop
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/97.png"/>

**G.2 — Cài nền trên .32** (root cài JDK, oracle cài FMW + Veridata — OUI từ chối chạy bằng root):

```bash
dnf install -y /u01/soft/*jdk-8u491*.rpm        # bằng root
su - oracle
cat > /tmp/oraInst.loc << 'EOF'
inventory_loc=/u01/app/oraInventory
inst_group=oinstall
EOF

# FMW Infrastructure
cat > /tmp/fmw.rsp << 'EOF'
[ENGINE]
Response File Version=1.0.0.0.0
[GENERIC]
ORACLE_HOME=/u01/app/oracle/product/fmw12c
INSTALL_TYPE=Fusion Middleware Infrastructure
DECLINE_AUTO_UPDATES=true
EOF
cd /u01/soft && unzip -o -q *Infrastructure*.zip
java -jar /u01/soft/fmw_12.2.1.4.0_infrastructure.jar -silent -responseFile /tmp/fmw.rsp -invPtrLoc /tmp/oraInst.loc

# Veridata Web Server
cd /u01/soft && unzip -o -q *Veridata*.zip
cat > /tmp/veridata.rsp << 'EOF'
[ENGINE]
Response File Version=1.0.0.0.0
[GENERIC]
ORACLE_HOME=/u01/app/oracle/product/fmw12c
INSTALL_TYPE=GoldenGate Veridata Web Server Installation
DECLINE_AUTO_UPDATES=true
EOF
java -jar /u01/soft/fmw_12.2.1.4.0_ogg.jar -silent -responseFile /tmp/veridata.rsp -invPtrLoc /tmp/oraInst.loc
```

**G.3 — RCU prefix mới `LABVD2`.** Schema `LABVD*` cũ đã theo Data Guard sang NPRI và về lý thuyết dùng lại được — nhưng **không phải tất cả**: riêng `LABVD_OPSS` đang giữ security store của domain trên .22, và OPSS **không cho 2 domain dùng chung một OPSS schema** kể cả khi domain cũ đã tắt (dấu vết nằm trong dữ liệu schema, không phải ở tiến trình). Cố dùng lại sẽ chết ở phase "OPSS Processing":

```
The schema LABVD_OPSS is already in use for security store(s). Please create a new schema.
```

Nên chạy RCU lần nữa với prefix `LABVD2`, lần này trỏ vào **NPRI (.31)**. Không mất gì: repo Veridata mới rỗng, mà **G.6** bên dưới vốn đã tạo lại connection + compare pair + job từ đầu:

```bash
{ echo 'Zxcasd123!@#'; for i in $(seq 10); do echo 'Zxcasd123#'; done; } | \
/u01/app/oracle/product/fmw12c/oracle_common/bin/rcu -silent -createRepository \
  -databaseType ORACLE -connectString 10.10.200.31:1521:NPRI \
  -dbUser sys -dbRole sysdba -schemaPrefix LABVD2 \
  -component STB -component OPSS -component IAU -component IAU_APPEND \
  -component IAU_VIEWER -component VERIDATA -f
```

Cả 7 component phải `Success`. Kiểm từ .32 — nhớ `/nolog` + `CONNECT` nháy kép, vì mật khẩu chứa dấu `@`, viết thẳng `sys/'...'@NPRI` thì sqlplus cắt ở `@` đầu tiên và báo `ORA-12154`:

```bash
sqlplus -s /nolog << 'EOF'
CONNECT sys/"Zxcasd123!@#"@//10.10.200.31:1521/NPRI AS SYSDBA
SELECT username FROM dba_users WHERE username LIKE 'LABVD2%' ORDER BY 1;
EOF
```

Ra 8 dòng `LABVD2_*` (6 component + `WLS`/`WLS_RUNTIME`).

**G.4 — Domain WLST trên .32** — script y hệt [3.1](#31--cài-veridata-22--2-agent-oracle-sql-server) B, đổi 3 chỗ: datasource URL trỏ **NPRI (.31)**, schema `LABVD2_STB`, ListenAddress → **10.10.200.32**.

Thêm một khác biệt nữa so với 3.1: **vòng lặp ghi đè URL cho toàn bộ datasource**. `getDatabaseDefaults()` không lấy URL từ dòng `set('URL',...)` phía trên — nó đọc **service table** trong schema `*_STB` rồi tự điền cho 5 datasource còn lại (`opss-data-source`, `opss-audit-DBDS`, `opss-audit-viewDS`, `WLSSchemaDataSource`, `VeridataDataSource`). Bỏ vòng lặp thì chúng giữ nguyên địa chỉ RCU đã ghi, và nếu bạn lỡ dùng lại prefix cũ `LABVD` thì đó là `//10.10.200.21:1521/OPRI` — OPSS quay về hỏi .21 vốn đang là standby MOUNT và chết với `ORA-01033`:

```
Can not connect DB with URL jdbc:oracle:thin:@//10.10.200.21:1521/OPRI
java.sql.SQLRecoverableException: ORA-01033: ORACLE initialization or shutdown in progress
```

Vòng lặp phải giữ đúng thụt lề 4 space — Jython không tha lỗi indent, mà terminal paste hay nuốt khoảng trắng. Ghi file xong kiểm bằng `sed -n '17,22p' /tmp/veridata_domain.py` trước khi chạy.

```bash
cat > /tmp/veridata_domain.py << 'EOF'
oh = '/u01/app/oracle/product/fmw12c'
readTemplate(oh + '/wlserver/common/templates/wls/wls.jar')
addTemplate(oh + '/oracle_common/common/templates/wls/oracle.jrf_template.jar')
addTemplate(oh + '/veridata/common/templates/wls/veridata_web_template.jar')
setOption('DomainName', 'veridata_domain')
setOption('ServerStartMode', 'prod')
setOption('JavaHome', '/usr/java/default')
cd('/Security/base_domain/User/weblogic')
cmo.setPassword('Zxcasd123!@#')
cd('/JDBCSystemResource/LocalSvcTblDataSource/JdbcResource/LocalSvcTblDataSource/JDBCDriverParams/NO_NAME_0')
set('DriverName','oracle.jdbc.OracleDriver')
set('URL','jdbc:oracle:thin:@//10.10.200.31:1521/NPRI')
set('PasswordEncrypted','Zxcasd123#')
cd('Properties/NO_NAME_0/Property/user')
set('Value','LABVD2_STB')
cd('/'); getDatabaseDefaults()
cd('/JDBCSystemResource')
for ds in ls(returnMap='true'):
    cd('/JDBCSystemResource/' + ds + '/JdbcResource/' + ds + '/JDBCDriverParams/NO_NAME_0')
    set('URL','jdbc:oracle:thin:@//10.10.200.31:1521/NPRI')
    cd('/JDBCSystemResource')
cd('/')
cd('/Servers/AdminServer'); set('ListenAddress','10.10.200.32'); set('ListenPort',7001)
cd('/Servers/VERIDATA_server1'); set('ListenAddress','10.10.200.32'); set('ListenPort',8830)
writeDomain('/u01/app/oracle/config/domains/veridata_domain')
closeTemplate(); exit()
EOF
/u01/app/oracle/product/fmw12c/oracle_common/common/bin/wlst.sh /tmp/veridata_domain.py

# boot.properties + start (AdminServer trước, đợi RUNNING mới start managed server)
DOMAIN=/u01/app/oracle/config/domains/veridata_domain
for S in AdminServer VERIDATA_server1; do
  mkdir -p $DOMAIN/servers/$S/security
  printf 'username=weblogic\npassword=Zxcasd123!@#\n' > $DOMAIN/servers/$S/security/boot.properties
done
nohup $DOMAIN/bin/startWebLogic.sh > /tmp/adminserver.log 2>&1 &
until grep -q 'Server state changed to RUNNING' /tmp/adminserver.log; do sleep 5; done
nohup $DOMAIN/bin/startManagedWebLogic.sh VERIDATA_server1 t3://10.10.200.32:7001 > /tmp/veridata.log 2>&1 &

# Cấp quyền: map role Administrator -> group veridataAdministrator
cat > /tmp/vd_grant.py << 'EOF'
connect('weblogic','Zxcasd123!@#','t3://10.10.200.32:7001')
atn = cmo.getSecurityConfiguration().getDefaultRealm().lookupAuthenticationProvider('DefaultAuthenticator')
if not atn.groupExists('veridataAdministrator'):
    atn.createGroup('veridataAdministrator', 'Veridata Administrator role')
atn.addMemberToGroup('veridataAdministrator', 'weblogic')
disconnect(); exit()
EOF
/u01/app/oracle/product/fmw12c/oracle_common/common/bin/wlst.sh /tmp/vd_grant.py
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/98.png"/>

**G.5 — Cài binary Agent + 2 agent trên .32** (agent Oracle 7850 → NPRI .31, agent SQL Server 7852 → MSSQL .33):

```bash
cat > /tmp/vdagent.rsp << 'EOF'
[ENGINE]
Response File Version=1.0.0.0.0
[GENERIC]
ORACLE_HOME=/u01/app/oracle/product/vdagent
INSTALL_TYPE=GoldenGate Veridata Agent Installation
DECLINE_AUTO_UPDATES=true
EOF
java -jar /u01/soft/fmw_12.2.1.4.0_ogg.jar -silent -responseFile /tmp/vdagent.rsp -invPtrLoc /tmp/oraInst.loc

export JAVA_HOME=/usr/java/default
A=/u01/app/oracle/product/vdagent/veridata/agent

# Agent Oracle (7850) -> NPRI .31, driver ojdbc8 từ Oracle home local (.32 đã cài 19c ở 4.1)
ODEP=/u01/app/oracle/vdagent_ora
$A/agent_config.sh $ODEP
cp /u01/app/oracle/product/19.3.0/dbhome_1/jdbc/lib/ojdbc8.jar $ODEP/drivers/
cat > $ODEP/agent.properties << 'EOF'
server.port=7850
database.url=jdbc:oracle:thin:@10.10.200.31:1521/NPRI
server.jdbcDriver=ojdbc8.jar
server.driversLocation=drivers
server.useSsl=false
EOF
$ODEP/agent.sh start

# Agent SQL Server (7852) -> MSSQL .33, driver DataDirect bundle sẵn trong agent
SDEP=/u01/app/oracle/vdagent_mssql
$A/agent_config.sh $SDEP
cp /u01/app/oracle/product/vdagent/oracle_common/modules/datadirect/wlsqlserver.jar $SDEP/drivers/
cat > $SDEP/agent.properties << 'EOF'
server.port=7852
database.url=jdbc:weblogic:sqlserver://10.10.200.33:1433;databaseName=FINACC
server.jdbcDriver=wlsqlserver.jar
server.driversLocation=drivers
server.useSsl=false
server.use2WaySsl=false
server.allowTrustedExpiredCertificates=true
server.identitystore.path=./config/certs/serverIdentity.jks
server.truststore.path=./config/certs/serverTrust.jks
server.identitystore.type=JKS
server.truststore.type=JKS
server.identitystore.keyfactory.alg.name=SunX509
server.truststore.keyfactory.alg.name=SunX509
server.ssl.algorithm.name=TLS
EOF
$SDEP/agent.sh start
sudo ss -tlnp | grep -E '785[02]'
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/99.png"/>

**G.6 — Console + job.** `http://10.10.200.32:8830/veridata` (`weblogic`/`Zxcasd123!@#`), cấu hình y hệt [3.2](#32--veridata-verify-oracle--mssql):

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/100.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/101.png"/>

| Connection | Agent host:port | Datasource Type | User / pass |
|---|---|---|---|
| `conn_ora` | `10.10.200.32:7850` | `Oracle` | `oggadmin` / `Zxcasd123!@#` |
| `conn_mssql` | `10.10.200.32:7852` | `SQL Server` | `ggmssql` / `Zxcasd123!@#` |

Group `grp_ora_mssql` (source `conn_ora`, target `conn_mssql`) → Compare Pair Manual Mapping: Source Schema `FINACC`; Target Catalog `FINACC`, Schema `dbo` → generate 2 cặp `ACCOUNTS`/`TRANSACTIONS` → Job `job_ora_mssql` → **Run**.

**Kết quả mong đợi:** record `MIG` xuất hiện ở cả 3 vệ tinh mới trong vài giây; `INFO ALL` trên 4 deployment → Extract + 3 Replicat `RUNNING`, `INFO DISTPATH` → 3 path `RUNNING`; Veridata .32 chạy `job_ora_mssql` → **0 out-of-sync** (đã tính cả record `MIG`).

### 4.5 — Bỏ .21/.22/.23

**Mục tiêu:** xác nhận bộ mới chạy đủ **trước** khi tắt bộ cũ, rồi mới gỡ. 4 kiểm tra — chỉ cần một cái trượt là dừng lại, đừng tắt gì cả. Bộ cũ còn sống là còn đường lùi; tắt rồi thì không.

**1. Data Guard** — trên `.31`:

```bash
dgmgrl << 'EOF'
CONNECT sys/"Zxcasd123!@#"@NPRI
SHOW CONFIGURATION LAG;
EOF
```

`npri - Primary database`, `nstb` lag `0 seconds`, `Configuration Status: SUCCESS`.

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/102.png"/>

**2. GoldenGate** — trên `.31`, `INFO ALL` từng deployment. Mỗi deployment một port khác nhau nên phải connect 4 lần:

```bash
export OGG_HOME=/u01/app/oracle/product/ogg_ora; export PATH=$OGG_HOME/bin:$PATH
adminclient
```

```
CONNECT http://10.10.200.31:9010 DEPLOYMENT Dep_Hub AS oggadmin PASSWORD Zxcasd123!@# !
INFO ALL
INFO DISTPATH p_ea_eb
INFO DISTPATH p_ea_ec
INFO DISTPATH p_ea_ed
CONNECT http://10.10.200.31:9110 DEPLOYMENT Dep_Maria AS oggadmin PASSWORD Zxcasd123!@# !
INFO ALL
CONNECT http://10.10.200.31:9210 DEPLOYMENT Dep_PG AS oggadmin PASSWORD Zxcasd123!@# !
INFO ALL
CONNECT http://10.10.200.31:9310 DEPLOYMENT Dep_MSSQL AS oggadmin PASSWORD Zxcasd123!@# !
INFO ALL
```

Tổng cộng **7 process phải `RUNNING`**:

| Deployment | Lệnh | Kỳ vọng |
|---|---|---|
| `Dep_Hub` | `INFO ALL` | `EXTRACT RUNNING EXT_FIN INTEGRATED` |
| `Dep_Hub` | `INFO DISTPATH p_ea_eb` / `ec` / `ed` | 3 path đều `RUNNING` |
| `Dep_Maria` | `INFO ALL` | `REPLICAT RUNNING RMAR` |
| `Dep_PG` | `INFO ALL` | `REPLICAT RUNNING RPPG` |
| `Dep_MSSQL` | `INFO ALL` | `REPLICAT RUNNING RMSSQL` |

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/103.png"/>

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/104.png"/>

**3. Veridata** — vào `http://10.10.200.32:8830/veridata`, chạy lại `job_ora_mssql`, xem **Finished Jobs → Report**: cả 2 cặp `tables in sync`, `Rows out-of-sync: 0`.

**4. Dữ liệu 3 vệ tinh** — trên `.31`:

```bash
export PATH=$PATH:/opt/mssql-tools18/bin

# nguồn Oracle NPRI để có số đối chiếu
export ORACLE_SID=NPRI
sqlplus -s / as sysdba << 'EOF'
SELECT COUNT(*) AS acc FROM FINACC.ACCOUNTS;
SELECT COUNT(*) AS txn FROM FINACC.TRANSACTIONS;
EOF

sudo docker exec mariadb mariadb -uroot -p'Zxcasd123!@#' -e \
  "SELECT COUNT(*) acc FROM finacc.accounts; SELECT COUNT(*) txn FROM finacc.transactions;
   SELECT * FROM finacc.transactions WHERE txn_type='MIG';"

ssh root@10.10.200.32 "docker exec postgres psql -U postgres -d finacc \
  -c 'SELECT COUNT(*) acc FROM accounts;' -c 'SELECT COUNT(*) txn FROM transactions;' \
  -c \"SELECT * FROM transactions WHERE txn_type='MIG';\""

sqlcmd -S 10.10.200.33,1433 -U ggmssql -P 'Zxcasd123!@#' -d FINACC -C -Q \
  "SELECT COUNT(*) acc FROM dbo.accounts; SELECT COUNT(*) txn FROM dbo.transactions;
   SELECT * FROM dbo.transactions WHERE txn_type='MIG';"
```

Cả 3 vệ tinh phải khớp số dòng với NPRI **và** có record `MIG` — đây là bằng chứng CDC của hub mới đang chạy thật, không phải chỉ còn data cũ từ lúc di trú ở [4.3](#43--di-trú-vệ-tinh-mariadb31-postgresql32-mssql33).

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/105.png"/>

Đủ 4 mục mới gỡ node cũ khỏi Data Guard (chạy từ .31):

```bash
dgmgrl << 'EOF'
CONNECT sys/"Zxcasd123!@#"@NPRI
REMOVE DATABASE OPRI;
REMOVE DATABASE OSTBA;
SHOW CONFIGURATION LAG;
EOF
```

<img src="/assets/img/2026-07-21-he-db-hon-hop-oracle-mariadb-postgresql-mssql/106.png"/>

Tắt dần bộ cũ:

```bash
# .21: OGG hub cũ + MariaDB cũ + Oracle OPRI
sudo systemctl stop OracleGoldenGate    # ServiceManager cũ (tên service theo registerServiceManager)
docker compose -f /opt/mariadb/docker-compose.yml down
sqlplus / as sysdba <<< "SHUTDOWN IMMEDIATE;" && lsnrctl stop

# .22: Veridata cũ (đã stop ở 4.4) + PostgreSQL cũ + Oracle OSTBA
docker compose -f /opt/pg/docker-compose.yml down
sqlplus / as sysdba <<< "SHUTDOWN IMMEDIATE;" && lsnrctl stop
```

Trên **.23** (PowerShell admin): `Stop-Service MSSQLSERVER` rồi shutdown VM.

**Kết quả mong đợi:** toàn bộ "production" nằm ở .31/.32/.33 (cấu hình cuối: NPRI↔NSTB Data Guard, hub OGG .31 → 3 vệ tinh mới, Veridata .32); .21/.22/.23 tắt hẳn — di trú hoàn tất, không mất dữ liệu.

---

## Kết luận

Lab đã dựng một **data platform Oracle-centric thu nhỏ** mô phỏng gần đủ luồng dữ liệu doanh nghiệp lớn:

- **Đồng bộ dự phòng** — Oracle Data Guard (physical standby, lag 0) làm HA cho DB lõi.
- **Đồng bộ phục vụ khai thác** — GoldenGate Hub tập trung đẩy CDC từ Oracle ra **3 họ DB khác nhau** (MariaDB, PostgreSQL, MS SQL Server) qua một điểm duy nhất, biến chúng thành hệ vệ tinh cho report/portal/analytics.
- **Đối soát** — Veridata so từng row **Oracle ↔ MS SQL Server** ra 0 lệch (cặp Veridata 12.2.1.4 hỗ trợ); MariaDB/PostgreSQL đối soát thủ công bằng SQL.
- **Di trú** — chuyển toàn hệ sang bộ VM mới bằng Data Guard switchover + native replication/backup-restore cho vệ tinh, downtime tối thiểu.

**Bài học rút ra:**
- **Veridata chỉ hỗ trợ một số DB** (Oracle, SQL Server, DB2, Sybase, Informix, Teradata, NSK, Hive) — **KHÔNG MySQL/PostgreSQL**. Chọn cặp đối soát tự động theo đúng ma trận này (Oracle↔SQL Server là chuẩn).
- **SQL Server phải Developer/Standard/Enterprise** — Express bị OGG từ chối (`OGG-05311`).
- **OGG for SQL Server chạy trên Linux** deliver remote qua Microsoft ODBC — giữ được mô hình hub tập trung, không cần cài OGG lên Windows.
- **Mô hình OGG Hub** (một điểm chạy mọi Extract/Replicat) là cách production để quản lý replication đa nền.
