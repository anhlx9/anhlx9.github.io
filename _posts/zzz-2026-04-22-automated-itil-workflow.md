---
title: "Automated ITIL Workflow: Zabbix 7.2 + ServiceDesk Plus + Grafana + Docmost"
categories:
- Monitoring
- ITIL
- Docker

feature_image: "../assets/postbanner.jpg"
feature_text: |
  ### Lab Automated ITIL Workflow: Zabbix phát hiện sự cố → tự động tạo ticket SDP → Grafana dashboard → Docmost báo cáo
---

Bài viết trình bày thiết kế và triển khai lab **Automated ITIL Workflow** hoàn chỉnh chạy trên 2 VM ESXi. Toàn bộ stack (Zabbix 7.2, Grafana, Docmost, ServiceDesk Plus, PostgreSQL 16) chạy trên **Docker Compose** tại một VM duy nhất. Khi xảy ra sự cố trên host được giám sát, hệ thống tự động:

1. Zabbix phát hiện alert (disk full, host down)
2. Webhook tạo ticket trên **ServiceDesk Plus** và gắn cho kỹ thuật viên theo **round-robin**
3. **Grafana** hiển thị realtime metrics và trạng thái ticket
4. **Docmost** tự động cập nhật báo cáo sự cố

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Kiến trúc lab](#2-kiến-trúc-lab)
  - [2.1. Topology](#21-topology)
  - [2.2. Kế hoạch IP và phần mềm](#22-kế-hoạch-ip-và-phần-mềm)
  - [2.3. Luồng Automated ITIL Workflow](#23-luồng-automated-itil-workflow)
- [3. Chuẩn bị VM itil-stack](#3-chuẩn-bị-vm-itil-stack)
  - [3.1. Cài Docker](#31-cài-docker)
  - [3.2. Cấu trúc thư mục dự án](#32-cấu-trúc-thư-mục-dự-án)
  - [3.3. docker-compose.yml](#33-docker-composeyml)
- [4. Setup PostgreSQL 16](#4-setup-postgresql-16)
- [5. Setup Zabbix 7.2 + Grafana](#5-setup-zabbix-72--grafana)
  - [5.1. Tạo database Zabbix trong PostgreSQL](#51-tạo-database-zabbix-trong-postgresql)
  - [5.2. Khởi động Zabbix Server + Web + Agent](#52-khởi-động-zabbix-server--web--agent)
  - [5.3. Cấu hình Zabbix Web lần đầu](#53-cấu-hình-zabbix-web-lần-đầu)
  - [5.4. Cài Zabbix Agent trên Windows Server 2022](#54-cài-zabbix-agent-trên-windows-server-2022)
  - [5.5. Add hosts vào Zabbix](#55-add-hosts-vào-zabbix)
  - [5.6. Khởi động Grafana](#56-khởi-động-grafana)
  - [5.7. Thêm Zabbix datasource](#57-thêm-zabbix-datasource)
  - [5.8. Import Zabbix Dashboards](#58-import-zabbix-dashboards)
- [6. Setup ServiceDesk Plus 14](#6-setup-servicedesk-plus-14)
  - [6.1. Cài đặt SDP trực tiếp trên Ubuntu](#61-cài-đặt-sdp-trực-tiếp-trên-ubuntu)
  - [6.2. Đăng nhập lần đầu vào SDP](#62-đăng-nhập-lần-đầu-vào-sdp)
  - [6.3. Tạo 3 kỹ thuật viên](#63-tạo-3-kỹ-thuật-viên)
  - [6.4. Cấu hình Round-Robin Auto Assign](#64-cấu-hình-round-robin-auto-assign)
  - [6.5. Lấy API Token — cập nhật macro Zabbix](#65-lấy-api-token--cập-nhật-macro-zabbix)
  - [6.6. Tạo Webhook Media Type gọi SDP API](#66-tạo-webhook-media-type-gọi-sdp-api)
  - [6.7. Tạo Action tự động tạo ticket](#67-tạo-action-tự-động-tạo-ticket)
  - [6.8. Thêm SDP datasource Grafana](#68-thêm-sdp-datasource-grafana)
  - [6.9. Tạo dashboard ITIL Overview](#69-tạo-dashboard-itil-overview)
- [7. Setup Confluence 9](#7-setup-confluence-9)
  - [7.1. Tạo database Confluence trong PostgreSQL](#71-tạo-database-confluence-trong-postgresql)
  - [7.2. Khởi động Confluence](#72-khởi-động-confluence)
  - [7.3. Setup wizard + trial license](#73-setup-wizard--trial-license)
  - [7.4. Tạo Space và Page báo cáo](#74-tạo-space-và-page-báo-cáo)
  - [7.5. Script cập nhật Confluence qua API](#75-script-cập-nhật-confluence-qua-api)
  - [7.6. Tích hợp vào Zabbix Action](#76-tích-hợp-vào-zabbix-action)
- [8. Test Automated Workflow](#8-test-automated-workflow)
  - [8.1. Test: Disk Full](#81-test-disk-full)
  - [8.2. Test: Host Down](#82-test-host-down)
- [9. Kết quả](#9-kết-quả)

---

### 1. Giới thiệu

**ITIL (Information Technology Infrastructure Library)** là bộ khung thực hành quản lý dịch vụ CNTT, trong đó **Incident Management** là quy trình quan trọng nhất — đảm bảo mọi sự cố được phát hiện, ghi nhận, phân công và xử lý đúng hạn.

Bài lab này tự động hóa toàn bộ quy trình:

| Thành phần | Vai trò |
|---|---|
| **Zabbix 7.2** | Giám sát hạ tầng, phát hiện sự cố, gửi alert |
| **ServiceDesk Plus 14** | ITSM platform, quản lý ticket, SLA, assignment |
| **Grafana** | Dashboard tổng hợp metrics Zabbix + SDP tickets |
| **Docmost** | Knowledge base, tự động cập nhật incident report |
| **PostgreSQL 16** | Backend database dùng chung cho Zabbix và Confluence |

**Phần mềm sử dụng:**

- Zabbix 7.2 (open source)
- Grafana latest với plugin `alexanderzobnin-zabbix-app`
- Docmost (open source, self-hosted — [docmost.com](https://docmost.com))
- ManageEngine ServiceDesk Plus 14 (trial 30 ngày — [đăng ký tại đây](https://sdpondemand.manageengine.com/Register.do?opDownload&deployment=Onpremises&pos=Downloads&loc=servicemgmt&prev=AB3))
- PostgreSQL 16

---

### 2. Kiến trúc lab

#### 2.1. Topology


```
┌────────────────────────────────────────────────────────────────────┐
│                    VMware ESXi — Lab-vlan200 (10.10.200.0/24)      │
│                                                                    │
│  ┌──────────────────────────────────────────┐                      │
│  │  itil-stack  (10.10.200.11)              │                      │
│  │  Ubuntu Server 24.04 — 16 vCPU, 24GB RAM │                      │
│  │  100GB Disk                               │                      │
│  │                                           │                      │
│  │  ┌────────────┐  ┌──────────────────────┐│                      │
│  │  │ Zabbix 7.2 │  │ ServiceDesk Plus 14  ││                      │
│  │  │ :8080      │  │ :8400                ││                      │
│  │  └────────────┘  └──────────────────────┘│                      │
│  │  ┌────────────┐  ┌──────────────────────┐│                      │
│  │  │  Grafana   │  │  Docmost             ││                      │
│  │  │  :3000     │  │  :8090               ││                      │
│  │  └────────────┘  └──────────────────────┘│                      │
│  │  ┌────────────────────────────────────┐  │                      │
│  │  │         PostgreSQL 16  :5432       │  │                      │
│  │  └────────────────────────────────────┘  │                      │
│  └──────────────────────────────────────────┘                      │
│                          │ Zabbix Agent                            │
│               ┌──────────┴──────────────┐                         │
│               ▼                         ▼                          │
│  ┌─────────────────────┐   ┌───────────────────────────────┐      │
│  │ win2022             │   │ itil-stack (self-monitoring)  │      │
│  │ 10.10.200.12        │   │ Zabbix Agent container        │      │
│  │ Windows Server 2022 │   │ Host: itil-stack              │      │
│  └─────────────────────┘   └───────────────────────────────┘      │
└────────────────────────────────────────────────────────────────────┘
```

#### 2.2. Kế hoạch IP và phần mềm

| VM | Hostname | IP | OS | Vai trò |
|---|---|---|---|---|
| Stack | `itil-stack` | `10.10.200.11` | Ubuntu Server 24.04 | Chạy toàn bộ Docker Compose stack |
| Monitor | `win2022` | `10.10.200.12` | Windows Server 2022 | Host được giám sát |

**Port mapping trên itil-stack:**

| Service | Port | URL |
|---|---|---|
| PostgreSQL | 5432 | TCP (internal + expose để quản lý) |
| Zabbix Web | 8080 | `http://10.10.200.11:8080` |
| Zabbix Server | 10051 | TCP |
| Grafana | 3000 | `http://10.10.200.11:3000` |
| Docmost | 8090 | `http://10.10.200.11:8090` |
| ServiceDesk Plus | 8400 | `http://10.10.200.11:8400` |

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
            ┌─────────────────┼──────────────────┐
            ▼                 ▼                  ▼
  ┌─────────────────┐  ┌────────────┐  ┌────────────────┐
  │  Webhook → SDP  │  │  Grafana   │  │  Script →      │
  │  Tạo ticket     │  │  Dashboard │  │  Docmost       │
  │  Round-robin    │  │  realtime  │  │  Update Page   │
  │  Assign         │  │  update    │  │  (API)         │
  └────────┬────────┘  └────────────┘  └────────────────┘
           │
  ┌────────▼────────────────────────────┐
  │  KTV được assign (round-robin):      │
  │  Ticket 1 → Nguyen Van A            │
  │  Ticket 2 → Tran Thi B             │
  │  Ticket 3 → Le Van C               │
  │  Ticket 4 → Nguyen Van A ...        │
  └─────────────────────────────────────┘
```

---

### 3. Chuẩn bị VM itil-stack

#### 3.1. Cài Docker

Trên VM `itil-stack` (Ubuntu Server 24.04):

```bash
# Cập nhật hệ thống
sudo apt update && sudo apt upgrade -y

# Cài Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker

# Cài Docker Compose plugin
sudo apt install -y docker-compose-plugin

# Kiểm tra
docker version
docker compose version
```

<img src="../assets/img/2026-04-22-automated-itil-workflow/01.png"/>

#### 3.2. Cấu trúc thư mục dự án

```bash
sudo mkdir -p /opt/itil-stack/zabbix/alertscripts
sudo chown -R $USER:$USER /opt/itil-stack
cd /opt/itil-stack
```

```
/opt/itil-stack/
├── docker-compose.yml
└── zabbix/
    └── alertscripts/
        └── docmost_update.py   # Script cập nhật Docmost (tạo ở mục 7)
```

#### 3.3. docker-compose.yml

Tạo file `docker-compose.yml` chứa toàn bộ service. Các service sẽ được khởi động **từng bước theo từng mục** bên dưới:

```bash
cat > /opt/itil-stack/docker-compose.yml << 'EOF'
volumes:
  postgres_data:
  zabbix_server_data:
  grafana_data:
  confluence_data:

services:

  # ─── PostgreSQL 16 ──────────────────────────────────────────────────
  postgres:
    image: postgres:16
    container_name: itil-postgres
    restart: unless-stopped
    network_mode: host
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres_root_2024
      POSTGRES_DB: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -h 10.10.200.11"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ─── Zabbix Server ──────────────────────────────────────────────────
  zabbix-server:
    image: zabbix/zabbix-server-pgsql:7.2-ubuntu-latest
    container_name: itil-zabbix-server
    restart: unless-stopped
    network_mode: host
    environment:
      DB_SERVER_HOST: 10.10.200.11
      DB_SERVER_PORT: 5432
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pass_2024
      POSTGRES_DB: zabbix
      ZBX_CACHESIZE: 128M
      ZBX_STARTPOLLERS: 5
    volumes:
      - zabbix_server_data:/var/lib/zabbix
      - ./zabbix/alertscripts:/usr/lib/zabbix/alertscripts
    depends_on:
      postgres:
        condition: service_healthy

  # ─── Zabbix Web UI ──────────────────────────────────────────────────
  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:7.2-ubuntu-latest
    container_name: itil-zabbix-web
    restart: unless-stopped
    network_mode: host
    environment:
      ZBX_SERVER_HOST: 10.10.200.11
      ZBX_SERVER_PORT: 10051
      DB_SERVER_HOST: 10.10.200.11
      DB_SERVER_PORT: 5432
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pass_2024
      POSTGRES_DB: zabbix
      PHP_TZ: Asia/Ho_Chi_Minh
      ZBX_LISTEN_PORT: 8080
    depends_on:
      - zabbix-server

  # ─── Zabbix Agent 2 ─────────────────────────────────────────────────
  zabbix-agent:
    image: zabbix/zabbix-agent2:7.2-ubuntu-latest
    container_name: itil-zabbix-agent
    restart: unless-stopped
    network_mode: host
    privileged: true
    environment:
      ZBX_HOSTNAME: itil-stack
      ZBX_SERVER_HOST: 10.10.200.11
      ZBX_SERVER_PORT: 10051
      ZBX_PASSIVE_ALLOW: "true"
      ZBX_LISTENPORT: 10050
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /dev:/host/dev:ro

  # ─── Grafana ────────────────────────────────────────────────────────
  grafana:
    image: grafana/grafana:latest
    container_name: itil-grafana
    restart: unless-stopped
    network_mode: host
    environment:
      GF_SERVER_HTTP_PORT: 3000
      GF_SERVER_HTTP_ADDR: 10.10.200.11
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: Admin@2024
      GF_INSTALL_PLUGINS: alexanderzobnin-zabbix-app,alexanderzobnin-zabbix-triggers-panel
      GF_PLUGINS_ALLOW_LOADING_UNSIGNED_PLUGINS: alexanderzobnin-zabbix-app
    volumes:
      - grafana_data:/var/lib/grafana

  # ─── Confluence 9 ───────────────────────────────────────────────────
  confluence:
    image: atlassian/confluence:9
    container_name: itil-confluence
    restart: unless-stopped
    network_mode: host
    environment:
      ATL_JDBC_URL: jdbc:postgresql://10.10.200.11:5432/confluence
      ATL_JDBC_USER: confluence
      ATL_JDBC_PASSWORD: confluence_pass_2024
      ATL_DB_TYPE: postgresql
      JVM_MINIMUM_MEMORY: 1024m
      JVM_MAXIMUM_MEMORY: 2048m
      ATL_PROXY_NAME: 10.10.200.11
      ATL_PROXY_PORT: 8090
      ATL_TOMCAT_PORT: 8090
    volumes:
      - confluence_data:/var/atlassian/application-data/confluence
    depends_on:
      postgres:
        condition: service_healthy

EOF
```

> **Lưu ý:** Tất cả service dùng `network_mode: host` — containers nghe trực tiếp trên IP `10.10.200.11` của host, không qua Docker NAT. Không cần khai báo `ports` khi dùng host network.

---

### 4. Setup PostgreSQL 16

Khởi động PostgreSQL trước tiên — đây là nền tảng database cho Zabbix và Confluence:

```bash
cd /opt/itil-stack
docker compose up -d postgres
```

Chờ container healthy (khoảng 30 giây), sau đó kiểm tra:

```bash
# Kiểm tra trạng thái
docker compose ps postgres

# Verify kết nối
docker exec -it itil-postgres psql -U postgres -c "\l"
```

<img src="../assets/img/2026-04-22-automated-itil-workflow/02.png"/>

Kết quả hiển thị danh sách database mặc định (postgres, template0, template1) là PostgreSQL đã sẵn sàng.

---

### 5. Setup Zabbix 7.2 + Grafana

#### 5.1. Tạo database Zabbix trong PostgreSQL

Tạo user và database riêng cho Zabbix:

```bash
docker exec -i itil-postgres psql -U postgres << 'EOF'
CREATE USER zabbix WITH PASSWORD 'zabbix_pass_2024';
CREATE DATABASE zabbix
  OWNER zabbix
  ENCODING 'UTF8'
  LC_COLLATE='en_US.utf8'
  LC_CTYPE='en_US.utf8';
GRANT ALL PRIVILEGES ON DATABASE zabbix TO zabbix;
\l
EOF
```

Kiểm tra database `zabbix` đã xuất hiện trong danh sách:

```bash
docker exec -it itil-postgres psql -U postgres -c "\l"
```

<img src="../assets/img/2026-04-22-automated-itil-workflow/03.png"/>

#### 5.2. Khởi động Zabbix Server + Web + Agent

```bash
cd /opt/itil-stack
docker compose up -d zabbix-server zabbix-web zabbix-agent
```

Zabbix Server sẽ tự động khởi tạo schema vào database `zabbix` trong lần khởi động đầu tiên (mất khoảng 1–2 phút). Theo dõi log:

```bash
docker compose logs -f zabbix-server
# Chờ đến khi thấy: "server #0 started [main process]"
```

Kiểm tra trạng thái:

```bash
docker compose ps zabbix-server zabbix-web zabbix-agent
```

<img src="../assets/img/2026-04-22-automated-itil-workflow/04.png"/>

#### 5.3. Cấu hình Zabbix Web lần đầu

Truy cập `http://10.10.200.11:8080`, đăng nhập bằng `Admin / zabbix`.

<img src="../assets/img/2026-04-22-automated-itil-workflow/05.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/06.png"/>

Đổi mật khẩu ngay sau khi đăng nhập: **User Settings → Change Password**.

<img src="../assets/img/2026-04-22-automated-itil-workflow/07.png"/>

**Sửa host "Zabbix server" mặc định:**

Zabbix tự tạo sẵn host "Zabbix server" trỏ về `127.0.0.1` — không hoạt động trong Docker. Cần sửa lại:

**Data Collection → Hosts → Zabbix server → Interfaces → Agent:**
- Loại: **IP**
- IP: `10.10.200.11`
- Port: `10050`

Click **Update** để lưu.


<img src="../assets/img/2026-04-22-automated-itil-workflow/08.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/09.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/10.png"/>

#### 5.4. Cài Zabbix Agent trên Windows Server 2022

Tải Zabbix Agent 2 cho Windows từ [zabbix.com/download](https://www.zabbix.com/download):

- OS: **Windows**
- Package: **Agent 2**

Chạy installer, điền các field theo UI:

| Field | Value |
|---|---|
| **Host name** | `win2022` |
| **Zabbix server IP/DNS** | `10.10.200.11` |
| **Agent listen port** | `10050` |
| **Server or Proxy for active checks** | `10.10.200.11` |
| **Enable PSK** | ☐ (không cần cho lab) |
| **Add agent location to the PATH** | ☑ |

> **PSK** (Pre-Shared Key) — mã hóa TLS giữa Agent và Server. Không bắt buộc trong môi trường lab nội bộ.
<img src="../assets/img/2026-04-22-automated-itil-workflow/11.png"/>
 
<img src="../assets/img/2026-04-22-automated-itil-workflow/12.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/13.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/14.png"/>

Khởi động service:

```powershell
# Kiểm tra service
Get-Service -Name "Zabbix Agent 2"

# Start nếu chưa chạy
Start-Service -Name "Zabbix Agent 2"
```

Mở firewall cho port 10050:

```powershell
New-NetFirewallRule -DisplayName "Zabbix Agent" -Direction Inbound -Protocol TCP -LocalPort 10050 -Action Allow
```

<img src="../assets/img/2026-04-22-automated-itil-workflow/15.png"/>

#### 5.5. Add hosts vào Zabbix

**Data Collection → Hosts → Create host**

**Host 1: win2022**

| Field | Value |
|---|---|
| Host name | `win2022` |
| Groups | `Windows servers (new)` |
| Interfaces > Agent | IP: `10.10.200.12`, Port: `10050` |
| Templates | `Windows by Zabbix agent active` |

> Hostname phải khớp chính xác với **Host name** đã khai báo khi cài agent trên Windows (`win2022`).


<img src="../assets/img/2026-04-22-automated-itil-workflow/16.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/17.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/18.png"/>


**Host 2: itil-stack (self-monitoring)**

> Mô phỏng giám sát server linux 

| Field | Value |
|---|---|
| Host name | `itil-stack` |
| Groups | `Linux servers` |
| Interfaces > Agent | IP: `10.10.200.11`, Port: `10050` |
| Templates | `Linux by Zabbix agent active` |

Đợi vài phút để Zabbix thu thập dữ liệu — trạng thái host chuyển sang màu xanh lá (Available).

<img src="../assets/img/2026-04-22-automated-itil-workflow/19.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/20.png"/>

#### 5.6. Khởi động Grafana

```bash
cd /opt/itil-stack
docker compose up -d grafana
```

Kiểm tra:

```bash
docker compose ps grafana
```
<img src="../assets/img/2026-04-22-automated-itil-workflow/21.png"/>

#### 5.7. Thêm Zabbix datasource

Truy cập `http://10.10.200.11:3000`, đăng nhập `admin / Admin@2024`.

<img src="../assets/img/2026-04-22-automated-itil-workflow/22.png"/>

**Configuration → Plugins** — tìm plugin **Zabbix**, click **Enable**:

<img src="../assets/img/2026-04-22-automated-itil-workflow/23.png"/>

**Configuration → Data Sources → Add data source → Zabbix**

| Field | Value |
|---|---|
| URL | `http://10.10.200.11:8080/api_jsonrpc.php` |
| Username | `Admin` |
| Password | `<zabbix-password>` |

Click **Save & Test** — hiển thị "Zabbix API version: 7.2.x" là thành công.

<img src="../assets/img/2026-04-22-automated-itil-workflow/24.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/25.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/26.png"/>

#### 5.8. Import Zabbix Dashboards

Import dashboard Zabbix từ [grafana.com](https://grafana.com/grafana/dashboards) bằng Dashboard ID.

**Dashboards → New → Import → Import via grafana.com**

Nhập ID rồi click **Load**, chọn **Datasource → Zabbix** rồi click **Import**:

| Dashboard ID | Tên | Mô tả | Trạng thái |
|---|---|---|---|
| `5363` | [Zabbix - Full Server Status](https://grafana.com/grafana/dashboards/5363) | Tổng quan server: CPU, RAM, Disk, Network | ✅ Dùng được |
| `16238` | [ISP PROVISION - Zabbix 6.0 Server](https://grafana.com/grafana/dashboards/16238) | Dashboard hiện đại: Time series, Stat, Gauge — dùng panel Grafana 8+ | ⚠️ Cần plugin triggers panel |

>
> Grafana v10+ **không còn** `grafana-cli` — cài plugin đúng cách qua `GF_INSTALL_PLUGINS` trong `docker-compose.yml`:
> ```yaml
> GF_INSTALL_PLUGINS: alexanderzobnin-zabbix-app,alexanderzobnin-zabbix-triggers-panel
> ```
> Sau đó recreate container:
> ```bash
> docker compose up -d --force-recreate grafana
> ```

<img src="../assets/img/2026-04-22-automated-itil-workflow/27.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/28.png"/>

---

### 6. Setup ServiceDesk Plus 14

#### 6.1. Cài đặt SDP trực tiếp trên Ubuntu

ManageEngine **không có Docker image công khai** cho ServiceDesk Plus — cài trực tiếp trên host bằng installer `.bin`.

```bash
# Tải installer SDP 14 (64-bit Linux)
wget -O /tmp/ManageEngine_ServiceDesk_Plus_64bit.bin \
  "https://www.manageengine.com/products/service-desk/91677414/ManageEngine_ServiceDesk_Plus_64bit.bin"
```

```powershell
# Hoặc copy file từ Windows lên server
scp "D:\softs\ManageEngine_ServiceDesk_Plus_64bit.bin" ubuntu@10.10.200.11:/tmp/

```

> Nếu link trên hết hạn, tải thủ công tại [ManageEngine Downloads](https://sdpondemand.manageengine.com/Register.do?opDownload&deployment=Onpremises&pos=Downloads&loc=servicemgmt&prev=AB3) → **ServiceDesk Plus** → **Linux 64-bit**.

```bash
# Cài các gói phụ thuộc
sudo apt install -y libc6 libstdc++6 fontconfig

chmod +x /tmp/ManageEngine_ServiceDesk_Plus_64bit.bin
```

> **Ubuntu 24.04 — dùng `env -i` khi chạy installer.** Lỗi `Malformed \uxxxx encoding` xảy ra do LAX framework bị ảnh hưởng bởi biến môi trường shell hiện tại. Giải pháp: chạy installer trong môi trường tối giản với `LANG`/`LC_ALL` chuẩn:
>
> ```bash
> sudo env -i \
>   HOME=/root USER=root LOGNAME=root \
>   PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
>   LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 \
>   /tmp/ManageEngine_ServiceDesk_Plus_64bit.bin -i console
> ```

Trong quá trình cài (interactive console), điền:

| Prompt | Giá trị |
|---|---|
| Installation directory | `/opt/ManageEngine/ServiceDesk` |
| Web server port | `8400` |
| Start as service | `Yes` |

<img src="../assets/img/2026-04-22-automated-itil-workflow/29.png"/>


**Cấu hình SDP dùng PostgreSQL Docker dùng chung** (không dùng bundled PostgreSQL của SDP):

Sau khi installer chạy xong, SDP tạo sẵn config với `sdpadmin` user kết nối vào bundled PG port `65432`. Cần chuyển sang Docker PG port `5432`:

```bash
# Tạo user sdpadmin và DB trong Docker PG (giữ đúng username SDP đã cấu hình)
docker exec -i itil-postgres psql -U postgres << 'EOF'
CREATE USER sdpadmin WITH PASSWORD 'sdp_admin_2024';
CREATE DATABASE servicedesk OWNER sdpadmin ENCODING 'UTF8';
GRANT ALL PRIVILEGES ON DATABASE servicedesk TO sdpadmin;
\l
EOF
```

```bash
# Backup config gốc
cp /opt/ManageEngine/ServiceDesk/conf/database_params.conf \
   /opt/ManageEngine/ServiceDesk/conf/database_params.conf.bak

# Viết lại config: đổi port 65432 → 5432, plain text password (SDP tự encrypt khi start)
cat > /opt/ManageEngine/ServiceDesk/conf/database_params.conf << 'DBEOF'
drivername=org.postgresql.Driver
url=jdbc:postgresql://localhost:5432/servicedesk?charSet=UTF-8&loginTimeout=30&socketTimeout=1200&connectTimeout=20&sslmode=disable
username=sdpadmin
password=sdp_admin_2024
rodatasource.drivername=org.postgresql.Driver
rodatasource.url=jdbc:postgresql://localhost:5432/servicedesk?charSet=UTF-8&loginTimeout=5&socketTimeout=1200&connectTimeout=5&sslmode=disable
rodatasource.username=sdpadmin
rodatasource.password=sdp_admin_2024
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

> **Lưu ý:** PostgreSQL Docker chạy với `network_mode: host` nên SDP kết nối được qua `localhost:5432` trực tiếp từ host. SDP sẽ tự động encrypt plain text password và tạo schema lần đầu khởi động.

Khởi động SDP:

```bash
cd /opt/ManageEngine/ServiceDesk/bin
sudo ./run.sh start

# Kiểm tra port 8400 đang lắng nghe
ss -tlnp | grep 8400
```

SDP mất **3–5 phút** để khởi động hoàn toàn. Theo dõi log:

```bash
tail -f /opt/ManageEngine/ServiceDesk/logs/serverOut.txt
# Chờ đến khi thấy: "Server startup in ... ms"
```

<img src="../assets/img/2026-04-22-automated-itil-workflow/30.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/31.png"/>

**Tạo systemd service để SDP tự khởi động :**

```bash
sudo tee /etc/systemd/system/sdp.service << 'EOF'
[Unit]
Description=ManageEngine ServiceDesk Plus
After=network.target

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
```

```bash
# Reload systemd và enable service
sudo systemctl daemon-reload
sudo systemctl enable sdp

# Stop SDP đang chạy (nếu có), rồi start lại qua systemd
cd /opt/ManageEngine/ServiceDesk/bin
sudo ./run.sh stop
sudo systemctl start sdp

# Kiểm tra trạng thái
sudo systemctl status sdp
```


<img src="../assets/img/2026-04-22-automated-itil-workflow/32.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/33.png"/>


#### 6.2. Đăng nhập lần đầu vào SDP

Truy cập `http://10.10.200.11:8400`, đăng nhập bằng tài khoản mặc định:

| Field | Value |
|---|---|
| Username | `administrator` |
| Password | `administrator` |

<img src="../assets/img/2026-04-22-automated-itil-workflow/34.png"/>

SDP yêu cầu đổi mật khẩu ngay lần đầu đăng nhập:

<img src="../assets/img/2026-04-22-automated-itil-workflow/35.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/36.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/37.png"/>

#### 6.3. Tạo 3 kỹ thuật viên

> **Lưu ý:** SDP tự tạo sẵn 5 technician mẫu (Shawn Adams, Heather Graham, John Roberts, Howard Stern, Jeniffer Doe) khi cài đặt. Cần xóa hết trước, sau đó tạo 3 KTV mới cho lab.

**Xóa 5 technician mẫu:**

**Admin → Users & Permission → Technicians** — chọn tất cả 5 technician mẫu → **Actions → Delete**

<img src="../assets/img/2026-04-22-automated-itil-workflow/38.png"/>

**Tạo 3 technician mới — Admin → Users & Permission → Technicians → Add New**

Điền form theo thứ tự cho từng KTV:

| Field | KTV 1 | KTV 2 | KTV 3 |
|---|---|---|---|
| **First Name** | `Nguyen Van` | `Tran Thi` | `Le Van` |
| **Last Name** | `A` | `B` | `C` |
| **Display Name** | `Nguyen Van A` | `Tran Thi B` | `Le Van C` |
| **Primary Email** | `nguyenvana@lab.local` | `tranthib@lab.local` | `levanc@lab.local` |
| **Enable login for this technician** | ☑ | ☑ | ☑ |
| **Login Name** | `nguyenvana` | `tranthib` | `levanc` |
| **Password / Retype Password** | `Abc@12345` | `Abc@12345` | `Abc@12345` |
| **Assigned Roles** | `SDAdmin` | `SDAdmin` | `SDAdmin` |

> - Tick **"Enable login for this technician"** để hiện ra các field Login Name, Password, Retype Password.
> - **Assigned Roles** là bắt buộc, chọn từ dropdown:
>   - `SDAdmin` — Full quyền: nhận, xử lý, đóng ticket, cấu hình hệ thống *(dùng cho lab)*
>   - `SDCo-ordinator` — Điều phối: nhận và xử lý ticket, không có quyền admin *(dùng cho production)*
>   - `SDGuest` — Chỉ xem, không xử lý được ticket *(không dùng)*
> - Điền xong click **Save and Add New** để tạo tiếp KTV tiếp theo.

<img src="../assets/img/2026-04-22-automated-itil-workflow/39.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/40.png"/>
 
#### 6.4. Cấu hình Round-Robin Auto Assign

**Admin → Automation → Technician Auto Assign**

Bật toggle **Enabled**, sau đó cấu hình:

| Field | Giá trị |
|---|---|
| **Select Tech Auto Assign model** | ● Round Robin |
| **Execute when a request is** | ● Created |
| **Apply Tech Auto Assign while creating requests to** | ● All Requests |
| **Assign only to Online Technicians** | ☐ |
| **Include or Exclude Technicians** | ☑ |
| → Criteria | ● **Include** |
| → Technician(s) | `Nguyen Van A`, `Tran Thi B`, `Le Van C` |
| **Configure Rules** | ☐ |

> **Lưu ý:** Chọn **Include** (không phải Exclude). Nếu để Exclude — 3 KTV này sẽ bị loại khỏi round-robin, ngược lại với mục đích.

Click **Save**.

<img src="../assets/img/2026-04-22-automated-itil-workflow/41.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/42.png"/>

#### 6.5. Lấy API Token — cập nhật macro Zabbix

Trong SDP, API authtoken là **per-technician** — mỗi KTV có key riêng. Lấy key của `administrator` (tài khoản dùng để tạo ticket qua webhook):

<img src="../assets/img/2026-04-22-automated-itil-workflow/43.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/44.png"/>


Sau đó cập nhật macro global trong Zabbix:

**Zabbix: Administration → General → Macros**

| Macro | Value |
|---|---|
| `{$SDP_API_TOKEN}` | `<authtoken-vừa-copy>` |

<img src="../assets/img/2026-04-22-automated-itil-workflow/45.png"/>

#### 6.6. Tạo Webhook Media Type gọi SDP API

**Alerts → Media types → Create media type**

Điền **Name** = `ServiceDesk Plus`, **Type** = `Webhook`.

Zabbix tự tạo sẵn 5 parameter mặc định: `URL`, `HTTPProxy`, `To`, `Subject`, `Message` — click **Remove** để xóa hết cả 5.


Sau đó click **Add** để thêm lần lượt 8 parameter sau:

| Name | Value |
|---|---|
| `sdp_url` | `http://10.10.200.11:8400` |
| `sdp_token` | `{$SDP_API_TOKEN}` |
| `event_id` | `{EVENT.ID}` |
| `host` | `{HOST.NAME}` |
| `trigger_name` | `{TRIGGER.NAME}` |
| `trigger_severity` | `{TRIGGER.SEVERITY}` |
| `event_status` | `{EVENT.STATUS}` |
| `event_time` | `{EVENT.TIME} {EVENT.DATE}` |

**Script (Webhook):**

> **Lưu ý:** Copy script bên dưới, paste trực tiếp vào ô Script trong Zabbix — **không qua Word hay Notepad** để tránh lỗi encoding ký tự đặc biệt.

```javascript
var params = JSON.parse(value);

var technicians = ['Nguyen Van A', 'Tran Thi B', 'Le Van C'];
var techIndex = parseInt(params.event_id) % 3;
var assignedTech = technicians[techIndex];

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

> **Lỗi `SyntaxError: invalid json (at offset 1)` khi test:**  
> Nguyên nhân: script bị lỗi encoding khi copy-paste (ký tự Unicode trong comment/string bị hỏng), khiến `value` thành `undefined`. Cách kiểm tra: thay toàn bộ nội dung script bằng một dòng duy nhất `return value;` rồi Test lại — nếu response hiển thị JSON thì script cũ bị lỗi encoding, xóa đi và nhập lại script sạch ở trên.

<img src="../assets/img/2026-04-22-automated-itil-workflow/46.png"/>

#### 6.7. Tạo Action tự động tạo ticket

**Alerts → Actions → Trigger actions → Create action**

**Action tab:**

| Field | Value |
|---|---|
| Name | `Auto Create SDP Ticket` |
| Conditions | Trigger severity >= Warning |

<img src="../assets/img/2026-04-22-automated-itil-workflow/47.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/48.png"/>

**Operations tab → Add operation:**

| Field | Value |
|---|---|
| Operation type | Send message |
| Send to users | Admin |
| Send only to | ServiceDesk Plus |

<img src="../assets/img/2026-04-22-automated-itil-workflow/49.png"/>

**Recovery operations:** thêm operation Send message để thông báo khi RESOLVED.

<img src="../assets/img/2026-04-22-automated-itil-workflow/49.png"/>

> **Lưu ý quan trọng:** Tạo action xong **vẫn chưa đủ**. Zabbix yêu cầu mỗi user phải được gán media type trong profile thì mới gửi được alert. Tiếp tục bước dưới.

**Gán media "ServiceDesk Plus" cho user Admin:**

**Users → Users → Admin → tab Media → Add**

| Field | Value |
|---|---|
| Type | `ServiceDesk Plus` |
| Send to | `1` (bất kỳ giá trị — được truyền vào `{ALERT.SENDTO}`, webhook không dùng) |
| When active | `1-7,00:00-24:00` |
| Use if severity | tích hết các severity |

Click **Add → Update**.

<img src="../assets/img/2026-04-22-automated-itil-workflow/50.png"/> 

**Kiểm tra webhook hoạt động — Test thủ công:**

**Alerts → Media types → ServiceDesk Plus → Test**

Zabbix hiện form test với các parameter hiển thị nguyên macro `{EVENT.ID}`, `{HOST.NAME}`... — các macro này **không được resolve** khi test thủ công. Cần **xóa và nhập giá trị thật** vào từng ô:

| Parameter | Nhập giá trị |
|---|---|
| `event_id` | `1` |
| `event_status` | `PROBLEM` |
| `event_time` | `04:00:00 2026-04-22` |
| `host` | `win2022` |
| `sdp_token` | `<token thật lấy từ mục 6.5>` |
| `sdp_url` | `https://10.10.200.11:8400` |
| `trigger_name` | `Test trigger` |
| `trigger_severity` | `High` |

Click **Test**.

<img src="../assets/img/2026-04-22-automated-itil-workflow/51.png"/>

- **Thành công:** Response hiển thị `Ticket created: <số ID> → Assigned to: Nguyen Van A`
- **Thất bại:** Response hiển thị lỗi JSON hoặc `false` → kiểm tra:
  1. Token đúng chưa? (copy lại từ SDP → Edit Technician administrator → Authtoken details)
  2. SDP đang chạy chưa? `ss -tlnp | grep 8400`
  3. Xem log chi tiết:

```bash
docker compose -f /opt/itil-stack/docker-compose.yml logs --tail=30 zabbix-server | grep -i "webhook\|alert\|sdp"
```

Sau khi test thành công, vào SDP kiểm tra ticket vừa tạo tại **Requests** — sẽ thấy ticket mới với subject `[High] Test trigger - win2022`.

<img src="../assets/img/2026-04-22-automated-itil-workflow/52.png"/> 

#### 6.8. Thêm SDP datasource Grafana

Cài Infinity plugin để Grafana đọc dữ liệu từ SDP REST API:

```bash
docker exec itil-grafana grafana-cli plugins install yesoreyeram-infinity-datasource
docker restart itil-grafana
```

**Connections → Data Sources → Add data source → Infinity**

| Field | Value |
|---|---|
| Name | `ServiceDesk Plus` |
| Base URL | `http://10.10.200.11:8400/api/v3` |
| HTTP Headers → Header | `authtoken` |
| HTTP Headers → Value | `<api-key-từ-SDP>` (lấy ở mục 6.5) |

Click **Save & Test**.

<img src="../assets/img/2026-04-22-automated-itil-workflow/sdp-grafana-datasource.png"/>

#### 6.9. Tạo dashboard ITIL Overview

**Dashboards → New Dashboard → Add visualization**

| Panel | Datasource | Query | Visualization |
|---|---|---|---|
| Host Status | Zabbix | Host availability | Stat |
| Active Problems | Zabbix | Problems | Table |
| CPU Usage | Zabbix | CPU utilization (all hosts) | Time series |
| Disk Usage | Zabbix | Used disk space on `/` | Gauge |
| SDP Open Tickets | Infinity | `/requests?input_data={"list_info":{"search_fields":{"status.name":"Open"}}}` | Stat |

<img src="../assets/img/2026-04-22-automated-itil-workflow/sdp-grafana-dashboard.png"/>

---

### 7. Setup Confluence 9

#### 7.1. Tạo database Confluence trong PostgreSQL

Tạo user và database riêng cho Confluence:

```bash
docker exec -i itil-postgres psql -U postgres << 'EOF'
CREATE USER confluence WITH PASSWORD 'confluence_pass_2024';
CREATE DATABASE confluence
  OWNER confluence
  ENCODING 'UTF8'
  LC_COLLATE='C'
  LC_CTYPE='C'
  TEMPLATE template0;
GRANT ALL PRIVILEGES ON DATABASE confluence TO confluence;
\l
EOF
```

Kiểm tra cả 2 database `zabbix` và `confluence` đã tồn tại:

```bash
docker exec -it itil-postgres psql -U postgres -c "\l"
```

#### 7.2. Khởi động Confluence

```bash
cd /opt/itil-stack
docker compose up -d confluence

# Confluence mất 3-5 phút để khởi động
docker compose logs -f confluence
# Chờ đến khi thấy: "Confluence is ready to serve"
```

<img src="../assets/img/2026-04-22-automated-itil-workflow/53.png"/> 

#### 7.3. Setup wizard + trial license

Truy cập `http://10.10.200.11:8090`, hoàn tất setup wizard:

<img src="../assets/img/2026-04-22-automated-itil-workflow/54.png"/> 

1. **Setup type:** Production installation
2. **Database:** chọn **External Database → PostgreSQL**
   - Host: `10.10.200.11`
   - Port: `5432`
   - Database: `confluence`
   - Username: `confluence`
   - Password: `confluence_pass_2024`
3. **License:** vào [Atlassian Trial](https://www.atlassian.com/try) để nhận trial license, nhập vào

<img src="../assets/img/2026-04-22-automated-itil-workflow/21-confluence-setup.png"/>

#### 7.4. Tạo Space và Page báo cáo

1. Tạo **Space**: `IT Operations` (key: `ITOPS`)
2. Tạo **Page**: `Incident Report — Automated`

Lấy **Page ID** qua API:

```bash
curl -u admin:admin_password \
  "http://10.10.200.11:8090/rest/api/content?spaceKey=ITOPS&title=Incident+Report+Automated" \
  | python3 -m json.tool | grep '"id"' | head -1
```

<img src="../assets/img/2026-04-22-automated-itil-workflow/22-confluence-page.png"/>

#### 7.5. Script cập nhật Confluence qua API

Tạo script Python trong thư mục `alertscripts`:

```bash
cat > /opt/itil-stack/zabbix/alertscripts/confluence_update.py << 'PYEOF'
#!/usr/bin/env python3
# /opt/itil-stack/zabbix/alertscripts/confluence_update.py
# Zabbix gọi: confluence_update.py <to> <subject> <body>

import sys
import json
import urllib.request
import datetime
import base64

# ─── CẤU HÌNH ────────────────────────────────────────────────────────
CONFLUENCE_URL  = "http://10.10.200.11:8090"
CONFLUENCE_USER = "admin"
CONFLUENCE_PASS = "admin_password"   # đổi sau khi setup
PAGE_ID         = "12345"            # thay bằng Page ID thực tế
# ─────────────────────────────────────────────────────────────────────

def get_page(page_id):
    url = f"{CONFLUENCE_URL}/rest/api/content/{page_id}?expand=body.storage,version"
    creds = base64.b64encode(f"{CONFLUENCE_USER}:{CONFLUENCE_PASS}".encode()).decode()
    req = urllib.request.Request(url, headers={
        "Authorization": f"Basic {creds}",
        "Content-Type": "application/json"
    })
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())

def update_page(page_id, subject, body_text):
    page = get_page(page_id)
    version = page["version"]["number"] + 1
    current = page["body"]["storage"]["value"]
    now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    new_row = (
        f"<tr><td>{now}</td>"
        f"<td>{subject}</td>"
        f"<td>{body_text.replace(chr(10), '<br/>')}</td>"
        f"<td>Open</td></tr>"
    )

    if "<table>" not in current:
        updated = (
            "<h2>Incident Log</h2><table><tbody>"
            "<tr><th>Time</th><th>Subject</th><th>Detail</th><th>Status</th></tr>"
            + new_row + "</tbody></table>"
        )
    else:
        updated = current.replace("</tr>", "</tr>" + new_row, 1)

    payload = json.dumps({
        "version": {"number": version},
        "title": page["title"],
        "type": "page",
        "body": {"storage": {"value": updated, "representation": "storage"}}
    }).encode("utf-8")

    creds = base64.b64encode(f"{CONFLUENCE_USER}:{CONFLUENCE_PASS}".encode()).decode()
    req = urllib.request.Request(
        f"{CONFLUENCE_URL}/rest/api/content/{page_id}",
        data=payload, method="PUT",
        headers={"Authorization": f"Basic {creds}", "Content-Type": "application/json"}
    )
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())["id"]

if __name__ == "__main__":
    if len(sys.argv) < 4:
        print("Usage: confluence_update.py <to> <subject> <body>")
        sys.exit(1)
    try:
        result = update_page(PAGE_ID, sys.argv[2], sys.argv[3])
        print(f"Confluence page updated: {result}")
    except Exception as e:
        print(f"ERROR: {e}", file=sys.stderr)
        sys.exit(1)
PYEOF

chmod +x /opt/itil-stack/zabbix/alertscripts/confluence_update.py
```

Kiểm tra Python có sẵn trong container Zabbix Server:

```bash
docker exec itil-zabbix-server python3 --version || \
  docker exec itil-zabbix-server apt-get install -y python3
```

#### 7.6. Tích hợp vào Zabbix Action

**Alerts → Media types → Create media type**

| Field | Value |
|---|---|
| Name | `Confluence Update` |
| Type | `Script` |
| Script name | `confluence_update.py` |
| Script parameters | `{ALERT.SENDTO}` / `{ALERT.SUBJECT}` / `{ALERT.MESSAGE}` |

**Alerts → Actions → Auto Create SDP Ticket → Operations → Add operation:**

- Operation type: `Send message`
- Media type: `Confluence Update`
- Subject: `[{TRIGGER.SEVERITY}] {TRIGGER.NAME} on {HOST.NAME}`
- Message:
  ```
  Host: {HOST.NAME}
  IP: {HOST.IP}
  Trigger: {TRIGGER.NAME}
  Severity: {TRIGGER.SEVERITY}
  Status: {EVENT.STATUS}
  Time: {EVENT.TIME} {EVENT.DATE}
  Event ID: {EVENT.ID}
  ```

<img src="../assets/img/2026-04-22-automated-itil-workflow/23-zabbix-confluence-action.png"/>

---

### 8. Test Automated Workflow

#### 8.1. Test: Disk Full

**Trên itil-stack (Ubuntu):**

```bash
# Kiểm tra dung lượng trống trước
df -h /

# Tạo file chiếm >90% disk (điều chỉnh count phù hợp)
dd if=/dev/zero of=/tmp/fill_disk_test bs=1M count=85000 status=progress

# Kiểm tra
df -h /
```

**Trên win2022 (Windows):**

```powershell
# Giả lập disk full C:
$path = "C:\fill_disk_test.tmp"
$fs = [System.IO.File]::Create($path)
$fs.SetLength(80GB)
$fs.Close()
Get-PSDrive C
```

**Kết quả mong đợi (1–2 phút):**

1. Zabbix trigger `Disk space is critically low` kích hoạt
2. Webhook gọi SDP API → ticket mới được tạo tự động
3. Ticket assign cho kỹ thuật viên theo round-robin
4. Grafana cập nhật số lượng active problems
5. Confluence page được thêm row mới

<img src="../assets/img/2026-04-22-automated-itil-workflow/24-test-disk-full.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/25-sdp-ticket-created.png"/>

**Dọn dẹp sau test:**

```bash
# Ubuntu
rm /tmp/fill_disk_test

# Windows
Remove-Item "C:\fill_disk_test.tmp"
```

#### 8.2. Test: Host Down

Shutdown VM `win2022` từ vSphere Client hoặc từ bên trong:

```powershell
Stop-Computer -Force
```

Zabbix phát hiện host unreachable sau **~5 phút**.

**Kết quả mong đợi:**

1. Trigger `Zabbix agent is not available` kích hoạt
2. SDP tạo ticket: `[High] Zabbix agent not available — win2022`
3. Round-robin assign cho kỹ thuật viên tiếp theo
4. Confluence cập nhật báo cáo

<img src="../assets/img/2026-04-22-automated-itil-workflow/26-test-host-down.png"/>

<img src="../assets/img/2026-04-22-automated-itil-workflow/27-grafana-overview.png"/>

---

### 9. Kết quả

<img src="../assets/img/2026-04-22-automated-itil-workflow/28-full-workflow-result.png"/>

| Sự kiện | Zabbix | SDP | Grafana | Confluence |
|---|---|---|---|---|
| Disk > 90% | Alert PROBLEM | Ticket tạo tự động | Panel cập nhật | Row mới |
| Host Down | Alert PROBLEM | Ticket tạo tự động | Host đỏ | Row mới |
| Disk OK | Alert RESOLVED | Ticket resolve | Panel xanh | Ghi recovered |
| Host Up | Alert RESOLVED | Ticket resolved | Host xanh | Cập nhật báo cáo |

**Round-robin assignment logic:**

```
event_id mod 3 = 0  →  Le Van C
event_id mod 3 = 1  →  Nguyen Van A
event_id mod 3 = 2  →  Tran Thi B
```

**Tóm tắt các endpoint:**

| Service | URL | Credentials |
|---|---|---|
| Zabbix | `http://10.10.200.11:8080` | `Admin / <password>` |
| Grafana | `http://10.10.200.11:3000` | `admin / Admin@2024` |
| Confluence | `http://10.10.200.11:8090` | `admin / <password>` |
| ServiceDesk Plus | `http://10.10.200.11:8400` | `administrator / admin` |

**Thứ tự khởi động toàn bộ stack (tóm tắt):**

```bash
cd /opt/itil-stack

# 1. PostgreSQL
docker compose up -d postgres

# 2. Tạo DB Zabbix, khởi động Zabbix + Grafana
#    (tạo DB thủ công qua psql trước, xem mục 5.1)
docker compose up -d zabbix-server zabbix-web zabbix-agent grafana

# 3. SDP — cài trực tiếp trên host (không dùng Docker)
# sudo /opt/ManageEngine/ServiceDesk/bin/run.sh start

# 4. Tạo DB Confluence, khởi động Confluence
#    (tạo DB thủ công qua psql trước, xem mục 7.1)
docker compose up -d confluence
```

Lab này cung cấp nền tảng để mở rộng thêm:

- **Auto-resolve ticket SDP** khi Zabbix báo RECOVERY
- **Escalation rules** trong SDP khi ticket quá SLA
- **Notification qua Teams/Slack** song song với SDP
- **Weekly incident report** Confluence tổng hợp tự động mỗi thứ Hai