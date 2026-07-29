---
title: "Kong Gateway hybrid mode — 2 Control Plane, 2 Data Plane, PostgreSQL HA bằng Patroni"
categories:
- Linux
- Database
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Tách control plane khỏi data plane trên 4 node, chuẩn bị nền cho Kong Enterprise
---

Kong Gateway là lớp đứng chắn trước toàn bộ API của một tổ chức. Mọi request đi vào nó trước, và nó quyết định request thuộc service nào, có được đi tiếp không, đi với tốc độ bao nhiêu — rồi mới chuyển xuống backend. Nhân là nginx với OpenResty, còn giá trị nằm ở hệ plugin: xác thực (`key-auth`, JWT, OIDC), giới hạn tần suất, biến đổi request/response, ghi log, xuất metric. Thứ trước đây mỗi service tự cài đặt lấy, mỗi nơi một kiểu, được kéo về một chỗ.

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/kong-production-architecture.png)

Trong production, Kong thường nằm ngay sau load balancer và trước cụm microservice. Ba mô hình triển khai phổ biến:

1. **DB-backed truyền thống** — mọi node Kong đều nối vào PostgreSQL chung, node nào cũng vừa nhận traffic vừa mở Admin API để sửa cấu hình.
2. **DB-less** — cấu hình nạp từ một file YAML khai báo, không có database. Đây là cách Kong chạy phổ biến nhất trên Kubernetes, cấu hình đến từ CRD.
3. **Hybrid mode** — tách hẳn làm hai phần riêng biệt, và đây là kiến trúc của bài này.

Hybrid chia cụm thành **Control Plane** và **Data Plane**. CP giữ toàn bộ cấu hình trong PostgreSQL và là nơi duy nhất có Admin API để ghi. DP chỉ hứng traffic của người dùng; nó **không mở một kết nối nào tới database**, không có Admin API, và nhận cấu hình bằng cách để CP đẩy xuống qua một kênh mTLS riêng, rồi cache lại vào `lmdb` trên đĩa của chính nó.

Cách chia này giải quyết đúng hai vấn đề của mô hình truyền thống. Node hứng traffic là node tiếp xúc nhiều rủi ro nhất, mà ở mô hình cũ nó lại vừa cầm credential database vừa mở sẵn Admin API — thứ mà Kong OSS **không hề có authentication**. Và khi database chết, mô hình cũ sập cả cụm, còn hybrid thì DP vẫn phục vụ bình thường từ cache. Bài test số 5 của lab này chứng minh đúng điều đó.

Còn một chuyện về phiên bản. Kong từng có nhánh OSS miễn phí song song với Enterprise, nhưng **OSS đã dừng ở 3.9.1**: từ 3.10 không còn package, không còn release mới, và không còn bản vá bảo mật. Ai chạy OSS trong production thì đây mới là điểm đáng cân nhắc, không phải chuyện thiếu tính năng.

Những gì phải mua Enterprise mới có: RBAC và workspace, Dev Portal, Vitals, nhóm plugin `openid-connect` / `mtls-auth` / `rate-limiting-advanced`, secrets management với vault backend. Kong Manager bản OSS ở cổng `8002` vẫn dùng được nhưng ai vào cũng là admin.

Lab này chọn **3.9.1 — bản OSS cuối cùng** vì mục tiêu là phần kiến trúc: kênh mTLS `8005` hoạt động ra sao, cấu hình đi từ CP xuống DP bằng đường nào, DP xoay xở thế nào khi database biến mất. Toàn bộ những thứ đó giống hệt nhau giữa OSS và Enterprise, nên đây chính là nền để thêm RBAC, Dev Portal và plugin Enterprise vào sau.

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/00.png)

Các thành phần trong lab:

- **Kong Gateway OSS 3.9.1** — cài trên cả 4 node, cùng một package, chỉ khác nhau ở `role` trong `kong.conf`: hai node thành CP, hai node thành DP.
- **PostgreSQL 16 + Patroni 4** — chỗ duy nhất chứa cấu hình Kong. Chỉ Control Plane nói chuyện với nó.
- **etcd 3.5** — DCS cho Patroni bầu leader. 3 member để có quorum.
- **HAProxy** — lớp tiếp nhận duy nhất trên cả 4 node. Nó giữ mọi cổng mà bên ngoài gọi vào, chia tải sang Kong của cả hai node trong cặp, và trên CP còn kiêm việc route kết nối database về Patroni leader hiện tại.
- **keepalived** — cấp endpoint cho HAProxy chứ không tự chia tải: VIP `.51` Up trên cặp CP, VIP `.50` Up trên cặp DP.
- **demo-api** — upstream REST CRUD viết bằng stdlib Python, chạy 2 instance để test load balancing, health check và retry.

## Mục lục

- [Mục lục](#mục-lục)
- [Mục tiêu](#mục-tiêu)
- [Kiến trúc](#kiến-trúc)
- [Môi trường](#môi-trường)
  - [Bảng node](#bảng-node)
  - [Bảng VIP](#bảng-vip)
  - [Port matrix](#port-matrix)
- [Các bước thực hiện](#các-bước-thực-hiện)
  - [Step 1: Chuẩn hóa OS trên cả 4 node](#step-1-chuẩn-hóa-os-trên-cả-4-node)
  - [Step 2: HAProxy trên cả 4 node](#step-2-haproxy-trên-cả-4-node)
    - [2.1 — Cấu hình trên hai node CP](#21--cấu-hình-trên-hai-node-cp)
    - [2.2 — Cấu hình trên hai node DP](#22--cấu-hình-trên-hai-node-dp)
  - [Step 3: keepalived cấp hai VIP](#step-3-keepalived-cấp-hai-vip)
    - [3.1 — VIP `.51` trên hai node CP](#31--vip-51-trên-hai-node-cp)
    - [3.2 — VIP `.50` trên hai node DP](#32--vip-50-trên-hai-node-dp)
  - [Step 4: Dựng etcd cluster 3 node](#step-4-dựng-etcd-cluster-3-node)
  - [Step 5: PostgreSQL 16 và Patroni](#step-5-postgresql-16-và-patroni)
  - [Step 6: Tạo user và database kong](#step-6-tạo-user-và-database-kong)
  - [Step 7: Cài Kong 3.9.1 lên cả 4 node](#step-7-cài-kong-391-lên-cả-4-node)
  - [Step 8: Chứng chỉ cho kênh mTLS](#step-8-chứng-chỉ-cho-kênh-mtls)
  - [Step 9: Dựng schema bằng migrations](#step-9-dựng-schema-bằng-migrations)
  - [Step 10: Cấu hình Control Plane](#step-10-cấu-hình-control-plane)
    - [10.1 — kong.conf](#101--kongconf)
    - [10.2 — Password và thứ tự khởi động, qua systemd drop-in](#102--password-và-thứ-tự-khởi-động-qua-systemd-drop-in)
    - [10.3 — Khởi động và kiểm tra](#103--khởi-động-và-kiểm-tra)
  - [Step 11: Cấu hình Data Plane](#step-11-cấu-hình-data-plane)
    - [11.1 — kong.conf](#111--kongconf)
    - [11.2 — Kiểm tra kênh sync từ phía CP](#112--kiểm-tra-kênh-sync-từ-phía-cp)
    - [11.3 — Kiểm tra từ phía DP](#113--kiểm-tra-từ-phía-dp)
  - [Step 12: Sample app backend](#step-12-sample-app-backend)
  - [Step 13: Khai báo Service và Route](#step-13-khai-báo-service-và-route)
  - [Step 14: Gắn plugin key-auth và rate limiting](#step-14-gắn-plugin-key-auth-và-rate-limiting)
- [Kiểm tra kết quả](#kiểm-tra-kết-quả)
  - [1. Cấu hình đã sync đều hai DP](#1-cấu-hình-đã-sync-đều-hai-dp)
  - [2. Data Plane thật sự read-only](#2-data-plane-thật-sự-read-only)
  - [3. Failover PostgreSQL](#3-failover-postgresql)
  - [4. Failover Control Plane](#4-failover-control-plane)
  - [5. Bài test then chốt — down database](#5-bài-test-then-chốt--down-database)
  - [6. Health check và failover Data Plane](#6-health-check-và-failover-data-plane)
- [Đường lên Enterprise](#đường-lên-enterprise)
- [Kết luận](#kết-luận)

## Mục tiêu

Dựng một cụm Kong Gateway OSS hybrid mode đầy đủ HA ở cả ba tầng — database, control plane, data plane — rồi kiểm chứng bằng thực nghiệm bốn điều:

- Cấu hình chỉ tạo được qua Admin API của Control Plane, Data Plane hoàn toàn read-only.
- Data Plane nhận cấu hình qua kênh mTLS `8005` và cache vào `lmdb` trên disk.
- **Tắt sạch cụm PostgreSQL thì Data Plane vẫn proxy bình thường** — đây là điểm bán hàng chính của hybrid mode.
- Rớt một CP hoặc một DP đều không làm gián đoạn traffic.

Sau lab này bạn biết đúng chỗ nào cần license Enterprise, chỗ nào OSS đã đủ.

## Kiến trúc

```
                        ┌──────────────────────────────┐
   Client ──▶ VIP .50 ──▶│  HAProxy :80 :443            │
                        │  + keepalived (2 node DP)    │
                        └───────┬──────────────┬───────┘
                                │              │
                        ┌───────▼──────┐ ┌─────▼────────┐
                        │  Kong-DP01   │ │  Kong-DP02   │
                        │ :8000 :8443  │ │ :8000 :8443  │
                        │ role=data_   │ │ role=data_   │
                        │ plane        │ │ plane        │
                        │ database=off │ │ database=off │
                        └───────┬──────┘ └─────┬────────┘
                                │ mTLS 8005    │
                                └───────┬──────┘
                                   VIP .51
                        ┌───────────────▼──────────────┐
                        │  HAProxy :8005 :8001 :8002   │
                        │  mode tcp passthrough        │
                        │  + keepalived (2 node CP)    │
                        └───────┬──────────────┬───────┘
                                │ :18005       │ :18005
                ┌───────────────▼┐            ┌▼───────────────┐
                │   Kong-CP01    │            │   Kong-CP02    │
                │ :18005 :18001  │            │ :18005 :18001  │
                │ role=control_  │            │ role=control_  │
                │ plane          │            │ plane          │
                │ proxy_listen=  │            │ proxy_listen=  │
                │ off            │            │ off            │
                └───────┬────────┘            └────────┬───────┘
                        │ 127.0.0.1:5000               │
                ┌───────▼────────┐            ┌────────▼───────┐
                │ HAProxy local  │            │ HAProxy local  │
                └───────┬────────┘            └────────┬───────┘
                        └──────────┬───────────────────┘
                                   │
              ┌────────────────────▼──────────────────┐
              │  Patroni cluster (scope: kong-pg)     │
              │  Kong-CP01 leader                     │
              │  Kong-CP02 replica                    │
              │  Kong-DP01 replica (nofailover)       │
              │  PostgreSQL 16 + etcd 3 member        │
              └───────────────────────────────────────┘
```

Năm quyết định thiết kế đáng nói:

**HAProxy giữ cổng công khai, Kong lùi về cổng nội bộ.** Trên cặp DP điều này là hiển nhiên: HAProxy nghe `80`/`443`, Kong nghe `8000`/`8443`. Cặp CP làm y hệt — HAProxy nghe `8005`/`8001`/`8002`, Kong lùi về `18005`/`18001`/`18002`. Nhờ vậy hai tiến trình không bao giờ tranh cổng, và địa chỉ mà DP hay admin gọi vào luôn là địa chỉ của load balancer chứ không phải của một node cụ thể.

**keepalived là endpoint, HAProxy là load balancer.** VIP chỉ đảm bảo *luôn có một địa chỉ sống* để gọi vào; quyết định request đi về Kong nào là của HAProxy ngay sau nó. Kết quả là bốn node Kong đều active — cặp CP chia nhau kết nối cluster của DP và request Admin API, cặp DP chia nhau traffic client. Không có node nào chỉ standby lãng phí tài nguyên.

**Kênh CP–DP đi qua L4 passthrough, không phải L7.** HAProxy chia tải ở `mode tcp`, không đụng vào TLS, byte nào vào byte đó ra. Nhờ vậy `cluster_mtls = shared` (mặc định) hoạt động nguyên vẹn: hai bên dùng chung cặp cert `CN=kong_clustering`, DP bắt tay TLS thẳng với Kong ở CP bất kể dial vào IP nào. Chỉ cần thêm `ssl` vào dòng `bind` để HAProxy terminate TLS là phải chuyển sang `cluster_mtls = pki` — ngoài phạm vi bài này.

**Patroni đặt ở Kong-CP01, Kong-CP02 và Kong-DP01.** etcd cần 3 member mới có quorum. Hai CP là bên thật sự dùng database nên giữ vai trò leader và sync standby; Kong-DP01 gắn tag `nofailover: true` — chỉ làm replica và etcd voter, không bao giờ được lên leader, để tải write không rơi vào node đang gánh traffic.

**Không có VIP cho PostgreSQL.** Mỗi CP dùng chính HAProxy của mình, Kong trỏ `pg_host = 127.0.0.1` port `5000`. HAProxy hỏi REST API của Patroni (`/primary` trên port `8008`) để biết node nào đang là leader và chỉ route về đó. Đây là pattern chuẩn của Patroni, bớt được một cặp keepalived và không có single point of failure ở tầng VIP database.

## Môi trường

| Thành phần | Thông số |
|-----------|---------|
| OS | Ubuntu 24.04.4 LTS (noble) |
| CPU/RAM | 4 vCPU / 8GB mỗi node |
| Storage | 100GB |
| Network | VLAN 200 — 10.10.200.0/24, gateway 10.10.200.1 |
| Kong | Gateway OSS 3.9.1 (repo `gateway-39`) |
| Database | PostgreSQL 16 (repo mặc định của noble) + Patroni 4 |
| DCS | etcd 3.5.17 |

### Bảng node

| Host | IP | Kong role | Dịch vụ ghép thêm |
|------|-----|-----------|-------------------|
| Kong-CP01 | 10.10.200.11 | `control_plane` | HAProxy, keepalived (MASTER .51), etcd, Patroni/PG16, demo-api |
| Kong-CP02 | 10.10.200.12 | `control_plane` | HAProxy, keepalived (BACKUP .51), etcd, Patroni/PG16, demo-api |
| Kong-DP01 | 10.10.200.21 | `data_plane` | HAProxy, keepalived (MASTER .50), etcd, Patroni/PG16 (`nofailover`) |
| Kong-DP02 | 10.10.200.22 | `data_plane` | HAProxy, keepalived (BACKUP .50) |

### Bảng VIP

| VIP | IP | Nổi trên | HAProxy chia tải sang | Phục vụ |
|-----|-----|----------|----------------------|---------|
| Cửa vào client | 10.10.200.50 | Kong-DP01, Kong-DP02 | Kong của cả hai DP | `80` → DP `8000`, `443` → DP `8443` |
| Cluster + admin của CP | 10.10.200.51 | Kong-CP01, Kong-CP02 | Kong của cả hai CP | `8005` → CP `18005`, `8001` → CP `18001`, `8002` → CP `18002` |

### Port matrix

Đọc bảng này theo đúng một quy tắc: **cổng công khai thuộc về HAProxy, cổng `1xxxx` và `8000`/`8443` thuộc về Kong.**

| Port | Node | Chủ sở hữu | Mục đích |
|------|------|-----------|---------|
| 8005 | Kong-CP01, Kong-CP02 | HAProxy | Cluster config sync CP ↔ DP (mTLS), phục vụ VIP `.51` |
| 8001 | Kong-CP01, Kong-CP02 | HAProxy | Kong Admin API, phục vụ VIP `.51` |
| 8002 | Kong-CP01, Kong-CP02 | HAProxy | Kong Manager OSS, phục vụ VIP `.51` |
| 18005 / 18001 / 18002 | Kong-CP01, Kong-CP02 | Kong | Cổng nội bộ, chỉ HAProxy gọi vào |
| 80 / 443 | Kong-DP01, Kong-DP02 | HAProxy | Cửa vào của client, phục vụ VIP `.50` |
| 8000 / 8443 | Kong-DP01, Kong-DP02 | Kong | Cổng proxy nội bộ, chỉ HAProxy gọi vào |
| 5000 / 5001 | Kong-CP01, Kong-CP02 | HAProxy | → PostgreSQL leader / replica, bind `127.0.0.1` |
| 7000 | cả 4 node | HAProxy | Trang stats |
| 8100 | cả 4 node | Kong | Status API — health check của HAProxy và endpoint Prometheus |
| 2379 / 2380 | Kong-CP01, Kong-CP02, Kong-DP01 | etcd | client / peer |
| 5432 | Kong-CP01, Kong-CP02, Kong-DP01 | PostgreSQL | — |
| 8008 | Kong-CP01, Kong-CP02, Kong-DP01 | Patroni | REST API, HAProxy dùng làm health check |
| 9001 | Kong-CP01, Kong-CP02 | demo-api | Upstream giả lập |

Backend `demo-api` cố tình đặt trên hai node CP: CP không proxy traffic nên còn dư tài nguyên, và cách này cho ta hai upstream target thật ở hai IP khác nhau mà không cần thêm VM.

## Các bước thực hiện

Lab đi theo 5 giai đoạn, mỗi giai đoạn làm dứt điểm trên đúng nhóm node của nó rồi mới sang giai đoạn sau. Thứ tự này quan trọng: lớp dưới luôn có mặt trước khi lớp trên cần đến nó, nên không có bước nào phải quay lại sửa.

| Giai đoạn | Step | Làm trên |
|-----------|------|----------|
| A. Chuẩn bị hạ tầng | 1 | cả 4 node |
| B. Lớp tiếp nhận | 2–3 | cả 4 node |
| C. Database | 4–6 | Kong-CP01, Kong-CP02, Kong-DP01 |
| D. Kong | 7–11 | cả 4 node |
| E. Ứng dụng và plugin | 12–14 | CP cho backend, Admin API cho phần còn lại |

### Step 1: Chuẩn hóa OS trên cả 4 node

**Mục tiêu:** đặt hostname, phân giải tên nội bộ, đồng bộ giờ, bật sysctl cần cho HAProxy/keepalived, nới giới hạn file descriptor, và tạo SSH key để các bước sau copy file giữa các node không phải gõ password.

Chạy trên **từng node**, thay tên cho đúng:

```bash
# Đổi tên tương ứng: Kong-CP01 / Kong-CP02 / Kong-DP01 / Kong-DP02
hostnamectl set-hostname Kong-CP01

tee -a /etc/hosts << 'EOF'
10.10.200.11  Kong-CP01
10.10.200.12  Kong-CP02
10.10.200.21  Kong-DP01
10.10.200.22  Kong-DP02
10.10.200.51  Kong-CP-VIP
10.10.200.50  Kong-Proxy-VIP
EOF

timedatectl set-timezone Asia/Ho_Chi_Minh
sed -i 's/^#\?NTP=.*/NTP=vn.pool.ntp.org/' /etc/systemd/timesyncd.conf
systemctl restart systemd-timesyncd
apt update && apt install -y curl jq gnupg ca-certificates psmisc net-tools
```

Sysctl:

```bash
tee /etc/sysctl.d/99-kong-lab.conf << 'EOF'
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
net.core.somaxconn = 65535
EOF
sysctl --system
```

```bash
tee /etc/security/limits.d/99-kong.conf << 'EOF'
*     soft nofile 65535
*     hard nofile 65535
root  soft nofile 65535
root  hard nofile 65535
EOF
```

### Step 2: HAProxy trên cả 4 node

**Mục tiêu:** dựng lớp tiếp nhận trước khi có bất kỳ backend nào. Từ giờ mọi thứ phía sau chỉ việc mọc lên và HAProxy sẽ tự phát hiện.

Ở bước này **toàn bộ backend sẽ DOWN** — Kong và PostgreSQL chưa tồn tại. Đó là trạng thái đúng, HAProxy vẫn start bình thường và chờ sẵn.

#### 2.1 — Cấu hình trên hai node CP

Chạy trên **Kong-CP01 và Kong-CP02**, nội dung giống hệt nhau:

```bash
apt install -y haproxy

tee /etc/haproxy/haproxy.cfg << 'EOF'
global
    log /dev/log local0
    maxconn 4000
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    tcp
    retries 2
    timeout client  30m
    timeout server  30m
    timeout connect 4s
    timeout check   5s

listen stats
    mode http
    bind 0.0.0.0:7000
    stats enable
    stats uri /
    stats refresh 5s

# ── Database: chỉ Patroni leader trả 200 cho /primary ──
listen pg_primary
    bind 127.0.0.1:5000
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server Kong-CP01 10.10.200.11:5432 maxconn 150 check port 8008
    server Kong-CP02 10.10.200.12:5432 maxconn 150 check port 8008
    server Kong-DP01 10.10.200.21:5432 maxconn 150 check port 8008

# Endpoint read-only, không dùng cho Kong nhưng tiện để kiểm tra replica
listen pg_replica
    bind 127.0.0.1:5001
    balance roundrobin
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2
    server Kong-CP01 10.10.200.11:5432 maxconn 150 check port 8008
    server Kong-CP02 10.10.200.12:5432 maxconn 150 check port 8008
    server Kong-DP01 10.10.200.21:5432 maxconn 150 check port 8008

# ── Kênh cluster CP ↔ DP: TCP passthrough thuần, KHÔNG terminate TLS ──
listen kong_cluster
    mode tcp
    bind 0.0.0.0:8005
    balance leastconn
    option httpchk GET /status
    http-check expect status 200
    timeout client 1h
    timeout server 1h
    default-server inter 3s fall 3 rise 2 check port 8100 on-marked-down shutdown-sessions
    server Kong-CP01 10.10.200.11:18005
    server Kong-CP02 10.10.200.12:18005

listen kong_admin
    mode http
    bind 0.0.0.0:8001
    balance leastconn
    option httpchk GET /status
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 check port 8100
    server Kong-CP01 10.10.200.11:18001
    server Kong-CP02 10.10.200.12:18001

listen kong_manager
    mode http
    bind 0.0.0.0:8002
    balance leastconn
    option httpchk GET /status
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 check port 8100
    server Kong-CP01 10.10.200.11:18002
    server Kong-CP02 10.10.200.12:18002
EOF

systemctl enable --now haproxy
systemctl restart haproxy
systemctl status haproxy --no-pager
```

Ba chi tiết quyết định listen `kong_cluster` chạy hay gãy:

- **`mode tcp`, không `ssl`.** HAProxy chỉ chuyển byte, TLS bắt tay thẳng giữa DP và Kong ở CP. Đây là điều kiện để `cluster_mtls = shared` còn dùng được — thêm một chữ `ssl` vào dòng `bind` là HAProxy terminate TLS, DP không còn xác thực được `CN=kong_clustering` và toàn bộ kênh chết.
- **`timeout client/server 1h`.** Kênh CP–DP là WebSocket sống dài, không phải request ngắn. Để nguyên `30m` của `defaults` thì HAProxy cắt kết nối giữa chừng, DP reconnect liên tục và `config_hash` nhấp nháy.
- **`check port 8100`.** Health check hỏi Status API của Kong chứ không TCP connect vào `18005` — cổng mở nhưng Kong hỏng vẫn qua được TCP check, còn `/status` thì không.

`balance leastconn` cho kênh cluster vì kết nối ở đây là dài hạn, chọn CP đang giữ ít kết nối nhất phản ánh tải thật hơn roundrobin. Admin API và Kong Manager đều stateless — đọc/ghi thẳng vào database dùng chung — nên vào CP nào cũng cho cùng kết quả.

`on-marked-down shutdown-sessions` ở `pg_primary` là chi tiết quan trọng: khi Patroni đổi leader, HAProxy cắt luôn các kết nối cũ để Kong reconnect ngay thay vì treo trên một session trỏ về node đã thành replica.

#### 2.2 — Cấu hình trên hai node DP

Chạy trên **Kong-DP01 và Kong-DP02**:

```bash
apt install -y haproxy

tee /etc/haproxy/haproxy.cfg << 'EOF'
global
    log /dev/log local0
    maxconn 20000
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    retries 2
    timeout client  60s
    timeout server  60s
    timeout connect 5s
    timeout check   3s

listen stats
    mode http
    bind 0.0.0.0:7000
    stats enable
    stats uri /
    stats refresh 5s

listen kong_proxy_http
    mode http
    bind 0.0.0.0:80
    balance roundrobin
    option forwardfor
    option httpchk GET /status
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 check port 8100
    server Kong-DP01 10.10.200.21:8000
    server Kong-DP02 10.10.200.22:8000

listen kong_proxy_https
    mode tcp
    bind 0.0.0.0:443
    balance roundrobin
    option httpchk GET /status
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 check port 8100
    server Kong-DP01 10.10.200.21:8443
    server Kong-DP02 10.10.200.22:8443
EOF

systemctl enable --now haproxy
systemctl restart haproxy
systemctl status haproxy --no-pager

```

`kong_proxy_http` chạy `mode http` để có `option forwardfor` — HAProxy chèn `X-Forwarded-For`, nhờ đó Kong ở Step 11 khôi phục được IP thật của client. `kong_proxy_https` phải là `mode tcp`: TLS do Kong terminate ở `8443`, HAProxy không có cert và cũng không cần có.

**Kết quả mong đợi:** HAProxy sống trên cả 4 node, giữ đúng các cổng công khai.

```bash
systemctl is-active haproxy
netstat -nlpt | grep haproxy
```

Trên một node CP:

```
active
tcp  0  0 0.0.0.0:7000     0.0.0.0:*  LISTEN  xxxx/haproxy
tcp  0  0 0.0.0.0:8005     0.0.0.0:*  LISTEN  xxxx/haproxy
tcp  0  0 0.0.0.0:8001     0.0.0.0:*  LISTEN  xxxx/haproxy
tcp  0  0 0.0.0.0:8002     0.0.0.0:*  LISTEN  xxxx/haproxy
tcp  0  0 127.0.0.1:5000   0.0.0.0:*  LISTEN  xxxx/haproxy
tcp  0  0 127.0.0.1:5001   0.0.0.0:*  LISTEN  xxxx/haproxy
```

Trên một node DP:

```
active
tcp  0  0 0.0.0.0:7000     0.0.0.0:*  LISTEN  xxxx/haproxy
tcp  0  0 0.0.0.0:80       0.0.0.0:*  LISTEN  xxxx/haproxy
tcp  0  0 0.0.0.0:443      0.0.0.0:*  LISTEN  xxxx/haproxy
```

Toàn bộ backend đang `DOWN`, đúng như dự kiến:

```bash
curl -s "http://10.10.200.11:7000/;csv" | awk -F, '$2 ~ /Kong-/ {print $1, $2, $18}'
```

```
pg_primary Kong-CP01 DOWN
pg_primary Kong-CP02 DOWN
pg_primary Kong-DP01 DOWN
kong_cluster Kong-CP01 DOWN
kong_cluster Kong-CP02 DOWN
kong_admin Kong-CP01 DOWN
kong_admin Kong-CP02 DOWN
```

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/01.png)

Trang stats `http://10.10.200.11:7000/` mở bằng trình duyệt cũng thấy y hệt, dễ đọc hơn nhiều. Từ giờ đến hết bài, đây là chỗ nhìn đầu tiên mỗi khi có gì đó không thông.

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/02.png)

### Step 3: keepalived cấp hai VIP

**Mục tiêu:** mỗi cặp node có một địa chỉ duy nhất luôn sống để bên ngoài gọi vào.

Hai VIP dùng `virtual_router_id` khác nhau (51 và 50) và chạy trên hai cặp node tách biệt nên không ảnh hưởng nhau: mất một CP thì client vẫn được phục vụ bình thường, mất một DP thì kênh cấu hình vẫn nguyên.

Cả hai đều track HAProxy chứ không track Kong — Kong chết là việc HAProxy tự xử lý tại chỗ bằng health check, VIP không cần nhảy. Chỉ khi HAProxy chết thì node đó mới thật sự hết khả năng phục vụ.

#### 3.1 — VIP `.51` trên hai node CP

```bash
apt install -y keepalived

# Kong-CP01: state MASTER, priority 110 — Kong-CP02: state BACKUP, priority 100
tee /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
    enable_script_security
    script_user root
}

vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    fall 2
    rise 2
    weight -30
}

vrrp_instance KONG_CP {
    state MASTER            # Kong-CP02: state BACKUP
    interface ens160
    virtual_router_id 51
    priority 110            # Kong-CP02: priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass Kongcp1!
    }
    virtual_ipaddress {
        10.10.200.51/24
    }
    track_script {
        chk_haproxy
    }
}
EOF

systemctl enable --now keepalived
systemctl status keepalived --no-pager
```

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/03.png)

#### 3.2 — VIP `.50` trên hai node DP

```bash
apt install -y keepalived

# Kong-DP01: state MASTER, priority 110 — Kong-DP02: state BACKUP, priority 100
tee /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
    enable_script_security
    script_user root
}

vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    fall 2
    rise 2
    weight -30
}

vrrp_instance KONG_PROXY {
    state MASTER            # Kong-DP02: state BACKUP
    interface ens160
    virtual_router_id 50
    priority 110            # Kong-DP02: priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass Kongdp1!
    }
    virtual_ipaddress {
        10.10.200.50/24
    }
    track_script {
        chk_haproxy
    }
}
EOF

systemctl enable --now keepalived
systemctl status keepalived --no-pager
```

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/04.png)

**Kết quả mong đợi:** mỗi VIP nằm đúng trên node MASTER của cặp nó.

Hai node BACKUP chỉ có IP thật của mình — không thấy VIP nào là **đúng**, VIP chỉ xuất hiện ở đó khi node MASTER tương ứng chết. Kiểm tra nhanh trang stats qua VIP:

```bash
curl -s -o /dev/null -w 'stats qua VIP: HTTP %{http_code}\n' http://10.10.200.51:7000/
```

### Step 4: Dựng etcd cluster 3 node

**Mục tiêu:** dựng DCS 3 member cho Patroni bầu leader.

Chạy trên **Kong-CP01, Kong-CP02, Kong-DP01**:

```bash
ETCD_VER=v3.5.17
curl -fsSL "https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz" -o /tmp/etcd.tar.gz
tar xzf /tmp/etcd.tar.gz -C /tmp
install -m 0755 /tmp/etcd-${ETCD_VER}-linux-amd64/etcd /usr/local/bin/etcd
install -m 0755 /tmp/etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/etcdctl
useradd -r -s /usr/sbin/nologin -d /var/lib/etcd etcd 2>/dev/null || true
mkdir -p /var/lib/etcd /etc/etcd
chown -R etcd:etcd /var/lib/etcd
chmod 700 /var/lib/etcd
```

File config — **đổi `NODE_NAME` và `NODE_IP` cho từng node** (`Kong-CP01`/`10.10.200.11`, `Kong-CP02`/`10.10.200.12`, `Kong-DP01`/`10.10.200.21`):

```bash
NODE_NAME=Kong-CP01
NODE_IP=10.10.200.11

tee /etc/etcd/etcd.conf.yml << EOF
name: ${NODE_NAME}
data-dir: /var/lib/etcd
listen-peer-urls: http://${NODE_IP}:2380
listen-client-urls: http://${NODE_IP}:2379,http://127.0.0.1:2379
initial-advertise-peer-urls: http://${NODE_IP}:2380
advertise-client-urls: http://${NODE_IP}:2379
initial-cluster: Kong-CP01=http://10.10.200.11:2380,Kong-CP02=http://10.10.200.12:2380,Kong-DP01=http://10.10.200.21:2380
initial-cluster-token: kong-pg-etcd
initial-cluster-state: new
enable-v2: false
auto-compaction-retention: "1"
EOF
```

Unit systemd:

```bash
tee /etc/systemd/system/etcd.service << 'EOF'
[Unit]
Description=etcd for Patroni
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=etcd
ExecStart=/usr/local/bin/etcd --config-file /etc/etcd/etcd.conf.yml
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now etcd
systemctl status etcd --no-pager
```

Start cả 3 node rồi kiểm tra:

```bash
etcdctl --endpoints=10.10.200.11:2379,10.10.200.12:2379,10.10.200.21:2379 endpoint status -w table
etcdctl --endpoints=10.10.200.11:2379 member list -w table
```

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/05.png)

Sáu điểm phải khớp trước khi đi tiếp:

| Kiểm tra | Vì sao |
|----------|--------|
| Cả 3 endpoint `started` | Thiếu một node là quorum chỉ còn 2/3, mất thêm một node nữa thì Patroni đứng |
| Đúng **một** node `IS LEADER = true` | Hai leader nghĩa là split-brain, thường do `initial-cluster-token` khác nhau giữa các node |
| `IS LEARNER` toàn `false` | Learner không được tính vào quorum |
| `RAFT INDEX` = `RAFT APPLIED INDEX` và **bằng nhau trên cả 3 node** | Chênh nhau là có node đang lag, thường do disk chậm |
| Cột `ERRORS` rỗng | — |
| `NAME` khớp đúng hostname, `PEER ADDRS` khớp đúng IP từng node | Sai chỗ này là do copy `etcd.conf.yml` mà quên đổi `NODE_NAME`/`NODE_IP` |

Cụm còn trống trơn ở bước này — `DB SIZE` khoảng 20 kB và `RAFT INDEX` một con số rất nhỏ. Client đầu tiên ghi vào nó là Patroni ở Step 5.

### Step 5: PostgreSQL 16 và Patroni

**Mục tiêu:** cụm PostgreSQL 16 một leader hai replica, Patroni quản lý toàn bộ lifecycle.

Chạy trên **Kong-CP01, Kong-CP02, Kong-DP01**. Noble có sẵn PostgreSQL 16 nên không cần thêm repo PGDG:

```bash
apt install -y postgresql-16 postgresql-client-16 python3-venv python3-dev libpq-dev gcc
```

PostgreSQL vừa cài tự tạo một cluster `main` và tự start. Patroni phải là bên duy nhất điều khiển PostgreSQL, nên xóa cluster đó đi — Patroni sẽ tự `initdb` lại:

```bash
systemctl disable --now postgresql
pg_dropcluster 16 main
mkdir -p /var/lib/postgresql/16/main /etc/patroni
chown -R postgres:postgres /var/lib/postgresql/16
chmod 700 /var/lib/postgresql/16/main
```

Kiểm tra ngay để chắc chắn — `pg_lsclusters` không còn dòng nào và thư mục rỗng hoàn toàn:

```bash
pg_lsclusters
ls -lA /var/lib/postgresql/16/main
```

Cài Patroni trong venv — Ubuntu 24.04 áp PEP 668 nên `pip install` thẳng vào system Python sẽ bị chặn:

```bash
python3 -m venv /opt/patroni
/opt/patroni/bin/pip install --upgrade pip
/opt/patroni/bin/pip install "patroni[etcd3]" psycopg2-binary
ln -sf /opt/patroni/bin/patroni /usr/local/bin/patroni
ln -sf /opt/patroni/bin/patronictl /usr/local/bin/patronictl
patroni --version
```

Config Patroni. **Đổi `name`, `listen`, `connect_address` cho từng node**, và trên **Kong-DP01 phải đặt `nofailover: true`**:

```bash
NODE_NAME=Kong-CP01
NODE_IP=10.10.200.11
NOFAILOVER=false      # trên Kong-DP01 đổi thành true

tee /etc/patroni/patroni.yml << EOF
scope: kong-pg
namespace: /service/
name: ${NODE_NAME}

restapi:
  listen: ${NODE_IP}:8008
  connect_address: ${NODE_IP}:8008

etcd3:
  hosts:
  - 10.10.200.11:2379
  - 10.10.200.12:2379
  - 10.10.200.21:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    synchronous_mode: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        max_connections: 300
        shared_buffers: 1GB
        wal_level: replica
        wal_log_hints: "on"
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        password_encryption: scram-sha-256
  initdb:
  - encoding: UTF8
  - data-checksums
  pg_hba:
  - local all all peer
  - host replication replicator 10.10.200.0/24 scram-sha-256
  - host all all 127.0.0.1/32 scram-sha-256
  - host all all 10.10.200.0/24 scram-sha-256

postgresql:
  listen: 0.0.0.0:5432
  connect_address: ${NODE_IP}:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  pgpass: /var/lib/postgresql/.pgpass_patroni
  authentication:
    replication:
      username: replicator
      password: 'Zxcasd123!@#'
    superuser:
      username: postgres
      password: 'Zxcasd123!@#'
    rewind:
      username: rewind_user
      password: 'Zxcasd123!@#'
  parameters:
    unix_socket_directories: /var/run/postgresql

tags:
  nofailover: ${NOFAILOVER}
  noloadbalance: false
  clonefrom: false
  nosync: false
EOF

chown postgres:postgres /etc/patroni/patroni.yml
chmod 600 /etc/patroni/patroni.yml
```

Password chứa `#` nên phải để trong dấu nháy đơn của YAML, không thì phần từ `#` trở đi bị coi là comment.

Unit systemd:

```bash
tee /etc/systemd/system/patroni.service << 'EOF'
[Unit]
Description=Patroni PostgreSQL HA
After=network-online.target etcd.service
Wants=network-online.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/opt/patroni/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now patroni
systemctl status patroni --no-pager
```

Start **Kong-CP01 trước** để nó bootstrap cluster và trở thành leader, đợi khoảng 30 giây rồi mới start Kong-CP02 và Kong-DP01 — hai node này sẽ `pg_basebackup` từ leader.

```bash
patronictl -c /etc/patroni/patroni.yml list
```

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/06.png)

Năm điểm phải khớp:

| Kiểm tra | Vì sao |
|----------|--------|
| Đúng một `Leader`, hai `Replica` ở state `streaming` | `State` là `starting` hoặc `creating replica` thì cứ đợi — `pg_basebackup` mất vài chục giây |
| Cột `TL` **giống nhau cả 3 node** | TL là timeline của PostgreSQL. Replica có TL thấp hơn leader nghĩa là nó bị bỏ lại sau một lần promote và phải `pg_rewind` |
| `Receive LSN` = `Replay LSN` và giống nhau giữa hai replica | Replica đã apply hết WAL nhận được, không tồn đọng |
| Cả hai cột `Lag` bằng `0` | Cột `Lag` đầu là lag lúc nhận WAL, cột sau là lag lúc replay. Chỉ một trong hai lớn hơn 0 cũng đủ để replica không đủ tươi khi failover |
| **Chỉ `Kong-DP01` có `Tags: nofailover: true`** | Đây chính là thiết kế của lab. Tag lọt sang node CP là bạn tự loại một CP khỏi khả năng làm leader |

Leader để trống cả 4 cột LSN/Lag là bình thường — nó là bên phát WAL, không có gì để nhận hay replay.

Cụm etcd dựng ở Step 4 giờ đã có dữ liệu — xác nhận Patroni ghi được vào đúng nơi:

```bash
etcdctl --endpoints=10.10.200.11:2379 get /service/kong-pg/ --prefix --keys-only
```

Phải thấy `leader`, `config` và ba key `members/Kong-CP01`, `.../Kong-CP02`, `.../Kong-DP01`. Đây là cách xác nhận Patroni dùng đúng **etcd v3 API**: `etcd.conf.yml` đặt `enable-v2: false`, khớp với section `etcd3:` trong `patroni.yml`. Dùng nhầm section `etcd:` (v2) thì Patroni không bootstrap được.

Giờ nhìn sang HAProxy — backend database đã tự chuyển sang `UP` mà không phải sửa gì:

```bash
curl -s "http://10.10.200.11:7000/;csv" | awk -F, '$1 ~ /^pg_/ && $2 ~ /Kong-/ {print $1, $2, $18}'
```

```
pg_primary Kong-CP01 UP
pg_primary Kong-CP02 DOWN
pg_primary Kong-DP01 DOWN
pg_replica Kong-CP01 DOWN
pg_replica Kong-CP02 UP
pg_replica Kong-DP01 UP
```

Đúng một node `UP` ở `pg_primary` — chính là leader. Hai replica `UP` ở `pg_replica`. `DOWN` ở đây không phải lỗi: nó có nghĩa là node đó trả `503` cho `/primary`, tức nó không phải leader. Khi Patroni failover, hai cột này tự đảo chỗ và Kong không cần biết.

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/07.png)

### Step 6: Tạo user và database kong

**Mục tiêu:** tạo chỗ chứa cấu hình cho Kong. Chỉ làm **một lần, trên Kong-CP01**.

Kết nối qua HAProxy `127.0.0.1:5000` chứ không nối thẳng vào PostgreSQL — cách này vừa kiểm chứng luôn endpoint mà Kong sẽ dùng ở Step 10:

```bash
PGPASSWORD='Zxcasd123!@#' psql -h 127.0.0.1 -p 5000 -U postgres << 'EOF'
CREATE USER kong WITH PASSWORD 'Zxcasd123!@#';
CREATE DATABASE kong OWNER kong;
EOF
```

**Kết quả mong đợi:**

```bash
PGPASSWORD='Zxcasd123!@#' psql -h 127.0.0.1 -p 5000 -U postgres -c "\l kong" -c "\du kong"
```

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/08.png)

Database `kong` phải thuộc owner `kong` — nếu owner là `postgres` thì migrations ở Step 9 sẽ fail khi tạo schema.

Xác nhận thêm là HAProxy đang trỏ đúng leader:

```bash
PGPASSWORD='Zxcasd123!@#' psql -h 127.0.0.1 -p 5000 -U postgres -Atc "select inet_server_addr(), pg_is_in_recovery();"
PGPASSWORD='Zxcasd123!@#' psql -h 127.0.0.1 -p 5001 -U postgres -Atc "select inet_server_addr(), pg_is_in_recovery();"
```

```
10.10.200.11|f
10.10.200.12|t
```

`pg_is_in_recovery()` trả `f` ở port `5000` nghĩa là đang nối vào leader; port `5001` trả `t` nghĩa là replica. Hai giá trị này mà giống nhau là cấu hình `option httpchk` đang sai.

### Step 7: Cài Kong 3.9.1 lên cả 4 node

**Mục tiêu:** có binary Kong, thư mục `/etc/kong` và user `kong` trên **cả bốn** node trước khi đụng tới cert hay cấu hình.

Chạy trên **cả 4 node**:

```bash
curl -1sLf 'https://packages.konghq.com/public/gateway-39/setup.deb.sh' | bash
apt install -y kong=3.9.1
kong version
```

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/09.png)

Nếu repo script không nhận diện được noble, tải trực tiếp `.deb`:

```bash
curl -Lo /tmp/kong-3.9.1.deb "https://packages.konghq.com/public/gateway-39/deb/ubuntu/pool/noble/main/k/ko/kong_3.9.1/kong_3.9.1_$(dpkg --print-architecture).deb"
apt install -y /tmp/kong-3.9.1.deb
```

**Kết quả mong đợi:** package đã tạo user và thư mục, nhưng chưa có service nào chạy — Kong chưa có cấu hình để mà start.

```bash
id kong
ls -ld /etc/kong
systemctl is-enabled kong
```

```
uid=996(kong) gid=996(kong) groups=996(kong)
drwxr-xr-x 2 root root 4096 Jul 28 16:20 /etc/kong
disabled
```

### Step 8: Chứng chỉ cho kênh mTLS

**Mục tiêu:** một cặp cert dùng chung cho toàn bộ CP và DP, `CN` phải đúng `kong_clustering`.

`cluster_mtls` mặc định là `shared`: hai đầu xác thực nhau bằng **chính cặp cert giống hệt nhau**, nên phải copy cả `tls.crt` lẫn `tls.key` sang mọi node.

Sinh cert trên **Kong-CP01**:

```bash
mkdir -p /etc/kong/certs
openssl req -new -x509 -nodes \
  -newkey ec -pkeyopt ec_paramgen_curve:secp384r1 -pkeyopt ec_param_enc:named_curve \
  -keyout /etc/kong/certs/tls.key -out /etc/kong/certs/tls.crt \
  -days 1095 -subj "/CN=kong_clustering"
openssl x509 -in /etc/kong/certs/tls.crt -noout -subject -dates
```

Copy sang 3 node còn lại. SSH key đã làm ở Step 1 nên không phải gõ password:

```bash
for h in Kong-CP02 Kong-DP01 Kong-DP02; do
  tar -C /etc/kong -cf - certs | ssh $h "tar -C /etc/kong -xf -"
done
```

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/10.png)

Chỉnh quyền trên **cả 4 node**:

```bash
chown -R kong:kong /etc/kong/certs
chmod 600 /etc/kong/certs/tls.key
chmod 644 /etc/kong/certs/tls.crt
ls -l /etc/kong/certs
```

```
-rw-r--r-- 1 kong kong  from  ... tls.crt
-rw------- 1 kong kong  from  ... tls.key
```

Owner sai ở đây là một lỗi im lặng khó chịu: Kong start được nhưng handshake với DP fail, vì worker chạy dưới user `kong` không đọc nổi `tls.key`.

**Kết quả mong đợi:** cả 4 node cùng một fingerprint.

```bash
openssl x509 -in /etc/kong/certs/tls.crt -noout -fingerprint -sha256
```

### Step 9: Dựng schema bằng migrations

**Mục tiêu:** tạo toàn bộ bảng của Kong trong database `kong`. Chỉ chạy **một lần, trên Kong-CP01**.

Password truyền qua biến môi trường chứ **không** đặt trong `kong.conf` — ký tự `#` trong `Zxcasd123!@#` sẽ bị Kong parse thành comment và bạn sẽ nhận một lỗi authentication rất khó lần:

```bash
env KONG_DATABASE=postgres \
    KONG_PG_HOST=127.0.0.1 \
    KONG_PG_PORT=5000 \
    KONG_PG_USER=kong \
    KONG_PG_PASSWORD='Zxcasd123!@#' \
    KONG_PG_DATABASE=kong \
    kong migrations bootstrap
```

**Kết quả mong đợi:**

```
Bootstrapping database...
migrating core on database 'kong'...
core migrated up to: 000_base (executed)
core migrated up to: 003_100_to_110 (executed)
...
core migrated up to: 024_380_to_390 (executed)
migrating acl on database 'kong'...
...
migrating session on database 'kong'...
session migrated up to: 002_320_to_330 (executed)
67 migrations processed
67 executed
Database is up-to-date
```

`bootstrap` chỉ chạy đúng một lần cho cả cụm. Kong-CP02 dùng chung database này nên **tuyệt đối không chạy lại** trên Kong-CP02.


### Step 10: Cấu hình Control Plane

**Mục tiêu:** hai node CP chạy `role = control_plane`, không proxy traffic, nghe trên cổng nội bộ `18001`/`18002`/`18005` để HAProxy ở Step 2 chia tải vào.

#### 10.1 — kong.conf

Chạy trên **cả Kong-CP01 và Kong-CP02**, nội dung giống hệt nhau:

```bash
tee /etc/kong/kong.conf << 'EOF'
role = control_plane

database = postgres
pg_host = 127.0.0.1
pg_port = 5000
pg_user = kong
pg_database = kong
pg_ssl = off

cluster_cert = /etc/kong/certs/tls.crt
cluster_cert_key = /etc/kong/certs/tls.key
cluster_listen = 0.0.0.0:18005

# CP không nhận traffic của client
proxy_listen = off

admin_listen = 0.0.0.0:18001
admin_gui_listen = 0.0.0.0:18002
admin_gui_url = http://10.10.200.51:8002
admin_gui_api_url = http://10.10.200.51:8001

status_listen = 0.0.0.0:8100
log_level = notice
EOF
```

| Tham số | Vì sao đặt như vậy |
|---------|--------------------|
| `pg_host = 127.0.0.1` / `pg_port = 5000` | Trỏ vào HAProxy local của Step 2, không trỏ thẳng IP node PostgreSQL. Leader đổi thì Kong không cần biết. |
| `proxy_listen = off` | CP chỉ quản lý cấu hình, traffic của client do DP xử lý. Bật proxy trên CP là sai mô hình hybrid. |
| `cluster_listen = 0.0.0.0:18005` | Cổng nội bộ, chỉ HAProxy gọi vào. Trong OSS đây là **kênh duy nhất** giữa CP và DP; `cluster_telemetry_listen` (8006) là tham số của bản Enterprise, khai trong OSS thì Kong bỏ qua và cổng không bao giờ mở. |
| `admin_listen` / `admin_gui_listen` dùng `18001`/`18002` | Nhường `8001`/`8002` cho HAProxy. Đây là điều kiện để hai tiến trình cùng sống trên một node. |
| `admin_gui_url` / `admin_gui_api_url` trỏ `10.10.200.51` | Link mà Kong Manager sinh ra cho trình duyệt phải là địa chỉ **công khai** — VIP, cổng 8002/8001 — chứ không phải cổng nội bộ. |
| `status_listen = 0.0.0.0:8100` | Cổng này HAProxy không proxy mà dùng làm health check, gọi thẳng vào IP từng node. |

Không có `pg_password` trong file này — cố ý, xem 10.2.

#### 10.2 — Password và thứ tự khởi động, qua systemd drop-in

`kong.conf` không chứa password vì Kong parse `#` thành comment. Đưa qua biến môi trường của service, chạy trên **cả hai CP**:

```bash
mkdir -p /etc/systemd/system/kong.service.d
tee /etc/systemd/system/kong.service.d/override.conf << 'EOF'
[Unit]
After=haproxy.service
Wants=haproxy.service

[Service]
Environment=KONG_PG_PASSWORD=Zxcasd123!@#
EOF

chmod 600 /etc/systemd/system/kong.service.d/override.conf
systemctl daemon-reload
```

Biến môi trường `KONG_*` luôn thắng giá trị cùng tên trong `kong.conf`, nên đây cũng là cách override nhanh bất kỳ tham số nào mà không phải sửa file config.

Khối `[Unit]` xử lý một phụ thuộc dễ quên: **trên CP, Kong không sống được nếu HAProxy chưa chạy**, vì `pg_host = 127.0.0.1:5000` chính là HAProxy. Thiếu nó Kong chết ngay lúc khởi tạo:

```
init_by_lua error: [PostgreSQL error] failed to retrieve PostgreSQL server_version_num: connection refused
```

Gặp thông báo này thì đừng đi tìm lỗi ở Kong hay ở PostgreSQL — kiểm tra `systemctl is-active haproxy` trước tiên.

#### 10.3 — Khởi động và kiểm tra

Trên **cả hai CP**:

```bash
systemctl enable --now kong
systemctl status kong --no-pager
```

```
● kong.service - Kong
     Loaded: loaded (/usr/lib/systemd/system/kong.service; enabled; preset: enabled)
    Drop-In: /etc/systemd/system/kong.service.d
             └─override.conf
     Active: active (running) since Tue 2026-07-28 16:33:54 +07; 3s ago
    Process: 73662 ExecStartPre=/usr/local/bin/kong prepare -p /usr/local/kong (code=exited, status=0/SUCCESS)
   Main PID: 73681 (nginx)
```

Dòng `Drop-In` phải xuất hiện — không có nghĩa là systemd chưa nạp `override.conf`, và Kong sẽ fail authentication với PostgreSQL.

Phân chia cổng giữa hai tiến trình phải đúng như thiết kế:

```bash
netstat -nlpt | grep -E 'nginx|haproxy'
```

**Kết quả mong đợi:** gọi Admin API qua VIP, và thấy cả hai CP luân phiên trả lời.

```bash
curl -s http://10.10.200.51:8001/ | jq '{version, hostname, "role": .configuration.role, "db": .configuration.database}'
```

```json
{
  "version": "3.9.1",
  "hostname": "Kong-CP01",
  "role": "control_plane",
  "db": "postgres"
}
```

```bash
for i in $(seq 6); do curl -s http://10.10.200.51:8001/ | jq -r .hostname; done
```

```
Kong-CP01
Kong-CP02
Kong-CP01
Kong-CP02
Kong-CP01
Kong-CP02
```

Ra cùng một hostname cả 6 lần nghĩa là một CP đang bị HAProxy đánh dấu DOWN — mở trang stats để biết lý do:

```bash
curl -s "http://10.10.200.11:7000/;csv" | awk -F, '$1 ~ /^kong_/ && $2 ~ /Kong-/ {print $1, $2, $18}'
```

```
kong_cluster Kong-CP01 UP
kong_cluster Kong-CP02 UP
kong_admin Kong-CP01 UP
kong_admin Kong-CP02 UP
kong_manager Kong-CP01 UP
kong_manager Kong-CP02 UP
```

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/11.png)

Kiểm tra chốt: hai CP dùng chung một database nên là active/active — tạo cấu hình ở CP nào cũng phải thấy ở CP kia.

```bash
curl -s -X POST http://10.10.200.11:8001/services \
  --data name=probe --data url=http://10.10.200.11:9001 -o /dev/null

curl -s http://10.10.200.12:8001/services/probe | jq '{name, host, port}'
```

```json
{
  "name": "probe",
  "host": "10.10.200.11",
  "port": 9001
}
```

Kong không kiểm tra upstream có sống hay không lúc tạo service, nên `:9001` chưa có gì nghe cũng không sao — ở đây chỉ cần biết bản ghi ghi ở CP01 có đọc được từ CP02. Lưu ý hai lệnh trên cố tình gọi thẳng IP node (`.11` và `.12`) chứ không qua VIP, để chắc chắn mỗi lệnh chạm đúng node mình muốn.

Xoá service thử nghiệm:

```bash
curl -s -X DELETE http://10.10.200.51:8001/services/probe -o /dev/null -w '%{http_code}\n'
```

```
204
```

Nếu CP02 trả `404` cho `/services/probe`, hai node đang không nhìn cùng một database — kiểm tra `pg_host`/`pg_port` trong `kong.conf` của CP02 và trạng thái `pg_primary` trên trang stats của node đó.

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/12.png)

### Step 11: Cấu hình Data Plane

**Mục tiêu:** hai node DP chạy `database = off`, kéo toàn bộ cấu hình từ CP qua VIP `.51` cổng `8005`.

#### 11.1 — kong.conf

VIP `.51` đã có từ Step 3 nên DP trỏ thẳng vào đó ngay từ đầu, không cần bước sửa lại về sau. Chạy trên **cả hai DP**:

```bash
tee /etc/kong/kong.conf << 'EOF'
role = data_plane
database = off

cluster_cert = /etc/kong/certs/tls.crt
cluster_cert_key = /etc/kong/certs/tls.key
cluster_control_plane = 10.10.200.51:8005
lua_ssl_trusted_certificate = /etc/kong/certs/tls.crt

proxy_listen = 0.0.0.0:8000 reuseport backlog=16384, 0.0.0.0:8443 http2 ssl reuseport backlog=16384

# DP không có Admin API — cấu hình chỉ đến từ CP
admin_listen = off
status_listen = 0.0.0.0:8100

# Giữ IP thật của client khi đi qua HAProxy
trusted_ips = 10.10.200.0/24
real_ip_header = X-Forwarded-For
real_ip_recursive = on

log_level = notice
EOF

systemctl enable --now kong
systemctl status kong --no-pager

```

| Tham số | Vì sao đặt như vậy |
|---------|--------------------|
| `database = off` | DP giữ cấu hình trong LMDB dưới `/usr/local/kong`, không có kết nối database nào. Đây là điểm mấu chốt của hybrid mode: database chết thì DP vẫn phục vụ traffic. |
| `cluster_control_plane = 10.10.200.51:8005` | VIP của cặp CP, cổng công khai do HAProxy giữ. DP không bao giờ biết tên hay IP của một CP cụ thể. |
| `lua_ssl_trusted_certificate` | Cert tự ký nên phải khai chính nó làm CA tin cậy, không thì DP từ chối chứng chỉ CP đưa ra. |
| `admin_listen = off` | Không có Admin API trên DP — mọi thay đổi cấu hình phải đi qua CP. Đây cũng là lớp bảo vệ: node hứng traffic không mở cổng quản trị. |
| `proxy_listen ... reuseport` | `reuseport` cho mỗi nginx worker một socket riêng, phân phối kết nối đều hơn khi tải cao. |
| `trusted_ips` + `real_ip_header` | Traffic tới DP đi qua HAProxy, không có hai dòng này thì log và plugin rate-limiting chỉ thấy IP của HAProxy. |

DP **không** cần drop-in `KONG_PG_PASSWORD` và cũng không chạy migrations — chúng không mở kết nối nào tới PostgreSQL. Kong-DP01 tình cờ có PostgreSQL chạy sẵn (nó là replica `nofailover` từ Step 5), nhưng Kong trên node đó tuyệt đối không đụng vào.

```bash
netstat -nlpt | grep nginx
```

8000 và 8443 xuất hiện **4 dòng mỗi cổng** — đó là `reuseport` đang làm việc: mỗi nginx worker giữ một socket riêng trên cùng cổng, kernel tự chia kết nối thay vì để các worker giành nhau một hàng đợi accept. `nginx_worker_processes` mặc định `auto` nên số worker bằng số core — VM lab 4 vCPU ra 4 dòng. Cổng 8100 không có `reuseport` nên chỉ một dòng.

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/13.png)

#### 11.2 — Kiểm tra kênh sync từ phía CP

**Kết quả mong đợi** — hỏi CP xem có bao nhiêu DP đã kết nối:

```bash
curl -s http://10.10.200.51:8001/clustering/data-planes | jq '.data[] | {hostname, ip, version, sync_status, config_hash}'
```

```json
{
  "hostname": "Kong-DP01",
  "ip": "10.10.200.11",
  "version": "3.9.1",
  "sync_status": "normal",
  "config_hash": "2e9cee4c1b6b54bdb2e57e6485bcf121"
}
{
  "hostname": "Kong-DP02",
  "ip": "10.10.200.11",
  "version": "3.9.1",
  "sync_status": "normal",
  "config_hash": "2e9cee4c1b6b54bdb2e57e6485bcf121"
}
```

Hai DP phải có **cùng một `config_hash`** — bằng nhau nghĩa là cả hai đang chạy đúng một phiên bản cấu hình. Cùng với `sync_status: normal` và đủ hai `hostname`, đó là ba trường cần soi ở đây.

Trường `ip` thì đừng trông đợi thấy `.21` và `.22`: **cả hai DP đều báo `10.10.200.11`**, và đó là kết quả đúng. `kong_cluster` chạy `mode tcp` không kèm `send-proxy`, nên HAProxy mở một kết nối TCP mới tới `18005` của CP với source IP của chính node đang giữ VIP. CP chỉ đọc được peer address của socket. Đây là metadata hiển thị chứ không phải cơ chế định danh — Kong nhận diện DP bằng client cert ở Step 8, và `hostname` vẫn phân biệt được vì DP tự khai tên ở tầng ứng dụng.

Kênh cluster cũng phải chia đôi, mỗi DP nối vào một CP khác nhau:

```bash
curl -s "http://10.10.200.11:7000/;csv" | awk -F, '$1 == "kong_cluster" {print $1, $2, $5}'
```

```
kong_cluster FRONTEND 1
kong_cluster Kong-CP01 1
kong_cluster Kong-CP02 1
kong_cluster BACKEND 2
```

Cột thứ ba là số kết nối đang mở (`scur`), tổng ở `BACKEND` bằng số DP đang kết nối.

Đừng trông đợi hai kênh chia đều mỗi CP một cái — rất thường gặp cảnh cả hai dồn vào cùng một CP, CP kia bằng `0`. `leastconn` chỉ cân nhắc **tại thời điểm mở kết nối mới**, mà kênh cluster lại sống rất lâu, nên chúng nằm im ở đó tới lần đứt tiếp theo. Cụm vẫn HA đầy đủ: mất CP đang giữ kênh thì cả hai DP tự nối sang CP còn lại, đúng như test 4b chứng minh.


#### 11.3 — Kiểm tra từ phía DP

Hỏi `/status` của cả bốn node để thấy hai vai trò khác nhau ra sao:

```bash
for ip in 10.10.200.21 10.10.200.22 10.10.200.11 10.10.200.12; do
  echo -n "$ip → "; curl -s http://$ip:8100/status | jq -c '{database, configuration_hash}'
done
```

```
10.10.200.21 → {"database":null,"configuration_hash":"2e9cee4c1b6b54bdb2e57e6485bcf121"}
10.10.200.22 → {"database":null,"configuration_hash":"2e9cee4c1b6b54bdb2e57e6485bcf121"}
10.10.200.11 → {"database":{"reachable":true},"configuration_hash":null}
10.10.200.12 → {"database":{"reachable":true},"configuration_hash":null}
```

Hai vai trò đối xứng nhau hoàn hảo. DP trả `database: null` — không có khái niệm kết nối database để mà báo cáo — nhưng có `configuration_hash` vì cấu hình của nó là snapshot nhận từ CP. CP thì ngược lại: `reachable: true`, còn `configuration_hash` là `null` vì nó đọc thẳng từ database, không giữ snapshot nào để băm. Một lệnh `/status` là biết node đóng vai gì.

Cuối cùng là bài test thật: tạo route trên CP rồi xem DP có nhận không.

```bash
CP=http://10.10.200.51:8001
curl -s -X POST $CP/services --data name=synctest --data url=http://10.10.200.11:9001 -o /dev/null
curl -s -X POST $CP/services/synctest/routes --data name=synctest-route --data 'paths[]=/synctest' -o /dev/null

# Gọi thẳng vào từng DP, không qua HAProxy
curl -s -o /dev/null -w '%{http_code}\n' http://10.10.200.21:8000/synctest
curl -s -o /dev/null -w '%{http_code}\n' http://10.10.200.22:8000/synctest
```

```
502
502
```

`502` mới là kết quả **đúng** ở bước này: DP đã biết route `/synctest` và đã cố gọi upstream `10.10.200.11:9001` — nơi chưa có gì lắng nghe vì `demo-api` mãi Step 12 mới dựng. Nhận `404` thì cấu hình chưa sync tới DP, quay lại 11.2.

Dọn dẹp:

```bash
curl -s -X DELETE $CP/routes/synctest-route -o /dev/null -w '%{http_code}\n'
curl -s -X DELETE $CP/services/synctest -o /dev/null -w '%{http_code}\n'
```

```
204
204
```

Route phải xoá trước service — Kong từ chối xoá một service còn route trỏ vào.

### Step 12: Sample app backend

**Mục tiêu:** hai upstream target thật phục vụ một REST API CRUD, biết mình là node nào, và bật/tắt được trạng thái health để test active health check của Kong.

`demo-api` viết bằng thư viện chuẩn của Python 3.12 — không cần pip, không cần Docker. Chạy trên **Kong-CP01 và Kong-CP02**:

```bash
useradd -r -s /usr/sbin/nologin -d /opt/demo-api demoapi 2>/dev/null || true
mkdir -p /opt/demo-api

tee /opt/demo-api/app.py << 'PYEOF'
#!/usr/bin/env python3
import itertools, json, os, socket, threading, time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlparse, parse_qs

NODE = os.environ.get("NODE_NAME", socket.gethostname())
PORT = int(os.environ.get("PORT", "9001"))
STARTED = time.time()
HEALTHY = {"ok": True}
FIELDS = ("sku", "name", "price")
LOCK = threading.Lock()
SEQ = itertools.count(4)
PRODUCTS = {
    1: {"id": 1, "sku": "SRV-R650", "name": "Dell PowerEdge R650", "price": 4200},
    2: {"id": 2, "sku": "SW-C9300", "name": "Cisco Catalyst 9300", "price": 3100},
    3: {"id": 3, "sku": "FW-PA440", "name": "Palo Alto PA-440", "price": 1800},
}


class Handler(BaseHTTPRequestHandler):
    protocol_version = "HTTP/1.1"
    server_version = "demo-api/2.0"

    def _send(self, code, payload=None):
        body = b"" if payload is None else json.dumps(payload, ensure_ascii=False).encode()
        self.send_response(code)
        if body:
            self.send_header("Content-Type", "application/json; charset=utf-8")
        if body or code != 204:
            self.send_header("Content-Length", str(len(body)))
        self.send_header("X-Upstream-Node", NODE)
        self.end_headers()
        self.wfile.write(body)

    def _body(self):
        n = int(self.headers.get("Content-Length") or 0)
        try:
            return json.loads(self.rfile.read(n)) if n else {}
        except json.JSONDecodeError:
            return None

    def _pid(self):
        tail = urlparse(self.path).path.rstrip("/").rsplit("/", 1)[-1]
        return int(tail) if tail.isdigit() else None

    def _err(self, code, msg):
        return self._send(code, {"node": NODE, "error": msg})

    def log_message(self, fmt, *args):
        print("%s %s" % (self.address_string(), fmt % args), flush=True)

    def do_GET(self):
        u = urlparse(self.path)
        path = u.path.rstrip("/") or "/"

        if path == "/healthz":
            if not HEALTHY["ok"]:
                return self._send(503, {"status": "unhealthy", "node": NODE})
            return self._send(200, {"status": "ok", "node": NODE,
                                    "uptime_s": round(time.time() - STARTED, 1)})

        if path == "/whoami":
            return self._send(200, {"node": NODE, "port": PORT,
                                    "client": self.client_address[0],
                                    "headers": dict(self.headers)})

        if path == "/api/v1/products":
            with LOCK:
                items = sorted(PRODUCTS.values(), key=lambda p: p["id"])
            return self._send(200, {"node": NODE, "count": len(items), "data": items})

        if path.startswith("/api/v1/products/"):
            with LOCK:
                item = PRODUCTS.get(self._pid())
            if not item:
                return self._err(404, "product not found")
            return self._send(200, {"node": NODE, "data": item})

        if path == "/slow":
            ms = min(int(parse_qs(u.query).get("ms", ["3000"])[0]), 30000)
            time.sleep(ms / 1000)
            return self._send(200, {"node": NODE, "slept_ms": ms})

        if path == "/boom":
            return self._err(500, "simulated failure")

        return self._err(404, "no route")

    def do_POST(self):
        path = urlparse(self.path).path.rstrip("/") or "/"

        if path in ("/admin/healthy", "/admin/unhealthy"):
            HEALTHY["ok"] = path.endswith("/healthy")
            return self._send(200, {"node": NODE, "healthy": HEALTHY["ok"]})

        if path != "/api/v1/products":
            return self._err(404, "no route")

        p = self._body()
        if p is None or any(f not in p for f in FIELDS):
            return self._err(400, "can du sku, name, price")

        with LOCK:
            item = {"id": next(SEQ)}
            item.update({f: p[f] for f in FIELDS})
            PRODUCTS[item["id"]] = item
            snap = dict(item)
        return self._send(201, {"node": NODE, "data": snap})

    def _update(self, full):
        if not urlparse(self.path).path.rstrip("/").startswith("/api/v1/products/"):
            return self._err(404, "no route")

        p = self._body()
        if p is None or (full and any(f not in p for f in FIELDS)):
            return self._err(400, "body khong hop le")

        with LOCK:
            item = PRODUCTS.get(self._pid())
            if not item:
                return self._err(404, "product not found")
            item.update({f: p[f] for f in FIELDS if f in p})
            snap = dict(item)
        return self._send(200, {"node": NODE, "data": snap})

    def do_PUT(self):
        self._update(True)

    def do_PATCH(self):
        self._update(False)

    def do_DELETE(self):
        if not urlparse(self.path).path.rstrip("/").startswith("/api/v1/products/"):
            return self._err(404, "no route")
        with LOCK:
            gone = PRODUCTS.pop(self._pid(), None)
        return self._send(204) if gone else self._err(404, "product not found")


if __name__ == "__main__":
    print("demo-api listening on 0.0.0.0:%d as %s" % (PORT, NODE), flush=True)
    ThreadingHTTPServer(("0.0.0.0", PORT), Handler).serve_forever()
PYEOF

chown -R demoapi:demoapi /opt/demo-api
```

CRUD trên tài nguyên `products`, mỗi bản ghi gồm `sku`, `name`, `price`:

| Method | Path | Trả về | Ghi chú |
|--------|------|--------|---------|
| GET | `/api/v1/products` | `200` | Danh sách, sắp theo `id` |
| POST | `/api/v1/products` | `201` | `400` nếu thiếu trường |
| GET | `/api/v1/products/{id}` | `200` / `404` | |
| PUT | `/api/v1/products/{id}` | `200` / `404` | Thay toàn bộ, bắt buộc đủ 3 trường |
| PATCH | `/api/v1/products/{id}` | `200` / `404` | Sửa một phần |
| DELETE | `/api/v1/products/{id}` | `204` / `404` | |

Ngoài ra là nhóm endpoint phục vụ riêng việc test gateway:

| Method | Path | Dùng để |
|--------|------|---------|
| GET | `/healthz` | Active health check của Kong |
| GET | `/whoami` | Xem node nào phục vụ, kèm toàn bộ header Kong thêm vào |
| GET | `/slow?ms=5000` | Test timeout của Kong |
| GET | `/boom` | Trả `500` để test retry |
| POST | `/admin/unhealthy` | Ép `/healthz` trả `503` |
| POST | `/admin/healthy` | Trả lại bình thường |

Hai điểm trong code đáng để ý. `LOCK` bọc mọi thao tác ghi vì `ThreadingHTTPServer` phục vụ mỗi request bằng một thread riêng — không có nó thì hai POST đồng thời có thể nhận cùng một `id`. Và `_send` cố tình **không** gắn `Content-Length` cho `204`: RFC 7230 cấm điều đó, mà Kong thì đọc header của upstream rất chặt.

Unit systemd — `%H` là hostname, nhờ đó `X-Upstream-Node` tự có giá trị đúng trên mỗi node:

```bash
tee /etc/systemd/system/demo-api.service << 'EOF'
[Unit]
Description=demo-api upstream cho lab Kong hybrid
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=demoapi
Group=demoapi
Environment=PORT=9001
Environment=NODE_NAME=%H
ExecStart=/usr/bin/python3 /opt/demo-api/app.py
Restart=always
RestartSec=3
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now demo-api
systemctl status demo-api --no-pager
```

**Kết quả mong đợi:**

```bash
curl -s http://10.10.200.11:9001/healthz | jq -c
curl -s http://10.10.200.12:9001/healthz | jq -c
```

```
{"status":"ok","node":"Kong-CP01","uptime_s":10.3}
{"status":"ok","node":"Kong-CP02","uptime_s":19.8}
```

Chạy thử trọn vòng CRUD, nối thẳng vào một instance chứ chưa qua Kong:

```bash
API=http://10.10.200.11:9001/api/v1/products

# CREATE
curl -s -X POST $API -H 'Content-Type: application/json' \
  -d '{"sku":"SRV-DL380","name":"HPE ProLiant DL380","price":3900}' | jq -c .data

# READ
curl -s $API/4 | jq -c .data

# UPDATE mot phan
curl -s -X PATCH $API/4 -H 'Content-Type: application/json' \
  -d '{"price":3500}' | jq -c .data

# DELETE roi doc lai
curl -s -o /dev/null -w 'delete: HTTP %{http_code}\n' -X DELETE $API/4
curl -s -o /dev/null -w 'doc lai: HTTP %{http_code}\n' $API/4
```

**Kết quả mong đợi:**

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/14.png)

Kho hàng nằm trong RAM của tiến trình, nên **hai instance có dữ liệu riêng và không đồng bộ**. Tạo một sản phẩm ở `.11` rồi đọc ở `.12` sẽ nhận `404`, và `systemctl restart demo-api` là mất sạch.

Đây là chủ ý: lab cần một upstream để soi hành vi của gateway, không cần một database thứ hai. Nó cũng là minh hoạ cho một luật chung — **backend giữ state cục bộ thì không đặt sau load balancer được**. Sau Step 13, gọi CRUD qua VIP `.50` sẽ thấy kết quả nhảy loạn tuỳ request rơi vào node nào.

### Step 13: Khai báo Service và Route

**Mục tiêu:** một upstream hai target có health check, một service, một route — tất cả tạo qua Admin API của CP.

Chạy từ bất cứ đâu tới được VIP `.51`:

```bash
CP=http://10.10.200.51:8001

# Upstream + active/passive health check
curl -sX POST $CP/upstreams \
  --data name=demo-api.upstream \
  --data slots=1000 \
  --data 'healthchecks.active.type=http' \
  --data 'healthchecks.active.http_path=/healthz' \
  --data 'healthchecks.active.timeout=2' \
  --data 'healthchecks.active.concurrency=2' \
  --data 'healthchecks.active.healthy.interval=5' \
  --data 'healthchecks.active.healthy.successes=2' \
  --data 'healthchecks.active.unhealthy.interval=5' \
  --data 'healthchecks.active.unhealthy.http_failures=2' \
  --data 'healthchecks.active.unhealthy.timeouts=2' \
  --data 'healthchecks.passive.unhealthy.http_failures=3' | jq -r .name

# Hai target
curl -sX POST $CP/upstreams/demo-api.upstream/targets \
  --data target=10.10.200.11:9001 --data weight=100 | jq -r .target
curl -sX POST $CP/upstreams/demo-api.upstream/targets \
  --data target=10.10.200.12:9001 --data weight=100 | jq -r .target

# Service trỏ vào upstream, kèm retry và timeout
curl -sX POST $CP/services \
  --data name=demo-api \
  --data host=demo-api.upstream \
  --data protocol=http \
  --data path=/ \
  --data retries=2 \
  --data connect_timeout=3000 \
  --data write_timeout=10000 \
  --data read_timeout=10000 | jq -r .name

# Route
curl -sX POST $CP/services/demo-api/routes \
  --data name=demo-api-v1 \
  --data 'paths[]=/demo' \
  --data 'methods[]=GET' \
  --data 'methods[]=POST' \
  --data strip_path=true | jq -r .name
```

**Kết quả mong đợi:** gọi qua VIP `.50` và thấy header `X-Upstream-Node` đổi qua lại giữa hai backend.

```bash
for i in $(seq 1 6); do
  curl -s -o /dev/null -D - http://10.10.200.50/demo/whoami | grep -i x-upstream-node
done
```

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/15.png)

Cấu hình vừa tạo trên CP, nhưng traffic thì DP xử lý — vòng round-robin ở trên chứng minh cấu hình đã đi hết chặng: Admin API `.51:8001` → HAProxy → Kong CP `18001` → PostgreSQL → kênh mTLS `8005` → DP → upstream.

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/16.png)

### Step 14: Gắn plugin key-auth và rate limiting

**Mục tiêu:** thấy plugin đồng bộ xuống DP, và hiểu giới hạn về rate limiting trong hybrid mode.

```bash
CP=http://10.10.200.51:8001

# Bắt buộc có API key mới vào được service demo-api
curl -sX POST $CP/services/demo-api/plugins \
  --data name=key-auth \
  --data 'config.key_names[]=apikey' \
  --data config.key_in_header=true | jq -r .name

# Consumer + key
curl -sX POST $CP/consumers --data username=lab-client | jq -r .username
curl -sX POST $CP/consumers/lab-client/key-auth \
  --data key=lab-secret-key-001 | jq -r .key

# Rate limiting — policy PHẢI là local hoặc redis trong hybrid mode
curl -sX POST $CP/services/demo-api/plugins \
  --data name=rate-limiting \
  --data config.minute=20 \
  --data config.policy=local \
  --data config.fault_tolerant=true | jq -r '.name + " policy=" + .config.policy'

# Gắn correlation-id và prometheus ở phạm vi global
curl -sX POST $CP/plugins --data name=correlation-id \
  --data config.header_name=X-Request-Id \
  --data config.generator=uuid \
  --data config.echo_downstream=true | jq -r .name
curl -sX POST $CP/plugins --data name=prometheus \
  --data config.status_code_metrics=true \
  --data config.latency_metrics=true \
  --data config.upstream_health_metrics=true | jq -r .name
```

Ba cờ của `prometheus` bắt buộc phải khai: từ Kong 3.0 plugin này mặc định **tắt** gần hết metric để giảm cardinality. Tạo plugin trống thì `/metrics` chỉ có metric mức node, không status code, không latency, không health của upstream.

`config.policy=cluster` là mặc định của `rate-limiting` nhưng **không dùng được ở đây**: nó ghi counter vào database, còn DP thì `database = off`. Hybrid mode chỉ còn `local` (counter riêng từng DP — 2 DP thành 40/phút chứ không phải 20) hoặc `redis` (dùng chung, chính xác). Muốn đúng con số thì phải có Redis; `rate-limiting-advanced` là plugin Enterprise.

**Kết quả mong đợi:**

```bash
# Không key → 401
curl -s -o /dev/null -w 'no key: HTTP %{http_code}\n' http://10.10.200.50/demo/api/v1/products

# Có key → 200
curl -s -H 'apikey: lab-secret-key-001' \
  -w '\nHTTP %{http_code}\n' http://10.10.200.50/demo/api/v1/products | tail -3

# Header rate limit và correlation id
curl -sD - -o /dev/null -H 'apikey: lab-secret-key-001' \
  http://10.10.200.50/demo/whoami | grep -iE 'ratelimit|x-request-id|x-kong'
```

POST thử một sản phẩm, lần này đi trọn đường VIP `.50` → DP → upstream:

```bash
curl -s -H 'apikey: lab-secret-key-001' -H 'Content-Type: application/json' \
  -d '{"sku":"SRV-DL380","name":"HPE ProLiant DL380","price":3900}' \
  http://10.10.200.50/demo/api/v1/products | jq -c
```

```
{"node":"Kong-CP01","data":{"id":5,"sku":"SRV-DL380","name":"HPE ProLiant DL380","price":3900}}
```

Trường `node` cho biết bản ghi vừa nằm lại ở instance nào. Gọi `GET /demo/api/v1/products` ngay sau đó có thể **không** thấy nó — request rơi vào instance còn lại, nơi chưa hề biết sản phẩm này. Đúng như đã nói ở Step 12: state cục bộ sau load balancer thì không nhất quán được.

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/17.png)

## Kiểm tra kết quả

Sáu bài test, xếp từ nhẹ đến nặng.

### 1. Cấu hình đã sync đều hai DP

```bash
curl -s http://10.10.200.51:8001/clustering/data-planes \
  | jq -r '.data[] | "\(.hostname)\t\(.sync_status)\t\(.config_hash[0:16])\t\(.last_seen)"'
```

```
Kong-DP01       normal  e3887ffdca587b7e        1785300029
Kong-DP02       normal  e3887ffdca587b7e        1785300030
```

Hai dòng phải cùng `config_hash`, `sync_status` đều `normal`, và `last_seen` chênh so với `date +%s` dưới khoảng 30 giây — lệch hàng phút nghĩa là DP đã rớt kênh mTLS từ lâu, bạn đang đọc dữ liệu tồn kho chứ không phải trạng thái sống.

Đối chiếu tiếp với chính DP:

```bash
for ip in 10.10.200.21 10.10.200.22; do
  echo -n "$ip: "; curl -s http://$ip:8100/status | jq -r .configuration_hash
done
```

```
10.10.200.21: e3887ffdca587b7e8b00071582eb4bf5
10.10.200.22: e3887ffdca587b7e8b00071582eb4bf5
```

Bước đối chiếu này mới quan trọng. `/clustering/data-planes` chỉ là CP đọc lại bản ghi trong PostgreSQL — DP chết thì bản ghi cũ vẫn nằm đó với `config_hash` đẹp đẽ. Còn `:8100/status` hỏi thẳng tiến trình Kong trên DP, trả về hash của cấu hình **đang nằm trong bộ nhớ nó**. Hai nguồn độc lập cho cùng một giá trị mới chứng minh được sync thật.

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/18.png)

### 2. Data Plane thật sự read-only

Gọi Admin API vào cả hai DP:

```bash
for ip in 10.10.200.21 10.10.200.22; do
  echo -n "$ip:8001 -> "
  curl -sS -m 3 -o /dev/null http://$ip:8001/services 2>&1 | head -1
done
```

```
10.10.200.21:8001 -> curl: (7) Failed to connect to 10.10.200.21 port 8001 after 0 ms: Couldn't connect to server
10.10.200.22:8001 -> curl: (7) Failed to connect to 10.10.200.22 port 8001 after 0 ms: Couldn't connect to server
```

Hai chi tiết cần đọc là **mã lỗi `(7)`** và **`after 0 ms`**: kernel trả RST ngay lập tức vì không tiến trình nào giữ cổng đó.

Kiểm tra từ xa chỉ chứng minh được "không với tới". Đứng trên chính DP mới chứng minh được "không tồn tại". Chạy trên **Kong-DP01 và Kong-DP02**:

```bash
grep -E '^role|^database|^admin_listen' /etc/kong/kong.conf
ss -lntp | grep -E ':8001|:8002|:18001' || echo 'khong co socket quan tri nao'
```

```
role = data_plane
database = off
admin_listen = off
khong co socket quan tri nao
```

Cấu hình khai `off` và thực tế không có socket nào mở — hai vế khớp nhau. `database = off` đóng nốt hướng còn lại: DP không có API để ghi, cũng không giữ kết nối nào tới PostgreSQL. Cộng với HAProxy trên DP chỉ bind `80`, `443` và `7000` (Step 2.2), node hứng traffic không còn đường nào chạm tới mặt phẳng ghi.

Đây là lý do bảo mật lớn nhất của hybrid mode. Admin API của Kong OSS **không có authentication** — ai gọi được `8001` là toàn quyền đổi route, gỡ `key-auth`, trỏ upstream sang chỗ khác. Mô hình truyền thống đặt Admin API ngay trên node hứng traffic nên chỉ còn trông cậy vào firewall; hybrid mode bỏ hẳn mặt phẳng ghi khỏi node tiếp xúc.

### 3. Failover PostgreSQL

Mở trước một terminal bắn liên tục qua VIP `.50`, giữ nó chạy suốt bài test:

```bash
while true; do
  curl -s -o /dev/null -w '%{http_code} ' -H 'apikey: lab-secret-key-001' \
    http://10.10.200.50/demo/whoami
  sleep 0.5
done
```

Ở terminal khác, ép đổi leader:

```bash
patronictl -c /etc/patroni/patroni.yml switchover --candidate Kong-CP02 --force
patronictl -c /etc/patroni/patroni.yml list
```

```
Successfully switched over to "Kong-CP02"
+ Cluster: kong-pg (7667778017656715364) ---------+----+-------------+-----+------------+-----+------------------+
| Member    | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag | Tags             |
+-----------+--------------+---------+-----------+----+-------------+-----+------------+-----+------------------+
| Kong-CP01 | 10.10.200.11 | Replica | stopped   |    |     unknown |     |    unknown |     |                  |
| Kong-CP02 | 10.10.200.12 | Leader  | running   |  2 |             |     |            |     |                  |
| Kong-DP01 | 10.10.200.21 | Replica | running   |  2 |             |     |            |     | nofailover: true |
+-----------+--------------+---------+-----------+----+-------------+-----+------------------+-----+----------------+
```

`stopped` và `unknown` ở leader cũ là trạng thái quá độ, không phải lỗi: Patroni dừng CP01 rồi khởi động lại nó ở vai replica, khoảnh khắc đó nó chưa kịp báo LSN. Đợi vài giây rồi `list` lại:

```
+ Cluster: kong-pg (7667778017656715364) ---------+----+-------------+-----+------------+-----+------------------+
| Member    | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag | Tags             |
+-----------+--------------+---------+-----------+----+-------------+-----+------------+-----+------------------+
| Kong-CP01 | 10.10.200.11 | Replica | streaming |  3 |   0/477FBE8 |   0 |  0/477FBE8 |   0 |                  |
| Kong-CP02 | 10.10.200.12 | Leader  | running   |  3 |             |     |            |     |                  |
| Kong-DP01 | 10.10.200.21 | Replica | streaming |  3 |   0/477FBE8 |   0 |  0/477FBE8 |   0 | nofailover: true |
+-----------+--------------+---------+-----------+----+-------------+-----+------------+-----+------------------+
```

`TL` nhảy từ `2` lên `3` — mỗi lần promote là timeline tăng một. Con số bao nhiêu không quan trọng bằng việc **cả ba node cùng một TL**: node nào tụt lại phải `pg_rewind` mới join lại được, và đó mới là dấu hiệu switchover có vấn đề.

Quay lại terminal đang bắn curl — **toàn bộ phải là `200`**, không một mã lỗi nào:

```
200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200
```

Traffic đi qua DP, mà DP không giữ kết nối nào tới database, nên đổi leader hoàn toàn vô hình với client. Bấm `^C` để dừng vòng lặp.

Còn phía CP thì HAProxy phải tự bám theo leader mới, và mặt phẳng ghi vẫn phải hoạt động:

```bash
curl -s "http://10.10.200.11:7000/;csv" | awk -F, '$1 == "pg_primary" && $2 ~ /Kong-/ {print $2, $18}'
curl -sX POST http://10.10.200.51:8001/services --data name=probe --data url=http://127.0.0.1:9001 | jq -r .name
curl -sX DELETE http://10.10.200.51:8001/services/probe -o /dev/null -w '%{http_code}\n'
```

```
Kong-CP01 DOWN
Kong-CP02 UP
Kong-DP01 DOWN
probe
204
```

`pg_primary` giờ chỉ còn Kong-CP02 `UP` — HAProxy tự phát hiện qua `option httpchk GET /primary`, không ai phải sửa cấu hình. `DOWN` nghĩa là node trả `503` cho `/primary`, tức không phải leader, chứ không phải node chết.

Hai lệnh cuối mới là phần quan trọng nhất: tạo rồi xóa được service `probe` (`204`) chứng minh Admin API vẫn ghi xuống database sau khi leader đổi chỗ. Chuỗi `Kong → HAProxy 127.0.0.1:5000 → Patroni leader` đứt ở đâu đó thì `POST` sẽ treo hoặc trả `500`.

### 4. Failover Control Plane

Với mô hình keepalived-endpoint + HAProxy-load-balancer, mất một CP có hai kịch bản khác hẳn nhau. Chạy cả hai mới thấy hết.

**4a — Kong chết, HAProxy còn sống.** Trên Kong-CP01:

```bash
systemctl stop kong
```

VIP `.51` **không** nhúc nhích: `chk_haproxy` vẫn thấy HAProxy chạy nên keepalived giữ nguyên. HAProxy phát hiện Kong-CP01 hỏng qua health check `8100` (~9 giây với `inter 3s fall 3`) rồi dồn hết sang Kong-CP02.

```bash
ip -4 addr show ens160 | grep 10.10.200.51          # VIP vẫn ở CP01
curl -s "http://10.10.200.11:7000/;csv" | awk -F, '$1 ~ /^kong_/ && $2 ~ /Kong-CP0/ {print $1, $2, $18}'
for i in $(seq 4); do curl -s http://10.10.200.51:8001/ | jq -r .hostname; done
```

Cả bốn request về Kong-CP02, và DP nào đang nối vào CP01 sẽ reconnect sang CP02. Bật lại `systemctl start kong` rồi kiểm tra lại — HAProxy đưa CP01 về `UP` sau 2 lần check thành công (`rise 2`).

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/19.png)

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/20.png)

![](/assets/img/2026-07-29-kong-oss-hybrid-mode-patroni-postgresql-ha/21.png)

**4b — Cả node chết.** Lần này VIP mới thật sự nhảy sang node còn lại, và kênh cluster của DP bị cắt.

Xác định node nào đang giữ VIP — phải stop đúng node đó, vì mọi kênh cluster đều đi qua HAProxy của nó:

```bash
ip -4 -br a show ens160
```

Trên node đang giữ `.51`:

```bash
systemctl stop keepalived haproxy kong
```

Đợi khoảng 10 giây cho DP retry, rồi kiểm tra từ node **còn lại** — giả sử VIP vừa nhảy sang Kong-CP02:

```bash
ip -4 -br a show ens160
systemctl is-active keepalived haproxy kong
curl -s "http://10.10.200.12:7000/;csv" | awk -F, '$1 == "kong_cluster" {print $2, $5, $18}'
curl -s http://10.10.200.12:18001/clustering/data-planes \
  | jq -r '.data[] | "\(.hostname) \(.sync_status) \(.last_seen)"'
date +%s
```

**Kết quả mong đợi:**

```
ens160  UP  10.10.200.12/24 10.10.200.51/24 fe80::250:56ff:fe93:aa5f/64
active
active
active
FRONTEND 2 OPEN
Kong-CP01 0 DOWN
Kong-CP02 2 UP
BACKEND 2 UP
Kong-DP01 normal 1785307454
Kong-DP02 normal 1785307451
1785307460
```

Đọc theo thứ tự: VIP `.51` đã sang CP02, ba dịch vụ đều sống, `FRONTEND 2` nghĩa là **cả hai DP đã nối lại** vào CP02, và `last_seen` cách `date +%s` chưa tới 10 giây. `Kong-CP01 0 DOWN` là đúng — Kong trên CP01 đã stop nên health check `8100` fail. Vòng lặp curl ở test 3 suốt quãng này vẫn toàn `200`.

Khởi động lại node vừa stop, VIP sẽ về lại vì priority cao hơn. Nhớ thứ tự: `haproxy` trước, `kong` sau — drop-in `After=haproxy.service` ở Step 10 lo việc này khi boot, nhưng khi start tay thì tự giữ đúng thứ tự.

Khác biệt giữa 4a và 4b chính là lý do keepalived track HAProxy chứ không track Kong: hỏng một tiến trình Kong là việc HAProxy tự xử lý tại chỗ, không cần động đến VIP — mà mỗi lần VIP nhảy là một lần **mọi** kết nối đang mở qua nó bị cắt.

### 5. Bài test then chốt — down database

Đây là điều duy nhất mà kiến trúc DB-less trên DP hứa hẹn, và cũng là lý do đáng dựng lab này.

```bash
# Trên cả Kong-CP01, Kong-CP02, Kong-DP01
systemctl stop patroni
```

Rồi bắn traffic:

```bash
curl -s -H 'apikey: lab-secret-key-001' http://10.10.200.50/demo/api/v1/products | jq -c '{node, count}'
curl -s -o /dev/null -w 'proxy: HTTP %{http_code}\n' -H 'apikey: lab-secret-key-001' http://10.10.200.50/demo/whoami
curl -s -o /dev/null -w 'admin: HTTP %{http_code}\n' http://10.10.200.51:8001/services
```

**Kết quả mong đợi:**

```
{"node":"Kong-CP02","count":3}
proxy: HTTP 200
admin: HTTP 500
```

`node` ra CP01 hay CP02 đều được — đó chỉ là target nào của upstream nhận request lần này.

Hai con số phải xuất hiện **cùng lúc** thì bài test mới có giá trị: `admin: HTTP 500` chứng minh CP thật sự mất database, nhờ đó `proxy: HTTP 200` mới không thể giải thích bằng việc database còn sống lén lút ở đâu đó.

Traffic vẫn chạy, kể cả `key-auth` và `rate-limiting` — DP đọc consumer và credential từ `lmdb` trên disk, không truy vấn database. Chỉ mặt phẳng quản trị chết. Restart DP trong lúc database vẫn down cũng vẫn lên được, vì nó load lại từ `lmdb`:

```bash
systemctl restart kong    # trên Kong-DP02
curl -s -o /dev/null -w 'sau restart: HTTP %{http_code}\n' -H 'apikey: lab-secret-key-001' http://10.10.200.50/demo/whoami
```

Bật lại Patroni trên cả 3 node khi xong. Lưu ý thứ tự: node từng là leader nên start trước.

### 6. Health check và failover Data Plane

Ép `demo-api` trên Kong-CP02 báo bệnh, không cần stop service:

```bash
curl -sX POST http://10.10.200.12:9001/admin/unhealthy | jq -c
sleep 12
for i in $(seq 1 6); do
  curl -s -H 'apikey: lab-secret-key-001' http://10.10.200.50/demo/whoami | jq -r .node
done
```

**Kết quả mong đợi:** cả 6 request về `Kong-CP01` — mỗi DP tự chạy active health check của mình và đã loại target Kong-CP02 khỏi ring balancer.

```bash
curl -sX POST http://10.10.200.12:9001/admin/healthy | jq -c   # trả lại bình thường
```

Trong hybrid mode, trạng thái health là **cục bộ trên từng DP**, CP không tổng hợp giúp bạn — Admin API của CP không có balancer nên `/upstreams/demo-api.upstream/health` không phản ánh thực tế đang chạy ở DP. Cách xác minh đúng là quan sát traffic như trên, hoặc đọc metric Prometheus ngay tại DP.

Trước khi đọc metric, phải **bắn ít nhất vài request qua chính DP đó**. Plugin `prometheus` chỉ đăng ký các họ metric khi có request đi qua sau lúc cấu hình có hiệu lực; scrape một DP vừa restart mà chưa nhận traffic thì `/metrics` chỉ trả về mấy chỉ số mức node và bạn sẽ tưởng metric không tồn tại:

```bash
for i in $(seq 1 5); do
  curl -s -o /dev/null -H 'apikey: lab-secret-key-001' http://10.10.200.21:8000/demo/whoami
done
curl -s http://10.10.200.21:8100/metrics | grep -E 'kong_upstream_target_health'
```

Cuối cùng, rớt một DP:

```bash
# Trên Kong-DP01
systemctl stop kong
```

HAProxy mất health check port `8100` của Kong-DP01 trong ~9 giây (`inter 3s fall 3`) và dồn hết về Kong-DP02. Client vẫn `200`. Trang stats `http://10.10.200.21:7000/` cho thấy chính xác thời điểm đó.

## Đường lên Enterprise

Lab này là bản nháp của kiến trúc Enterprise, nhưng đường nâng cấp có ba điều cần biết trước.

**OSS đã đóng băng ở 3.9.1.** Từ 3.10 Kong ngừng publish Docker image prebuilt cho OSS và không còn release community mới. Enterprise thì vẫn chạy đều 4 minor/năm, hiện ở 3.15.

**Không cài Enterprise trước rồi chờ license.** `free mode` đã bị bỏ từ 3.10 — chạy Enterprise không license giờ hành xử y như license hết hạn: toàn bộ entity thành **read-only**, node DB-less mới không start được. Traffic đang chạy thì vẫn proxy bằng cấu hình cũ, và DP trong hybrid mode vẫn nhận được config từ CP có license hết hạn, nhưng CP read-only nghĩa là bạn không tạo nổi một service nào.

**Đừng migrate in-place.** Kong chỉ cho migrate sang bản Enterprise dựa trên **đúng OSS version đang chạy**, tức 3.9.1 → Enterprise 3.9.x.x, bằng `kong migrations up` + `kong migrations -f finish`, và thao tác này **không thể đảo lại**. Nhưng Enterprise 3.9 đã EOL từ 01/2026, nên bạn còn phải leo tiếp 3.9 → 3.10 LTS → ... → 3.15.

Đường sạch hơn nhiều: DP là stateless, database chỉ chứa cấu hình. Dùng **decK** để mang cấu hình sang một cụm Enterprise mới:

```bash
# Trên cụm OSS hiện tại
deck gateway dump --kong-addr http://10.10.200.51:8001 -o kong-oss-state.yaml

# Trên cụm Enterprise sau khi đã có license
deck gateway diff --kong-addr http://<cp-ee-vip>:8001 -s kong-oss-state.yaml
deck gateway sync --kong-addr http://<cp-ee-vip>:8001 -s kong-oss-state.yaml
```

Khi license có, cách nạp **rất quan trọng**:

| Cách nạp | Có tự đẩy xuống DP? |
|----------|---------------------|
| `POST /licenses` qua Admin API của CP | **Có** — CP tự phân phối cho toàn bộ DP trong cụm |
| File `/etc/kong/license.json` | Không — phải copy tay lên từng node |
| Biến `KONG_LICENSE_DATA` / `KONG_LICENSE_PATH` | Không — phải set trên từng node |

Với hybrid mode thì dùng `/licenses` là lựa chọn duy nhất hợp lý.

Những gì lab OSS này **không** kiểm chứng được, phải chờ license:

- **RBAC và workspace** — Kong Manager OSS ở `:8002` không có phân quyền, không có multi-tenant. Đây thường là lý do chính người ta mua Enterprise.
- **Vitals** — biểu đồ latency/traffic theo entity. Lab này thay bằng plugin `prometheus`.
- **Dev Portal** — cổng tài liệu API cho consumer.
- **Plugin Enterprise** — `openid-connect`, `mtls-auth`, `rate-limiting-advanced` (cái này mới cho rate limit chính xác trong hybrid mà không cần dựng Redis riêng).
- **Secrets management** với vault backend.


## Kết luận

4 VM, 3 tầng HA, và một kiến trúc trong đó mặt phẳng ghi cấu hình tách hoàn toàn khỏi mặt phẳng xử lý traffic. Điều đáng nhớ nhất từ lab là bài test số 5: tắt sạch cụm PostgreSQL mà client không thấy một mã lỗi nào — vì Data Plane chưa từng biết database tồn tại.
