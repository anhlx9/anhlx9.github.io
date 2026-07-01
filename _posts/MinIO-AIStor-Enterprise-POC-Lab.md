# LAB POC – MinIO AIStor Enterprise (full subscription) trên VMware ESXi

> **Mục tiêu:** Dựng cluster **MinIO AIStor Enterprise (license trial 60 ngày)** đầy đủ tính năng cho khách hàng tài chính:
> - Storage node **HA, EC:4**
> - **Xác thực qua Active Directory** (Windows Server 2025), **LDAPS**
> - **Mã hóa at-rest CHỌN LỌC theo bucket** (KES + Vault, khóa riêng từng bucket)
> - **Vault auto-unseal** (transit seal)
> - **Toàn bộ kết nối HTTPS**, cert do **AD CS (Win2025) cấp** — dùng luôn cho production
> - **Giám sát Prometheus + Grafana**
>
> **Production về sau:** tăng node (8+ vật lý), KES/Vault HA tách host, auto-unseal bằng HSM/cloud KMS, dùng chung **AD CS** đã có.

---

## 1. Phạm vi & nguyên tắc

- Lab mô phỏng cơ chế enterprise, **không phải HA production thật**. Mọi VM trên 1 host ESXi → host là SPOF.
- AIStor **Free chỉ single-node** → cluster 4 node **bắt buộc license Enterprise/trial 60 ngày** (SUBNET). Trial **không có grace period**.
- **Critical-path:** sau khi cấu hình KES, AIStor cần **KES sống + Vault unsealed** mới khởi động/đọc-ghi. Auto-unseal giúp Vault tự mở khi restart (xem mục 8.3).

---

## 2. Kiến trúc tổng thể

```
                         ┌────────────────────────────┐
                         │  Win Server 2025 – DC01     │  AD DS + DNS + NTP + AD CS (Enterprise CA)
                         │  10.10.200.5  (lab.local)   │  ← cấp toàn bộ cert (lab + production)
                         └─────────────┬──────────────┘
                   LDAPS 636 │ DNS 53 │ NTP 123 │ (cấp cert)
        ┌───────────────────────────────┼───────────────────────────────┐
        ▼                               ▼                               ▼
┌──────────────┐   HTTPS 443  ┌──────────────────────────────┐   ┌──────────────┐
│  Client/mc   ├────────────▶ │       svc01 (Ubuntu 24)       │   │  4× MinIO    │
└──────────────┘              │        Docker Compose         │   │  AIStor node │
                              │ HAProxy:443  Grafana:3000(TLS)│   │ minio1..4    │
  S3 endpoint                 │ Prometheus:9090(TLS)          │   │ .11–.14      │
  s3.lab.local                │ KES:7373 ── Vault(main):8200  │   │ 3×300GB data │
  → 10.10.200.10              │              ▲ auto-unseal    │   │ EC:4 (8+4)   │
                              │              └ Vault-transit  │   └──────────────┘
                              │                :8210 (anchor) │   erasure set 12 ổ
                              │  10.10.200.10                 │
                              └──────────────────────────────┘
   Subnet 10.10.200.0/24 | GW 10.10.200.1 | egress NAT 443/80/53/123 | CẤM ping
   *** MỌI kết nối dùng HTTPS/LDAPS, verify bằng AD Root CA, gọi qua FQDN ***
```

---

## 3. Bảng VM & tài nguyên (ESXi)

| VM | OS | vCPU | RAM | Disk | Vai trò |
|----|----|------|-----|------|---------|
| DC01 | Windows Server 2025 | 4 | 16 GB | 100 GB | AD DS, DNS, NTP, **AD CS (Enterprise Root CA)** |
| svc01 | Ubuntu 24.04 | 8 | 16 GB | 120 GB | Docker: HAProxy, KES, Vault(main), Vault-transit, Prometheus, Grafana |
| minio1–4 | Ubuntu 24.04 | 4 | 8 GB | 1×200 (OS)+3×300 (data) | AIStor node |

**Tổng:** 28 vCPU, 64 GB RAM, ~4.7 TB (thin). Data VMDK: PVSCSI, thick eager-zeroed, independent-persistent.
**EC:** 4 node × 3 ổ = 12 drive → EC:4 (8+4), ~2.4 TB usable, chịu mất 1 node.

---

## 4. IP / DNS / Domain

| Host | FQDN | IP |
|------|------|-----|
| DC01 | `DC01.lab.local` | 10.10.200.5 |
| svc01 / S3 endpoint | `svc01.lab.local` / `s3.lab.local` | 10.10.200.10 |
| minio1–4 | `minio1..4.lab.local` | 10.10.200.11–14 |

- **Domain AD:** `lab.local`. **DNS** cho mọi VM Linux = **10.10.200.5 (DC01)**.
- Tạo bản ghi A trên DNS AD: `s3`, `svc01`, `minio1..4`.
- **Bắt buộc gọi nhau bằng FQDN** (không IP) để cert verify đúng hostname.

---

## 5. Thứ tự khởi động

1. **DC01** (AD/DNS/NTP/AD CS).
2. **Vault-transit** → init + unseal (1 lần, đây là trust anchor).
3. **Vault main** → init (**tự unseal** nhờ transit) → KV v1 + AppRole + policy.
4. **KES** → tạo các key.
5. **4× MinIO** → bật SSE-KMS cho bucket cần mã hóa.
6. **HAProxy / Prometheus / Grafana**.
> Vault main restart → **tự unseal** (không cần nhập tay). Chỉ Vault-transit cần unseal thủ công khi nó restart (hiếm).

---

## 6. Chuẩn bị chung VM Linux (4 node + svc01)

### 6.1 Netplan (ví dụ minio1)
```yaml
network:
  version: 2
  ethernets:
    ens160:
      addresses: [10.10.200.11/24]
      routes: [{ to: default, via: 10.10.200.1 }]
      nameservers: { addresses: [10.10.200.5], search: [lab.local] }
```

### 6.2 Time sync (BẮT BUỘC)
```bash
apt-get install -y chrony
# /etc/chrony/chrony.conf:  server 10.10.200.5 iburst   (DC01)
systemctl restart chrony && chronyc tracking
```

### 6.3 Firewall + cấm ping
```bash
ufw default deny incoming; ufw default deny outgoing
ufw allow out 53; ufw allow out 123/udp; ufw allow out 80/tcp; ufw allow out 443/tcp
ufw allow out to 10.10.200.0/24
ufw allow from 10.10.200.0/24 to any port 22 proto tcp
ufw allow from 10.10.200.0/24 to any port 9000 proto tcp
ufw allow from 10.10.200.0/24 to any port 9001 proto tcp
ufw enable
echo 'net.ipv4.icmp_echo_ignore_all=1' >/etc/sysctl.d/99-noping.conf && sysctl --system
```

### 6.4 Tin AD Root CA (mọi VM Linux)
```bash
# Copy file root CA xuất từ DC01 (mục 7) rồi:
cp lab-root-ca.crt /usr/local/share/ca-certificates/lab-root-ca.crt
update-ca-certificates
```

---

## 7. DC01 – Windows Server 2025 (AD + DNS + NTP + AD CS)

1. IP tĩnh `10.10.200.5`, hostname `DC01`. Cài **AD DS** → promote domain `lab.local` (DNS kèm theo).
2. **NTP:** cấu hình DC làm nguồn giờ domain; DC sync ra ngoài qua 123.
3. Cấu trúc AD: OU `Users`/`Groups`/`Service`; group `minio-admins`, `minio-readwrite`, `minio-readonly`; user `alice`(admins), `bob`(readonly); service account `svc-minio-ldap` (bind LDAP).
4. **Cài AD CS** (Active Directory Certificate Services):
   - Add Roles → **AD CS** → Certification Authority → **Enterprise → Root CA**.
   - Cho phép nhận **SAN từ CSR** (để cấp cert cho dịch vụ Linux):
     ```cmd
     certutil -setreg policy\EditFlags +EDITF_ATTRIBUTESUBJECTALTNAME2
     net stop certsvc && net start certsvc
     ```
   - **LDAPS 636** bật tự động khi DC có cert (DC tự enroll Domain Controller cert từ Enterprise CA).
   - **Xuất Root CA cert** (Base64) để phân phối cho Linux:
     ```cmd
     certutil -ca.cert C:\lab-root-ca.cer
     ```
     Chuyển sang PEM nếu cần: `openssl x509 -inform DER -in lab-root-ca.cer -out lab-root-ca.crt`

> **Production:** dùng lại chính Enterprise CA này (hoặc một **Subordinate CA** ký từ nó) để cấp cert cho cụm thật — không phát sinh CA mới.

---

## 8. Cấp cert từ AD CS cho các dịch vụ Linux

Quy trình cho **mỗi** dịch vụ (svc01 và từng minio node): tạo key+CSR có SAN trên Linux → nộp CSR cho AD CA → lấy cert → cài.

### 8.1 Tạo CSR (ví dụ cho svc01, gộp SAN cho mọi dịch vụ trên svc01)
```bash
mkdir -p ~/lab/certs && cd ~/lab/certs
openssl req -new -newkey rsa:2048 -nodes \
  -keyout svc01.key -out svc01.csr \
  -subj "/CN=svc01.lab.local" \
  -addext "subjectAltName=DNS:svc01.lab.local,DNS:s3.lab.local"
```
MinIO node (chạy trên từng node, ví dụ minio1):
```bash
openssl req -new -newkey rsa:2048 -nodes \
  -keyout private.key -out minio1.csr \
  -subj "/CN=minio1.lab.local" \
  -addext "subjectAltName=DNS:minio1.lab.local"
```

### 8.2 Nộp CSR cho AD CA (chạy trên 1 máy Windows tới được CA)
```cmd
certreq -submit -attrib "CertificateTemplate:WebServer" svc01.csr svc01.cer
certreq -submit -attrib "CertificateTemplate:WebServer" minio1.csr minio1.cer
```
Đổi sang PEM nếu cert trả về dạng DER:
```bash
openssl x509 -inform DER -in svc01.cer -out svc01.crt
```
> Cert dùng cho: HAProxy (`s3.pem` = `cat svc01.crt svc01.key`), KES server, Vault (main + transit), Grafana, Prometheus → **đều dùng `svc01.crt/.key`** (cùng host, SAN bao gồm svc01 + s3).
> MinIO mỗi node: `public.crt`+`private.key` đặt tại `/home/minio-user/.minio/certs/`, và `lab-root-ca.crt` đặt tại `/home/minio-user/.minio/certs/CAs/`.

---

## 9. svc01 – Docker Compose

```bash
apt-get install -y docker.io docker-compose-plugin
mkdir -p ~/lab/{certs,vault/config,vault/data,vault-transit/config,vault-transit/data,kes,prometheus,grafana,haproxy}
cd ~/lab
cp /usr/local/share/ca-certificates/lab-root-ca.crt certs/
cat certs/svc01.crt certs/svc01.key > certs/s3.pem
```

### 9.1 `docker-compose.yml`
```yaml
services:
  vault-transit:                       # trust anchor cho auto-unseal
    image: hashicorp/vault:latest
    cap_add: ["IPC_LOCK"]
    ports: ["8210:8200"]
    volumes: ["./vault-transit/config:/vault/config","./vault-transit/data:/vault/data","./certs:/certs"]
    command: server

  vault:                               # Vault chính (KES keystore) – auto-unseal
    image: hashicorp/vault:latest
    cap_add: ["IPC_LOCK"]
    depends_on: [vault-transit]
    ports: ["8200:8200"]
    volumes: ["./vault/config:/vault/config","./vault/data:/vault/data","./certs:/certs"]
    command: server

  kes:
    image: quay.io/minio/kes:latest
    depends_on: [vault]
    ports: ["7373:7373"]
    volumes: ["./kes:/etc/kes","./certs:/certs"]
    command: server --config=/etc/kes/kes-config.yaml

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
      GF_SECURITY_ADMIN_PASSWORD: "ĐỔI-MẬT-KHẨU"

  haproxy:
    image: haproxy:lts
    ports: ["80:80","443:443"]
    volumes: ["./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro","./certs:/certs:ro"]
```

### 9.2 Vault-transit: init + unseal + transit key (anchor)
`~/lab/vault-transit/config/vault.hcl`:
```hcl
storage "file" { path = "/vault/data" }
listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/certs/svc01.crt"
  tls_key_file  = "/certs/svc01.key"
}
api_addr = "https://svc01.lab.local:8210"
ui = true
```
```bash
docker compose up -d vault-transit
export VAULT_ADDR=https://svc01.lab.local:8210 VAULT_CACERT=~/lab/certs/lab-root-ca.crt
vault operator init            # LƯU 5 unseal key + root token
vault operator unseal          # nhập 3/5 (anchor – chỉ làm khi transit restart)
vault login <root-token>
vault secrets enable transit
vault write -f transit/keys/autounseal
cat > /tmp/au.hcl <<'EOF'
path "transit/encrypt/autounseal" { capabilities = ["update"] }
path "transit/decrypt/autounseal" { capabilities = ["update"] }
EOF
vault policy write autounseal /tmp/au.hcl
vault token create -policy=autounseal -period=24h -orphan   # → TRANSIT_TOKEN
```

### 9.3 Vault main: AUTO-UNSEAL qua transit
`~/lab/vault/config/vault.hcl`:
```hcl
storage "file" { path = "/vault/data" }

seal "transit" {                       # ← auto-unseal
  address     = "https://vault-transit:8200"
  token       = "<TRANSIT_TOKEN>"
  key_name    = "autounseal"
  mount_path  = "transit/"
  tls_ca_cert = "/certs/lab-root-ca.crt"
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/certs/svc01.crt"
  tls_key_file  = "/certs/svc01.key"
}
api_addr = "https://svc01.lab.local:8200"
ui = true
```
```bash
docker compose up -d vault
export VAULT_ADDR=https://svc01.lab.local:8200
vault operator init            # → RECOVERY keys + root token (KHÔNG phải unseal key)
                               # Vault main đã TỰ unsealed; restart sẽ tự unseal lại
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
> **Auto-unseal nghĩa là:** khi container `vault` (main) khởi động lại, nó tự dùng transit để mở khóa — **không cần nhập unseal key tay**. Recovery keys chỉ dùng cho rekey/generate-root, cất an toàn.
> **Production:** thay `vault-transit` bằng **HSM (PKCS#11)** hoặc **cloud KMS**; Vault main chạy **Raft HA 3 node**.

### 9.4 KES config + tạo các key
```bash
docker run --rm quay.io/minio/kes:latest identity new   # → API key kes:v1:... + Identity hash
```
`~/lab/kes/kes-config.yaml`:
```yaml
address: 0.0.0.0:7373
admin: { identity: disabled }
tls: { key: /certs/svc01.key, cert: /certs/svc01.crt }
policy:
  minio-server:
    allow: ["/v1/key/create/*","/v1/key/generate/*","/v1/key/decrypt/*","/v1/key/bulk/decrypt","/v1/key/list/*","/v1/status","/v1/metrics"]
    identities: ["<IDENTITY_HASH_API_KEY>"]
keystore:
  vault:
    endpoint: https://svc01.lab.local:8200
    engine: kv
    version: v1
    approle: { id: "<VAULT_APPROLE_ID>", secret: "<VAULT_APPROLE_SECRET>" }
    tls: { ca: /certs/lab-root-ca.crt }
```
```bash
docker compose up -d kes
export KES_SERVER=https://svc01.lab.local:7373 KES_API_KEY="kes:v1:..."
# Khóa cho backend (bắt buộc) + khóa riêng từng bucket:
kes key create minio-backend-default-key
kes key create key-bucketA
kes key create key-bucketB     # ví dụ cho phòng ban/khách hàng khác
```

### 9.5 Prometheus (TLS)
`~/lab/prometheus/web-config.yml`:
```yaml
tls_server_config: { cert_file: /certs/svc01.crt, key_file: /certs/svc01.key }
```
`~/lab/prometheus/prometheus.yml`:
```yaml
global: { scrape_interval: 15s }
scrape_configs:
  - job_name: minio-cluster
    metrics_path: /minio/v2/metrics/cluster
    scheme: https
    tls_config: { ca_file: /certs/lab-root-ca.crt }
    static_configs: [{ targets: ['minio1.lab.local:9000','minio2.lab.local:9000','minio3.lab.local:9000','minio4.lab.local:9000'] }]
  - job_name: minio-node
    metrics_path: /minio/v2/metrics/node
    scheme: https
    tls_config: { ca_file: /certs/lab-root-ca.crt }
    static_configs: [{ targets: ['minio1.lab.local:9000','minio2.lab.local:9000','minio3.lab.local:9000','minio4.lab.local:9000'] }]
```

### 9.6 HAProxy (HTTPS đầu-cuối, verify backend bằng AD CA)
`~/lab/haproxy/haproxy.cfg`:
```haproxy
defaults
  mode http
  timeout connect 10s
  timeout client 5m
  timeout server 5m
  timeout tunnel 1h
frontend s3
  bind *:80
  bind *:443 ssl crt /certs/s3.pem
  http-request redirect scheme https unless { ssl_fc }
  default_backend minio
backend minio
  balance roundrobin
  option httpchk GET /minio/health/live
  http-check expect status 200
  server minio1 minio1.lab.local:9000 check ssl verify required ca-file /certs/lab-root-ca.crt
  server minio2 minio2.lab.local:9000 check ssl verify required ca-file /certs/lab-root-ca.crt
  server minio3 minio3.lab.local:9000 check ssl verify required ca-file /certs/lab-root-ca.crt
  server minio4 minio4.lab.local:9000 check ssl verify required ca-file /certs/lab-root-ca.crt
```
```bash
docker compose up -d prometheus grafana haproxy
```

---

## 10. 4× MinIO AIStor node

### 10.1 Cài gói + license + cert AD CS (mỗi node)
```bash
curl -L https://dl.min.io/aistor/minio/release/linux-amd64/minio.deb -o minio.deb && sudo dpkg -i minio.deb
sudo mkdir -p /opt/minio /home/minio-user/.minio/certs/CAs
sudo cp minio.license /opt/minio/minio.license
sudo cp public.crt private.key /home/minio-user/.minio/certs/         # cert AD CS của node
sudo cp lab-root-ca.crt /home/minio-user/.minio/certs/CAs/            # tin AD CA (cho KES + LDAPS)
sudo chown -R minio-user:minio-user /opt/minio /home/minio-user/.minio /mnt/data1 /mnt/data2 /mnt/data3
```

### 10.2 `/etc/default/minio` (GIỐNG NHAU cả 4 node — `shasum -a 256` để kiểm)
```bash
# --- License & topology (HTTPS) ---
MINIO_LICENSE="/opt/minio/minio.license"
MINIO_VOLUMES="https://minio{1...4}.lab.local:9000/mnt/data{1...3}/minio"
MINIO_ROOT_USER="admin"
MINIO_ROOT_PASSWORD="ĐỔI-MẬT-KHẨU-MẠNH"
MINIO_OPTS="--console-address :9001"
MINIO_STORAGE_CLASS_STANDARD="EC:4"
MINIO_SERVER_URL="https://s3.lab.local"
MINIO_PROMETHEUS_AUTH_TYPE="public"

# --- KMS qua KES + Vault (KHÔNG bật auto-encryption → để chọn lọc theo bucket) ---
MINIO_KMS_KES_ENDPOINT="https://svc01.lab.local:7373"
MINIO_KMS_KES_API_KEY="kes:v1:...."
MINIO_KMS_KES_CAPATH="/home/minio-user/.minio/certs/CAs/lab-root-ca.crt"
MINIO_KMS_KES_KEY_NAME="minio-backend-default-key"
# (CỐ Ý không đặt MINIO_KMS_AUTO_ENCRYPTION)

# --- Xác thực AD qua LDAPS (verify đầy đủ, KHÔNG skip) ---
MINIO_IDENTITY_LDAP_SERVER_ADDR="DC01.lab.local:636"
MINIO_IDENTITY_LDAP_LOOKUP_BIND_DN="CN=svc-minio-ldap,OU=Service,DC=lab,DC=local"
MINIO_IDENTITY_LDAP_LOOKUP_BIND_PASSWORD="<mật khẩu svc-minio-ldap>"
MINIO_IDENTITY_LDAP_USER_DN_SEARCH_BASE_DN="OU=Users,DC=lab,DC=local"
MINIO_IDENTITY_LDAP_USER_DN_SEARCH_FILTER="(&(objectClass=user)(sAMAccountName=%s))"
MINIO_IDENTITY_LDAP_GROUP_SEARCH_BASE_DN="OU=Groups,DC=lab,DC=local"
MINIO_IDENTITY_LDAP_GROUP_SEARCH_FILTER="(&(objectClass=group)(member=%d))"
```
> LDAPS verify được vì cert DC do **AD CS** cấp và Linux đã tin AD Root CA (mục 6.4). Không cần `TLS_SKIP_VERIFY`.

### 10.3 Khởi động
```bash
sudo systemctl enable --now minio
journalctl -u minio -f
```

---

## 11. Mã hóa CHỌN LỌC theo bucket (khóa riêng từng bucket)

```bash
mc alias set adm https://s3.lab.local admin "<root-pass>"
mc admin kms key status adm                 # KES/Vault OK

# Bucket A: bật SSE-KMS mặc định với KHÓA RIÊNG key-bucketA
mc mb adm/bucketA
mc encrypt set sse-kms key-bucketA adm/bucketA

# Bucket B: KHÔNG bật mã hóa → object plaintext trên đĩa
mc mb adm/bucketB

# (Tùy chọn) Bucket C của phòng ban khác – khóa riêng để cô lập
mc mb adm/bucketC
mc encrypt set sse-kms key-bucketB adm/bucketC

# --- Kiểm chứng ---
mc encrypt info adm/bucketA      # → SSE-KMS, key-bucketA
mc encrypt info adm/bucketB      # → Auto encryption is not enabled
mc cp /tmp/f adm/bucketA/ && mc stat adm/bucketA/f | grep -i encrypt   # CÓ SSE-KMS + key-bucketA
mc cp /tmp/f adm/bucketB/ && mc stat adm/bucketB/f | grep -i encrypt   # KHÔNG có dòng Encrypted
```

Ý nghĩa cho khách tài chính:
- **Mỗi bucket một khóa** → cô lập mã hóa theo phòng ban/khách hàng; thu hồi/xoay 1 khóa không ảnh hưởng bucket khác.
- **Bucket B plaintext** có chủ đích (vd dữ liệu public/không nhạy cảm) → hiệu năng cao hơn, không phụ thuộc KMS.
- **Lưu ý:** **backend hệ thống (IAM/config) LUÔN mã hóa** bằng `minio-backend-default-key` khi đã cấu hình KES — độc lập với việc bucket có mã hóa hay không.
- Xoay khóa 1 bucket: `mc admin kms key create adm key-bucketA-v2` rồi `mc encrypt set sse-kms key-bucketA-v2 adm/bucketA`.

---

## 12. Tích hợp AD: gán policy theo group

```bash
mc idp ldap policy attach adm consoleAdmin --group='CN=minio-admins,OU=Groups,DC=lab,DC=local'
mc idp ldap policy attach adm readonly    --group='CN=minio-readonly,OU=Groups,DC=lab,DC=local'
mc idp ldap policy entities adm
```
Test: Console `https://s3.lab.local:9001` → `alice` (admin) / `bob` (read-only). App lấy S3 key qua **Access Key (service account)** kế thừa policy từ group.

---

## 13. Giám sát Prometheus + Grafana

- Prometheus (TLS) scrape `/minio/v2/metrics/cluster` + `/node` (mục 9.5).
- Grafana `https://svc01.lab.local:3000` → datasource Prometheus (`https://prometheus:9090`, ca = lab-root-ca) → import dashboard chính thức MinIO (Cluster/Node/Bucket/Replication).

---

## 14. Kịch bản nghiệm thu POC

| # | Hạng mục | Cách kiểm | Kỳ vọng |
|---|----------|-----------|---------|
| 1 | HA EC:4 | poweroff minio4 | Vẫn đọc/ghi; bật lại → tự healing |
| 2 | Giới hạn EC:4 | tắt 2 node | Mất quorum (minh hoạ cần nhiều node ở prod) |
| 3 | Xác thực AD (LDAPS) | login alice/bob | Quyền đúng theo group; kết nối verify cert |
| 4 | Mã hóa chọn lọc | `mc stat` bucketA vs bucketB | A: SSE-KMS key-bucketA; B: không mã hóa |
| 5 | Khóa riêng | `vault kv list kv` | Thấy key-bucketA/key-bucketB tách biệt |
| 6 | **Auto-unseal** | restart container `vault` | Vault main **tự unseal**, S3 mã hóa hoạt động lại không cần nhập tay |
| 7 | Anchor seal | tắt `vault-transit` rồi restart `vault` | Vault main **không tự unseal được** → minh hoạ vai trò trust anchor |
| 8 | Xoay khóa | tạo key mới + set lại bucket | Object mới dùng key mới |
| 9 | WORM/Object Lock | bật versioning + object lock | Không xóa/sửa trong thời hạn retention |
| 10 | Giám sát | Grafana | Số liệu cluster realtime |

---

## 15. Lab → Production

| Hạng mục | Lab POC | Production |
|----------|---------|-----------|
| Node | 4 VM | **8+ node vật lý**, NVMe JBOD |
| EC | EC:4 (12 ổ) | EC:4/EC:6, 16 ổ/set, tách failure domain |
| KES | 1 container | **≥3 KES** HA, tách host |
| Vault | main + transit (file) | **Vault Raft HA 3 node**; auto-unseal bằng **HSM/cloud KMS** thay transit |
| CA | **AD CS Enterprise Root** | **Cùng AD CS** (hoặc Subordinate CA) – không đổi |
| TLS | HTTPS verify qua AD CA | Như lab, cert được quản lý/chu kỳ gia hạn tự động (autoenrollment) |
| LB | 1 HAProxy | 2 HAProxy + VRRP, 2 switch 100G |
| Audit | file/HTTP | đẩy SIEM (Qradar/Elastic) |

---

## 16. Cần bạn xác nhận

1. **Disk/EC:** giữ 3×300GB/node (12 ổ, EC:4) hay 4 ổ/node (16 ổ)?
2. **ESXi:** 6 VM trên 1 host (SPOF lab) hay ≥2 host (mình thêm anti-affinity)?
3. **LUKS:** có cần thêm **mã hóa đĩa OS (LUKS)** trên ổ data như lớp phòng vệ thứ hai ngoài SSE-KMS không?
4. **AD CS template:** dùng template **WebServer** mặc định (kèm bật SAN) hay bạn muốn mình mô tả tạo **template riêng** cho cert dịch vụ (tự enroll, vòng đời gia hạn)?
