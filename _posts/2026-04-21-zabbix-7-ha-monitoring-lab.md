---
title: "Zabbix 7.2 HA Monitoring Lab — Cluster, Proxy, Grafana trên ESXi"
categories:
- System
- Monitoring
- Infrastructure

feature_image: "../assets/postbanner.jpg"
feature_text: |
  ### Xây dựng hệ thống monitoring production-grade với Zabbix 7.2 HA cluster, Zabbix Proxy cho isolated VLAN, và Grafana dashboard chạy hoàn toàn on-premise trên VMware ESXi
---

### Mục lục

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Thiết kế Lab](#2-thiết-kế-lab)
  - [2.1. Topology tổng thể](#21-topology-tổng-thể)
  - [2.2. VLAN và Network design](#22-vlan-và-network-design)
  - [2.3. VM Plan](#23-vm-plan)
- [3. Chuẩn bị ESXi — Tạo Portgroup và VLAN](#3-chuẩn-bị-esxi--tạo-portgroup-và-vlan)
- [4. Cài đặt PostgreSQL trên Zabbix Node 1](#4-cài-đặt-postgresql-trên-zabbix-node-1)
  - [4.1. Cài PostgreSQL 16](#41-cài-postgresql-16)
  - [4.2. Tạo database Zabbix](#42-tạo-database-zabbix)
  - [4.3. Cấu hình Streaming Replication sang Node 2](#43-cấu-hình-streaming-replication-sang-node-2)
- [5. Cài đặt Zabbix Server 7.2 HA](#5-cài-đặt-zabbix-server-72-ha)
  - [5.1. Cài Zabbix trên Node 1](#51-cài-zabbix-trên-node-1)
  - [5.2. Cài Zabbix trên Node 2](#52-cài-zabbix-trên-node-2)
  - [5.3. Cấu hình Keepalived VIP](#53-cấu-hình-keepalived-vip)
  - [5.4. Cài Zabbix Frontend (Nginx)](#54-cài-zabbix-frontend-nginx)
- [6. Cài đặt Zabbix Proxy](#6-cài-đặt-zabbix-proxy)
- [7. Cài đặt Zabbix Agent](#7-cài-đặt-zabbix-agent)
  - [7.1. Agent trên Ubuntu (VLAN 201)](#71-agent-trên-ubuntu-vlan-201)
  - [7.2. Agent trên Windows Server 2022 (VLAN 202)](#72-agent-trên-windows-server-2022-vlan-202)
- [8. Cấu hình Monitoring trong Zabbix](#8-cấu-hình-monitoring-trong-zabbix)
- [9. Cài đặt Grafana và kết nối Zabbix](#9-cài-đặt-grafana-và-kết-nối-zabbix)
- [10. Kiểm tra HA Failover](#10-kiểm-tra-ha-failover)

---

## 1. Giới thiệu

Trong môi trường production, hệ thống monitoring phải đảm bảo **High Availability** — không thể để monitoring server chính là single point of failure. Bài lab này xây dựng đầy đủ stack monitoring production-grade trên ESXi:

- **Zabbix 7.2 HA cluster** — 2 node active/standby, chia sẻ PostgreSQL, Keepalived VIP
- **Zabbix Proxy** — đứng ở VLAN isolated, thu thập data từ các host không có route trực tiếp đến Zabbix server
- **Grafana** — dashboard visualization kết nối qua Zabbix API
- **3 VLAN tách biệt** — mô phỏng môi trường thực tế: VLAN management (200), VLAN proxy (201), VLAN monitored hosts (202)

**Tại sao cần Zabbix Proxy?**

Trong thực tế, các VLAN production (DB server, app server...) thường bị firewall chặn route trực tiếp đến Zabbix server ở VLAN management. Zabbix Proxy đứng trong VLAN đó, thu thập data cục bộ rồi gửi về server — chỉ cần mở 1 kết nối từ proxy ra server (port 10051), không cần mở từng host.

---

## 2. Thiết kế Lab

### 2.1. Topology tổng thể

```
                    Internet
                       │
                  pfSense (GW)
                  10.10.200.1
                       │
          ─────────────┼─────────────
                   VLAN 200
          ─────────────┼─────────────
                       │
         ┌─────────────┼─────────────┐
         │             │             │
  zabbix-node1   zabbix-node2    grafana*
  10.10.200.21   10.10.200.22   (trên node1)
         │             │
         └──── VIP: 10.10.200.20 (Keepalived)
                   Zabbix Frontend
                       │
                       │ (Proxy kết nối về port 10051)
         ┌─────────────┘
         │
  zabbix-proxy
  VLAN200: 10.10.200.24
  VLAN201: 10.10.201.1
  VLAN202: 10.10.202.1
         │               │
         │               │
    VLAN 201        VLAN 202
  (host-only)     (host-only)
         │               │
  ubuntu-agent    win2022-agent
  10.10.201.10    10.10.202.10

* Grafana chạy trên zabbix-node1 (port 3000)
```

### 2.2. VLAN và Network design

| VLAN | Tên | Subnet | Mục đích |
|---|---|---|---|
| VLAN 200 | DC-Net | 10.10.200.0/24 | Management, Zabbix server, internet via pfSense |
| VLAN 201 | Host-Only-201 | 10.10.201.0/24 | Isolated — Ubuntu monitored hosts |
| VLAN 202 | Host-Only-202 | 10.10.202.0/24 | Isolated — Windows monitored hosts |

**ESXi vSwitch design:**

```
vSwitch0 (uplink: vmnic0)          vSwitch1 (no uplink — host-only)
├── VM Network (VLAN 200)          ├── VLAN201-HostOnly (VLAN 201)
│   └── gateway: 10.10.200.1       └── VLAN202-HostOnly (VLAN 202)
```

> VLAN 201 và 202 dùng **vSwitch riêng không có uplink** → hoàn toàn isolated, không route ra ngoài.

### 2.3. VM Plan

| VM | OS | Role | VLAN | IP | vCPU | RAM | Disk |
|---|---|---|---|---|---|---|---|
| zabbix-node1 | Ubuntu 24.04 | Zabbix Server HA node 1 + PostgreSQL primary + Grafana | 200 | 10.10.200.21 | 4 | 8 GB | 60 GB |
| zabbix-node2 | Ubuntu 24.04 | Zabbix Server HA node 2 + PostgreSQL standby | 200 | 10.10.200.22 | 4 | 8 GB | 60 GB |
| zabbix-proxy | Ubuntu 24.04 | Zabbix Proxy (passive) | 200+201+202 | 10.10.200.24 / 10.10.201.1 / 10.10.202.1 | 2 | 4 GB | 40 GB |
| ubuntu-agent | Ubuntu 24.04 | Monitored host (VLAN 201) | 201 | 10.10.201.10 | 2 | 2 GB | 20 GB |
| win2022-agent | Windows Server 2022 | Monitored host (VLAN 202) | 202 | 10.10.202.10 | 2 | 4 GB | 60 GB |
| **VIP** | — | Keepalived VIP (Zabbix HA + Frontend) | 200 | **10.10.200.20** | — | — | — |

> Tổng cộng: **5 VM**, ~26 vCPU, ~26 GB RAM, ~240 GB disk. ESXi host cần tối thiểu 32 GB RAM vật lý để lab thoải mái.

---

## 3. Chuẩn bị ESXi — Tạo Portgroup và VLAN

Đăng nhập **ESXi Host Client** → **Networking**:

**Tạo vSwitch1 (host-only, không uplink):**

1. **Virtual Switches → Add standard virtual switch**
   - Name: `vSwitch1`
   - Uplinks: **để trống** (không add vmnic nào)

2. **Port groups → Add port group**
   - Name: `VLAN201-HostOnly`, VLAN ID: `201`, vSwitch: `vSwitch1`

3. **Port groups → Add port group**
   - Name: `VLAN202-HostOnly`, VLAN ID: `202`, vSwitch: `vSwitch1`

**VM Network (VLAN 200) đã có sẵn trên vSwitch0.**

**Cấu hình NIC cho zabbix-proxy (3 NIC):**

Khi tạo VM zabbix-proxy → **Edit Settings → Add Network Adapter**:
- NIC 1: `VM Network` (VLAN 200)
- NIC 2: `VLAN201-HostOnly`
- NIC 3: `VLAN202-HostOnly`

<img src="../assets/img/2026-04-21-zabbix-7-ha-monitoring-lab/01-esxi-vswitch.png"/>

---

## 4. Cài đặt PostgreSQL trên Zabbix Node 1

### 4.1. Cài PostgreSQL 16

**Thực hiện trên zabbix-node1 (10.10.200.21):**

```bash
# Cấu hình hostname
hostnamectl set-hostname zabbix-node1

# Cấu hình /etc/hosts (thêm vào cả 2 node)
cat >> /etc/hosts << 'EOF'
10.10.200.21 zabbix-node1
10.10.200.22 zabbix-node2
10.10.200.20 zabbix-vip
10.10.200.24 zabbix-proxy
10.10.201.10 ubuntu-agent
10.10.202.10 win2022-agent
EOF

# Cài PostgreSQL 16
apt update
apt install -y curl ca-certificates
install -d /usr/share/postgresql-common/pgdg
curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail \
  https://www.postgresql.org/media/keys/ACCC4CF8.asc

sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
  https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
  > /etc/apt/sources.list.d/pgdg.list'

apt update
apt install -y postgresql-16

systemctl enable postgresql
systemctl start postgresql
```

### 4.2. Tạo database Zabbix

```bash
# Tạo user và database
sudo -u postgres psql << 'EOF'
CREATE USER zabbix WITH PASSWORD 'zabbix_password_2026';
CREATE DATABASE zabbix OWNER zabbix ENCODING UTF8 LC_COLLATE 'en_US.UTF-8' LC_CTYPE 'en_US.UTF-8' TEMPLATE template0;
GRANT ALL PRIVILEGES ON DATABASE zabbix TO zabbix;
EOF
```

Cho phép node2 và proxy kết nối đến PostgreSQL:

```bash
# Cấu hình postgresql.conf — lắng nghe tất cả interface
sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" \
  /etc/postgresql/16/main/postgresql.conf

# Cấu hình pg_hba.conf — cho phép kết nối từ VLAN 200
cat >> /etc/postgresql/16/main/pg_hba.conf << 'EOF'
# Zabbix nodes (VLAN 200)
host    zabbix          zabbix          10.10.200.21/32         scram-sha-256
host    zabbix          zabbix          10.10.200.22/32         scram-sha-256
# Replication user
host    replication     replicator      10.10.200.22/32         scram-sha-256
EOF

systemctl restart postgresql
```

### 4.3. Cấu hình Streaming Replication sang Node 2

**Trên node1** — tạo replication user:

```bash
sudo -u postgres psql -c "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'repl_pass_2026';"
```

Cấu hình `postgresql.conf` cho replication:

```bash
cat >> /etc/postgresql/16/main/postgresql.conf << 'EOF'
# Replication
wal_level = replica
max_wal_senders = 3
wal_keep_size = 256MB
hot_standby = on
EOF

systemctl restart postgresql
```

**Trên zabbix-node2** — cấu hình hostname và cài PostgreSQL:

```bash
hostnamectl set-hostname zabbix-node2

# Copy /etc/hosts giống node1
cat >> /etc/hosts << 'EOF'
10.10.200.21 zabbix-node1
10.10.200.22 zabbix-node2
10.10.200.20 zabbix-vip
10.10.200.24 zabbix-proxy
EOF

# Cài PostgreSQL 16 (giống node1)
apt update && apt install -y curl ca-certificates
install -d /usr/share/postgresql-common/pgdg
curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail \
  https://www.postgresql.org/media/keys/ACCC4CF8.asc
sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
  https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
  > /etc/apt/sources.list.d/pgdg.list'
apt update && apt install -y postgresql-16
systemctl stop postgresql

# Xóa data dir mặc định, đồng bộ từ node1
rm -rf /var/lib/postgresql/16/main
sudo -u postgres pg_basebackup -h 10.10.200.21 -U replicator \
  -D /var/lib/postgresql/16/main -P -Xs -R
# Nhập password: repl_pass_2026

systemctl start postgresql

# Kiểm tra standby
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
# Trả về: t (true) = đang là standby
```

> **Tại sao cần replication trong lab?** Nếu node1 (DB primary) bị shutdown, ta có thể promote node2 thành primary bằng `pg_ctl promote` — Zabbix server trên node2 tiếp tục hoạt động với DB local. Đây là mô hình HA thực tế nhất cho lab với 2 node.

---

## 5. Cài đặt Zabbix Server 7.2 HA

### 5.1. Cài Zabbix trên Node 1

**Thực hiện trên zabbix-node1:**

```bash
# Thêm Zabbix 7.2 repository
wget https://repo.zabbix.com/zabbix/7.2/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.2+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.2+ubuntu24.04_all.deb
apt update

# Cài Zabbix server, frontend, nginx
apt install -y zabbix-server-pgsql zabbix-frontend-php \
  zabbix-nginx-conf zabbix-sql-scripts zabbix-agent2

# Import schema database (chỉ làm trên node1 — node2 nhận qua replication)
zcat /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz | \
  sudo -u zabbix psql zabbix
```

Cấu hình Zabbix server HA node 1:

```bash
cat > /etc/zabbix/zabbix_server.conf << 'EOF'
LogFile=/var/log/zabbix/zabbix_server.log
LogFileSize=0
PidFile=/run/zabbix/zabbix_server.pid
SocketDir=/run/zabbix
DBHost=10.10.200.21
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix_password_2026
DBPort=5432
StartPollers=50
StartPingers=10
StartTrappers=10
CacheSize=512M
HistoryCacheSize=64M
HistoryIndexCacheSize=16M
TrendCacheSize=64M
ValueCacheSize=256M
Timeout=30
LogSlowQueries=3000
StatsAllowedIP=127.0.0.1

# HA Configuration
HANodeName=zabbix-node1
NodeAddress=10.10.200.21:10051
EOF

systemctl enable zabbix-server zabbix-agent2
systemctl start zabbix-server zabbix-agent2
```

### 5.2. Cài Zabbix trên Node 2

**Thực hiện trên zabbix-node2:**

```bash
# Cài Zabbix server (giống node1, KHÔNG import schema)
wget https://repo.zabbix.com/zabbix/7.2/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.2+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.2+ubuntu24.04_all.deb
apt update
apt install -y zabbix-server-pgsql zabbix-agent2
```

Cấu hình node 2 — **chú ý HANodeName và NodeAddress khác node1:**

```bash
cat > /etc/zabbix/zabbix_server.conf << 'EOF'
LogFile=/var/log/zabbix/zabbix_server.log
LogFileSize=0
PidFile=/run/zabbix/zabbix_server.pid
SocketDir=/run/zabbix
DBHost=10.10.200.21
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix_password_2026
DBPort=5432
StartPollers=50
StartPingers=10
StartTrappers=10
CacheSize=512M
HistoryCacheSize=64M
HistoryIndexCacheSize=16M
TrendCacheSize=64M
ValueCacheSize=256M
Timeout=30
LogSlowQueries=3000
StatsAllowedIP=127.0.0.1

# HA Configuration — khác node1
HANodeName=zabbix-node2
NodeAddress=10.10.200.22:10051
EOF

systemctl enable zabbix-server zabbix-agent2
systemctl start zabbix-server zabbix-agent2
```

> **Kiểm tra HA hoạt động — chạy trên node1:**
> ```bash
> zabbix_server -R ha_status
> # Output:
> # Failover delay: 60 seconds
> # Cluster status:
> #  #  Address                    Name                 Status
> #  1. 10.10.200.21:10051         zabbix-node1         active
> #  2. 10.10.200.22:10051         zabbix-node2         standby
> ```

<img src="../assets/img/2026-04-21-zabbix-7-ha-monitoring-lab/02-zabbix-ha-status.png"/>

### 5.3. Cấu hình Keepalived VIP

Keepalived quản lý VIP `10.10.200.20` — khi node1 bị lỗi, VIP tự động chuyển sang node2.

**Cài trên cả 2 node:**

```bash
apt install -y keepalived
```

**Cấu hình node1 (MASTER):**

```bash
cat > /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
  router_id zabbix-node1
  script_user root
  enable_script_security
}

vrrp_script chk_zabbix {
  script "/usr/bin/pgrep zabbix_server"
  interval 5
  weight -20
  fall 2
  rise 2
}

vrrp_instance VI_ZABBIX {
  state MASTER
  interface ens160
  virtual_router_id 51
  priority 110
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass zabbix_ha_2026
  }
  virtual_ipaddress {
    10.10.200.20/24
  }
  track_script {
    chk_zabbix
  }
}
EOF

systemctl enable keepalived
systemctl start keepalived
```

**Cấu hình node2 (BACKUP):** priority thấp hơn:

```bash
cat > /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
  router_id zabbix-node2
  script_user root
  enable_script_security
}

vrrp_script chk_zabbix {
  script "/usr/bin/pgrep zabbix_server"
  interval 5
  weight -20
  fall 2
  rise 2
}

vrrp_instance VI_ZABBIX {
  state BACKUP
  interface ens160
  virtual_router_id 51
  priority 100
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass zabbix_ha_2026
  }
  virtual_ipaddress {
    10.10.200.20/24
  }
  track_script {
    chk_zabbix
  }
}
EOF

systemctl enable keepalived
systemctl start keepalived
```

Kiểm tra VIP đã lên trên node1:

```bash
ip addr show ens160 | grep "10.10.200.20"
# 10.10.200.20/24 brd 10.10.200.255 scope global secondary ens160
```

### 5.4. Cài Zabbix Frontend (Nginx)

**Thực hiện trên zabbix-node1 (nơi có Grafana, cũng là Frontend chính):**

```bash
apt install -y zabbix-frontend-php zabbix-nginx-conf php8.3-pgsql
```

Cấu hình Nginx cho Zabbix Frontend:

```bash
# Uncomment listen port trong Zabbix nginx config
sed -i 's/#        listen/        listen/' /etc/zabbix/nginx.conf
sed -i 's/#        server_name/        server_name/' /etc/zabbix/nginx.conf

# Sửa server_name thành VIP
sed -i 's/server_name  example.com;/server_name  10.10.200.20;/' /etc/zabbix/nginx.conf

systemctl enable nginx php8.3-fpm
systemctl restart nginx php8.3-fpm zabbix-server
```

Truy cập `http://10.10.200.20` → Zabbix setup wizard:

- **DB Host:** `10.10.200.21`
- **DB Name:** `zabbix`
- **DB User:** `zabbix`
- **DB Password:** `zabbix_password_2026`
- **Zabbix Server:** `10.10.200.20` (VIP)

Default login: `Admin` / `zabbix`

<img src="../assets/img/2026-04-21-zabbix-7-ha-monitoring-lab/03-zabbix-frontend.png"/>

---

## 6. Cài đặt Zabbix Proxy

Zabbix Proxy chạy trên VM có 3 NIC — kết nối về Zabbix Server qua VLAN 200, thu thập data từ VLAN 201 và VLAN 202.

**Thực hiện trên zabbix-proxy:**

```bash
hostnamectl set-hostname zabbix-proxy

# Cấu hình IP cho 3 NIC
# ens160 = VLAN 200, ens192 = VLAN 201, ens224 = VLAN 202
cat > /etc/netplan/00-installer-config.yaml << 'EOF'
network:
  version: 2
  ethernets:
    ens160:
      addresses: [10.10.200.24/24]
      routes:
        - to: default
          via: 10.10.200.1
      nameservers:
        addresses: [8.8.8.8]
    ens192:
      addresses: [10.10.201.1/24]
    ens224:
      addresses: [10.10.202.1/24]
EOF
netplan apply

# Thêm /etc/hosts
cat >> /etc/hosts << 'EOF'
10.10.200.21 zabbix-node1
10.10.200.20 zabbix-vip
10.10.201.10 ubuntu-agent
10.10.202.10 win2022-agent
EOF
```

```bash
# Cài Zabbix Proxy 7.2
wget https://repo.zabbix.com/zabbix/7.2/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.2+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.2+ubuntu24.04_all.deb
apt update
apt install -y zabbix-proxy-sqlite3 zabbix-agent2
```

Cấu hình Zabbix Proxy:

```bash
cat > /etc/zabbix/zabbix_proxy.conf << 'EOF'
LogFile=/var/log/zabbix/zabbix_proxy.log
LogFileSize=0
PidFile=/run/zabbix/zabbix_proxy.pid
SocketDir=/run/zabbix

# Kết nối về Zabbix Server VIP
Server=10.10.200.20

# Proxy mode: 0 = active (proxy chủ động gửi về server)
ProxyMode=0
Hostname=zabbix-proxy

# SQLite local buffer (không cần DB riêng)
DBName=/var/lib/zabbix/zabbix_proxy.db
DBSocket=/var/lib/zabbix/zabbix_proxy.sock

ProxyLocalBuffer=1
ProxyOfflineBuffer=720
StartPollers=20
StartPingers=5
Timeout=30
EOF

mkdir -p /var/lib/zabbix
chown zabbix:zabbix /var/lib/zabbix

systemctl enable zabbix-proxy zabbix-agent2
systemctl start zabbix-proxy zabbix-agent2
```

**Đăng ký Proxy trên Zabbix Frontend:**

Vào **Administration → Proxies → Create proxy**:
- **Proxy name:** `zabbix-proxy`
- **Proxy mode:** `Active`
- **Interface:** `10.10.200.24`

<img src="../assets/img/2026-04-21-zabbix-7-ha-monitoring-lab/04-proxy-config.png"/>

---

## 7. Cài đặt Zabbix Agent

### 7.1. Agent trên Ubuntu (VLAN 201)

**Thực hiện trên ubuntu-agent (10.10.201.10):**

```bash
hostnamectl set-hostname ubuntu-agent

# Cấu hình IP
cat > /etc/netplan/00-installer-config.yaml << 'EOF'
network:
  version: 2
  ethernets:
    ens160:
      addresses: [10.10.201.10/24]
      routes:
        - to: default
          via: 10.10.201.1   # Default GW là Proxy VM
      nameservers:
        addresses: [8.8.8.8]
EOF
netplan apply

# Cài Zabbix Agent 2
wget https://repo.zabbix.com/zabbix/7.2/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.2+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.2+ubuntu24.04_all.deb
apt update
apt install -y zabbix-agent2
```

Cấu hình agent — báo cáo về **proxy** (không phải server trực tiếp):

```bash
cat > /etc/zabbix/zabbix_agent2.conf << 'EOF'
LogFile=/var/log/zabbix/zabbix_agent2.log
LogFileSize=0
Server=10.10.201.1
ServerActive=10.10.201.1
Hostname=ubuntu-agent
RefreshActiveChecks=120
BufferSend=5
BufferSize=100
EOF

systemctl enable zabbix-agent2
systemctl start zabbix-agent2
```

> **Lưu ý routing trong VLAN isolated:** ubuntu-agent dùng gateway `10.10.201.1` (IP VLAN 201 của proxy VM). Proxy VM cần bật IP forwarding:
> ```bash
> # Trên zabbix-proxy
> echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
> sysctl -p
> ```

### 7.2. Agent trên Windows Server 2022 (VLAN 202)

Tải Zabbix Agent 2 cho Windows từ `https://www.zabbix.com/download`:

1. **Version:** 7.2 | **OS:** Windows | **Component:** Agent 2
2. Chọn installer `.msi` → download
3. Cài đặt với các thông số:
   - **Zabbix server IP:** `10.10.202.1` (IP VLAN 202 của proxy)
   - **Host name:** `win2022-agent`
   - **Listen port:** `10050`

Hoặc cài qua PowerShell (silent install):

```powershell
# Download Zabbix Agent 2 MSI
$url = "https://cdn.zabbix.com/zabbix/binaries/stable/7.2/7.2.0/zabbix_agent2-7.2.0-windows-amd64-openssl.msi"
Invoke-WebRequest -Uri $url -OutFile C:\zabbix_agent2.msi

# Silent install
msiexec /i C:\zabbix_agent2.msi /quiet `
  SERVER=10.10.202.1 `
  SERVERACTIVE=10.10.202.1 `
  HOSTNAME=win2022-agent `
  ENABLEPATH=1

# Kiểm tra service
Get-Service "Zabbix Agent 2"
Start-Service "Zabbix Agent 2"
Set-Service "Zabbix Agent 2" -StartupType Automatic
```

**Cấu hình Windows Firewall** cho phép agent:

```powershell
New-NetFirewallRule -DisplayName "Zabbix Agent" `
  -Direction Inbound -Protocol TCP -LocalPort 10050 `
  -Action Allow
```

> Windows agent kết nối đến proxy qua `10.10.202.1` (IP VLAN 202 của proxy VM). VLAN 202 là host-only, mọi traffic đi qua proxy.

---

## 8. Cấu hình Monitoring trong Zabbix

**Thêm hosts được monitor qua proxy:**

Vào **Data collection → Hosts → Create host**:

**Host ubuntu-agent:**
- **Host name:** `ubuntu-agent`
- **Interfaces → Agent:** `10.10.201.10`, port `10050`
- **Templates:** `Linux by Zabbix agent`
- **Monitored by proxy:** `zabbix-proxy`

**Host win2022-agent:**
- **Host name:** `win2022-agent`
- **Interfaces → Agent:** `10.10.202.10`, port `10050`
- **Templates:** `Windows by Zabbix agent`
- **Monitored by proxy:** `zabbix-proxy`

Kiểm tra status sau vài phút tại **Monitoring → Hosts** — cột **Availability** chuyển xanh = OK.

<img src="../assets/img/2026-04-21-zabbix-7-ha-monitoring-lab/05-hosts-monitoring.png"/>

---

## 9. Cài đặt Grafana và kết nối Zabbix

**Thực hiện trên zabbix-node1:**

```bash
# Thêm Grafana repository
apt install -y apt-transport-https software-properties-common
wget -q -O - https://apt.grafana.com/gpg.key | \
  gpg --dearmor | tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] \
  https://apt.grafana.com stable main" | \
  tee /etc/apt/sources.list.d/grafana.list

apt update
apt install -y grafana

systemctl enable grafana-server
systemctl start grafana-server
```

**Cài plugin Grafana-Zabbix:**

```bash
grafana-cli plugins install alexanderzobnin-zabbix-app
systemctl restart grafana-server
```

Truy cập Grafana tại `http://10.10.200.21:3000` (default: `admin`/`admin`):

1. **Administration → Plugins → Zabbix** → Enable
2. **Connections → Data Sources → Add → Zabbix**:
   - **URL:** `http://10.10.200.20/api_jsonrpc.php` (Zabbix API qua VIP)
   - **Username:** `Admin`
   - **Password:** `zabbix`
   - **Test & Save**

3. Import dashboard mẫu:
   - **Dashboards → Import → Dashboard ID:** `7`  (Zabbix Server Health)
   - **Dashboard ID:** `10171` (Linux host metrics)
   - **Dashboard ID:** `12168` (Windows host metrics)

<img src="../assets/img/2026-04-21-zabbix-7-ha-monitoring-lab/06-grafana-dashboard.png"/>

---

## 10. Kiểm tra HA Failover

Test failover bằng cách tắt Zabbix service trên node1:

```bash
# Trên node1 — dừng Zabbix server
systemctl stop zabbix-server

# Theo dõi HA status trên node2
watch -n 5 'zabbix_server -R ha_status'
```

Sau khoảng 60 giây (failover delay mặc định), node2 lên thành `active`:

```
# Cluster status:
#  #  Address                    Name                 Status
#  1. 10.10.200.21:10051         zabbix-node1         stopped
#  2. 10.10.200.22:10051         zabbix-node2         active
```

Kiểm tra Keepalived VIP đã chuyển sang node2:

```bash
# Trên node2
ip addr show ens160 | grep "10.10.200.20"
# VIP 10.10.200.20 đã xuất hiện trên node2
```

Truy cập `http://10.10.200.20` — Frontend vẫn load bình thường. Monitoring tiếp tục không gián đoạn.

<img src="../assets/img/2026-04-21-zabbix-7-ha-monitoring-lab/07-ha-failover.png"/>

**Khôi phục node1:**

```bash
# Trên node1 — start lại
systemctl start zabbix-server

# Node1 tự động join lại cluster với role standby
zabbix_server -R ha_status
#  1. 10.10.200.21:10051         zabbix-node1         standby
#  2. 10.10.200.22:10051         zabbix-node2         active
```

> Sau failover, node2 giữ vai trò `active`. Để trả node1 về active cần force failover thủ công hoặc restart node2.

---

**Tổng kết Lab:**

| Component | Status | IP/URL |
|---|---|---|
| Zabbix Frontend | `http://10.10.200.20` | VIP — tự động failover |
| Zabbix Server HA node1 | `10.10.200.21:10051` | Active (default) |
| Zabbix Server HA node2 | `10.10.200.22:10051` | Standby |
| Zabbix Proxy | `10.10.200.24` | Active — monitor VLAN 201+202 |
| Grafana | `http://10.10.200.21:3000` | Dashboard |
| ubuntu-agent | `10.10.201.10` | Monitored via proxy |
| win2022-agent | `10.10.202.10` | Monitored via proxy |
