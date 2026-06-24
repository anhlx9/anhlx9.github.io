# POC: Zabbix 7.4 Full HA (Native HA + Patroni + Proxy Group)

> Mục tiêu: dựng một hệ thống monitor **không có single point of failure** ở cả 3 tầng — Zabbix server, Database (PostgreSQL) và Proxy — trên Ubuntu 24.04, dùng cho mô hình MSP giám sát nhiều khách hàng.

---

## 1. Tổng quan kiến trúc

Hệ thống chia thành **3 lớp HA độc lập**, mỗi lớp có cơ chế failover riêng và **không chồng lên nhau**:

| Lớp | VM | IP | Thành phần | Cơ chế HA |
|---|---|---|---|---|
| Server | zbx-node-01 | 10.10.200.21 | Zabbix server + HAProxy + Keepalived | Native HA (qua DB) |
| Server | zbx-node-02 | 10.10.200.22 | Zabbix server + HAProxy + Keepalived | Native HA (qua DB) |
| Database | pg-node-01 | 10.10.200.31 | etcd + Patroni + PostgreSQL 16 | Patroni (leader election) |
| Database | pg-node-02 | 10.10.200.32 | etcd + Patroni + PostgreSQL 16 | Patroni |
| Database | pg-node-03 | 10.10.200.33 | etcd + Patroni + PostgreSQL 16 | Patroni |
| Proxy | zbx-proxy-01 | 10.10.200.41 | Zabbix proxy (active) | Proxy Group |
| Proxy | zbx-proxy-02 | 10.10.200.42 | Zabbix proxy (active) | Proxy Group |
| Proxy | zbx-proxy-03 | 10.10.200.43 | Zabbix proxy (active) | Proxy Group |
| — | **VIP** | **10.10.200.11** | Virtual IP cho truy cập DB | Keepalived |

### Luồng dữ liệu

```
Agent/Device (KH)
      │  (active checks → list 3 proxy)
      ▼
[Proxy Group: .41 .42 .43]   ── tự cân bằng host & failover ──
      │  Server=.21,.22 (active proxy → active server node)
      ▼
[Zabbix server: .21 / .22]   ── native HA: 1 active, 1 standby ──
      │  DBHost = VIP .11 :5432
      ▼
[VIP .11] → HAProxy (trên .21/.22) → Patroni leader (1 trong .31/.32/.33)
      │
      ▼
[PostgreSQL primary] ←─ streaming replication ─→ [2 replica]
```

> **3 nguyên tắc vàng:**
> 1. `DBHost` của Zabbix server **= VIP 10.10.200.11** (không trỏ thẳng vào node DB).
> 2. Proxy khai `Server=10.10.200.21,10.10.200.22` (**IP thật của 2 node server, KHÔNG phải VIP**).
> 3. Native HA của Zabbix server **không dùng VIP** — node tự bầu active qua bảng `ha_node` trong DB.

---

## 2. Layer 1 — Patroni Cluster (10.10.200.31/32/33)

Mỗi node DB chạy 3 service: **etcd** (DCS) + **Patroni** + **PostgreSQL 16**.

### 2.1. Cài gói (chạy trên cả 3 node)

```bash
sudo apt update
sudo apt install -y postgresql-16 postgresql-client-16 etcd-server python3-pip python3-psycopg2
sudo systemctl disable --now postgresql   # Patroni sẽ tự quản lý postgres
sudo pip3 install --break-system-packages "patroni[etcd3]"
```

### 2.2. etcd cluster 3 node

`/etc/default/etcd` trên **mỗi** node (thay `<NODE_NAME>` / `<NODE_IP>` tương ứng):

```ini
ETCD_NAME="<NODE_NAME>"                       # etcd1 / etcd2 / etcd3
ETCD_DATA_DIR="/var/lib/etcd/default"
ETCD_LISTEN_PEER_URLS="http://<NODE_IP>:2380"
ETCD_LISTEN_CLIENT_URLS="http://<NODE_IP>:2379,http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://<NODE_IP>:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://<NODE_IP>:2379"
ETCD_INITIAL_CLUSTER="etcd1=http://10.10.200.31:2380,etcd2=http://10.10.200.32:2380,etcd3=http://10.10.200.33:2380"
ETCD_INITIAL_CLUSTER_TOKEN="zbx-etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

```bash
sudo systemctl enable --now etcd
etcdctl member list        # kiểm tra: phải thấy đủ 3 member
```

### 2.3. Patroni

`/etc/patroni/patroni.yml` (thay `<NODE_NAME>` = `pg-node-01/02/03`, `<NODE_IP>` tương ứng):

```yaml
scope: zabbix-pg-cluster
name: <NODE_NAME>

restapi:
  listen: 0.0.0.0:8008
  connect_address: <NODE_IP>:8008

etcd3:
  hosts: 10.10.200.31:2379,10.10.200.32:2379,10.10.200.33:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 300
        shared_buffers: 512MB
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
  initdb:
    - encoding: UTF8
    - data-checksums
  pg_hba:
    - host replication replicator 10.10.200.0/24 scram-sha-256
    - host all all 10.10.200.0/24 scram-sha-256
    - local all all trust

postgresql:
  listen: 0.0.0.0:5432
  connect_address: <NODE_IP>:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  authentication:
    superuser:
      username: postgres
      password: "ChangeMe_Super"
    replication:
      username: replicator
      password: "ChangeMe_Repl"

tags:
  failover_priority: 1
```

Service file `/etc/systemd/system/patroni.service`:

```ini
[Unit]
Description=Patroni PostgreSQL HA
After=network.target etcd.service

[Service]
Type=simple
User=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo chown -R postgres:postgres /etc/patroni /var/lib/postgresql
sudo systemctl daemon-reload
sudo systemctl enable --now patroni
patronictl -c /etc/patroni/patroni.yml list   # 1 Leader + 2 Replica
```

### 2.4. Tạo database & user cho Zabbix (chạy 1 lần trên **leader**)

```bash
sudo -u postgres psql -c "CREATE ROLE zabbix WITH LOGIN PASSWORD 'ChangeMe_Zbx';"
sudo -u postgres psql -c "CREATE DATABASE zabbix OWNER zabbix;"
```

> Schema sẽ được import ở bước 3.3 (sau khi cài gói server).

---

## 3. Layer 2 — Zabbix Server Native HA + HAProxy + Keepalived (10.10.200.21/22)

### 3.1. HAProxy — route 5432 về Patroni leader (cả 2 node)

`/etc/haproxy/haproxy.cfg`:

```
global
    maxconn 1000

defaults
    log     global
    mode    tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen postgres_write
    bind *:5432
    option httpchk GET /master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-node-01 10.10.200.31:5432 maxconn 200 check port 8008
    server pg-node-02 10.10.200.32:5432 maxconn 200 check port 8008
    server pg-node-03 10.10.200.33:5432 maxconn 200 check port 8008
```

> Chỉ Patroni leader trả HTTP 200 ở `GET /master:8008`, nên HAProxy luôn đẩy 5432 về đúng primary. Khi leader đổi, HAProxy tự redirect.

```bash
sudo apt install -y haproxy keepalived
sudo systemctl enable --now haproxy
```

### 3.2. Keepalived — quản lý VIP 10.10.200.11

`/etc/keepalived/keepalived.conf` trên **.21 (MASTER)**:

```
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
    fall 2
    rise 2
}

vrrp_instance VI_DB {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication { auth_type PASS; auth_pass Zbx_Vip_2026 }
    virtual_ipaddress { 10.10.200.11/24 }
    track_script { chk_haproxy }
}
```

Trên **.22 (BACKUP)**: y hệt nhưng `state BACKUP` và `priority 100`.

```bash
sudo systemctl enable --now keepalived
ip addr show eth0 | grep 10.10.200.11    # VIP đang ở node nào
```

### 3.3. Zabbix server — bật Native HA

```bash
# Thêm repo Zabbix 7.4 cho Ubuntu 24.04 (lấy file release hiện hành từ repo.zabbix.com)
sudo apt install -y zabbix-server-pgsql zabbix-sql-scripts zabbix-frontend-php zabbix-nginx-conf
```

Import schema (chạy **1 lần duy nhất**, từ node .21, qua VIP):

```bash
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | \
  psql -h 10.10.200.11 -U zabbix -d zabbix
```

`/etc/zabbix/zabbix_server.conf` — **node .21**:

```ini
DBHost=10.10.200.11        # ← VIP, KHÔNG phải IP node DB
DBName=zabbix
DBUser=zabbix
DBPassword=ChangeMe_Zbx
DBPort=5432

# ── Native HA ──
HANodeName=zbx-node-01
NodeAddress=10.10.200.21:10051
```

`/etc/zabbix/zabbix_server.conf` — **node .22**: y hệt, chỉ đổi:

```ini
HANodeName=zbx-node-02
NodeAddress=10.10.200.22:10051
```

```bash
sudo systemctl enable --now zabbix-server
# Kiểm tra trạng thái cluster (chạy trên node active):
sudo zabbix_server -R ha_status
```

Kết quả mong đợi: 1 node `active`, 1 node `standby`.

### 3.4. Frontend — để tự dò node active

`/etc/zabbix/web/zabbix.conf.php`:

```php
$DB['TYPE']     = 'POSTGRESQL';
$DB['SERVER']   = '10.10.200.11';   // VIP
$DB['PORT']     = '5432';
$DB['DATABASE'] = 'zabbix';
$DB['USER']     = 'zabbix';
$DB['PASSWORD'] = 'ChangeMe_Zbx';

// QUAN TRỌNG: comment 2 dòng dưới để frontend tự phát hiện node active qua NodeAddress
// $ZBX_SERVER      = '';
// $ZBX_SERVER_PORT = '';
```

> Truy cập UI qua IP từng node (`http://10.10.200.21`) — frontend tự kết nối tới node server đang active. Nếu thiếu `NodeAddress`, trang Queue/System info sẽ trống.

---

## 4. Layer 3 — Zabbix Proxy Group (10.10.200.41/42/43)

### 4.1. Cài & cấu hình (cả 3 proxy)

```bash
sudo apt install -y zabbix-proxy-sqlite3 zabbix-sql-scripts
```

`/etc/zabbix/zabbix_proxy.conf` (đổi `Hostname` theo từng node):

```ini
ProxyMode=0                              # 0 = active (proxy tự connect ra server)
Server=10.10.200.21,10.10.200.22         # ← cả 2 node server (IP thật)
Hostname=zbx-proxy-01                    # zbx-proxy-02 / zbx-proxy-03
DBName=/var/lib/zabbix/zabbix_proxy.db
```

```bash
sudo systemctl enable --now zabbix-proxy
```

### 4.2. Khai báo trên Frontend

1. **Data collection → Proxies**: tạo 3 proxy `zbx-proxy-01/02/03`, Mode = *Active*. Điền **Address for active agents** = IP proxy tương ứng (cần cho cơ chế redirect agent).
2. **Data collection → Proxy groups**: tạo group `PG-ALL-CUSTOMERS`:
   - *Failover period*: `1m` (range 10s–15m)
   - *Minimum number of proxies*: `2` (quorum — nhỏ hơn tổng số proxy)
   - Gán cả 3 proxy vào group.
3. Với mọi host khách hàng: **Monitored by → Proxy group → PG-ALL-CUSTOMERS** (không gán proxy đơn lẻ).

> Server tự phân bổ host đều giữa 3 proxy. Khi 1 proxy chết quá *Failover period*, host của nó được redistribute sang proxy còn lại. Đây là lý do **tối thiểu 3 proxy** mới HA ổn định.

---

## 5. Cấu hình Agent phía khách hàng

`/etc/zabbix/zabbix_agent2.conf` (hoặc zabbix_agentd):

```ini
# Active checks: liệt kê toàn bộ proxy trong group
ServerActive=10.10.200.41,10.10.200.42,10.10.200.43
# Passive checks:
Server=10.10.200.41,10.10.200.42,10.10.200.43
Hostname=<ten-host-trung-voi-frontend>
```

> ⚠️ **Cảnh báo firewall (lỗi hay gặp):** agent **phải reach được CẢ 3 proxy**. Khi load balancing/failover, proxy hiện tại có thể redirect agent sang proxy khác; nếu proxy đó bị firewall chặn, active check sẽ treo chờ timeout. Test kỹ rule trước khi tin là đã HA.

---

## 6. Ma trận test Failover (nghiệm thu POC)

| # | Hành động | Kết quả mong đợi | Lệnh kiểm chứng |
|---|---|---|---|
| 1 | `systemctl stop zabbix-server` trên node active | Node standby chuyển sang active trong ~`HA failover delay` (mặc định 60s); thu thập data tiếp tục | `zabbix_server -R ha_status` |
| 2 | Shutdown node đang giữ VIP | Keepalived chuyển VIP `.11` sang node kia; kết nối DB không gián đoạn | `ip a | grep .11` ; query qua `psql -h .11` |
| 3 | `patronictl switchover` / kill Patroni leader | Patroni bầu leader mới; HAProxy tự route 5432 về primary mới | `patronictl ... list` ; `curl node:8008/master` |
| 4 | `systemctl stop zabbix-proxy` trên 1 proxy | Sau `Failover period`, host của proxy đó được phân bổ lại sang 2 proxy còn lại | Proxy groups UI → cột host count |
| 5 | Mất link agent → proxy hiện tại | Agent tự discard redirect, quay về dùng list `ServerActive`, kết nối proxy khác | Log agent ; latest data của host |

**Tiêu chí PASS:** ở mọi kịch bản 1–5, không có host nào chuyển sang `Not supported`/no-data quá lâu hơn ngưỡng failover tương ứng.

---

## 7. Ghi chú vận hành

- **Major upgrade với Native HA:** phải hạ 1 node về standalone (comment `HANodeName`) để chạy DB upgrade, rồi mới bật lại HA — đừng upgrade khi replication đang lệch.
- **etcd quorum:** 3 node chịu được mất 1 node. Đừng để xuống còn 2 node lâu dài.
- **HAProxy stats (tuỳ chọn):** thêm `listen stats bind *:7000 mode http stats uri /` để soi backend nào đang UP.
- **Read-only offload (mở rộng sau):** có thể thêm 1 backend HAProxy port 5433 với `GET /replica` để Grafana đọc báo cáo từ replica, giảm tải primary.
- **Backup:** snapshot Patroni leader bằng `pg_basebackup`/`pgBackRest` về S3 — tách khỏi cụm HA (HA ≠ backup).

---

*Bản POC này dùng mật khẩu mẫu `ChangeMe_*` — đổi toàn bộ trước khi chạy thật, và cân nhắc bật PSK/TLS cho kênh server↔proxy↔agent.*
