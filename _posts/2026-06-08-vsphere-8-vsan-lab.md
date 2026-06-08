---
title: "VMware vSphere 8 + vSAN Lab — ESXi 8.0U3j & vCenter 8.0.3"
categories:
- VMware
tags:
- vsphere
- vsan
- esxi
- vcenter
- vmware
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Triển khai cụm hypervisor vSphere 8 với vSAN — bản mới nhất ESXi 8.0U3j & vCenter 8.0.3
---

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/topo.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/00.png)


Kể từ khi Broadcom hoàn tất việc mua lại VMware cuối năm 2023, vSphere đã trải qua nhiều thay đổi về mô hình cấp phép và subscription, nhưng về mặt kỹ thuật vẫn tiếp tục phát triển mạnh. vSphere 8 mang theo vSAN 8 với kiến trúc ESA mới, cải tiến vSphere Lifecycle Manager, và giao diện vCenter được làm mới đáng kể.

Một điểm thay đổi lớn về licensing: Broadcom bỏ hoàn toàn mô hình **perpetual license** và chuyển sang **subscription theo core** — không còn mua đứt như thời vSphere 7. Broadcom cũng gộp sản phẩm lại thành hai gói chính: **VMware vSphere Foundation (VVF)** giá ~$135/core/năm và **VMware Cloud Foundation (VCF)** giá ~$350/core/năm, với mức tối thiểu 16 core mỗi CPU socket. Mức tăng giá so với trước là 800–1.500% — không phải con số ước tính mà là mức đã được ghi nhận rộng rãi trong ngành. Điều này khiến nhiều doanh nghiệp vừa và nhỏ phải cân nhắc lại hoặc chuyển sang các nền tảng open-source như Proxmox VE. Với các tổ chức lớn đã đầu tư sâu vào hệ sinh thái VMware, vSphere 8 vẫn là lựa chọn ổn định — nhưng bài toán chi phí cần được tính kỹ hơn trước rất nhiều.

Bài lab này mình setup cụm 3 ESXi host, sử dụng các bản mới nhất hiện tại, bao gồm custom driver cho server dòng Dell PowerEdge, để chia sẻ các bước triển khai một cụm hạ tầng ảo hóa với VMware vSphere 8 và khám phá các tính năng mới:

- **ESXi 8.0U3j** — build 25429389 (27/05/2026)
- **vCenter Server 8.0.3** — build 25413364 (27/05/2026)
- **Dell Custom Addon 8.0.3 A08** — build 24859861 (version mới nhất hiện tại dành cho server dòng Dell PowerEdge)

> **Lưu ý:** Mình setup lab test với mục đích chia sẻ và nghiên cứu tính năng, không phải môi trường production — toàn bộ lab chạy trên ESXi 7.0.3 (nested virtualization).

- [1. Lab Topology](#1-lab-topology)
- [2. Cài đặt ESXi 8.0U3j](#2-cài-đặt-esxi-80u3j)
- [3. Cài đặt Dell Addon driver cho ESXi](#3-cài-đặt-dell-addon-driver-cho-esxi)
- [4. Deploy vCenter Server Appliance 8.0.3](#4-deploy-vcenter-server-appliance-803)
- [5. Tạo vSphere Datacenter \& Cluster](#5-tạo-vsphere-datacenter--cluster)
- [6. Cấu hình vSAN](#6-cấu-hình-vsan)
- [7. Active licenses](#7-active-licenses)
- [8. Lời kết](#8-lời-kết)

### 1. Lab Topology

ISO sử dụng trong bài lab:

| Component | File |
|-----------|------|
| ESXi | `VMware-VMvisor-Installer-8.0U3j-25429389.x86_64.iso` |
| vCenter | `VMware-VCSA-all-8.0.3-25413364.iso` |

Mình setup 3 ESXi host, mỗi host có 3 disk: 1 disk cài OS ESXi và 2 disk dành cho vSAN (1 cache tier + 1 capacity tier). vCenter được deploy dưới dạng VCSA trên esxi01. 

| Node | IP | vCPU | RAM | Disk |
|------|----|------|-----|------|
| esxi01 | 10.10.200.11 | 16 | 32 GB | 1 TB (OS) · 100 GB (vSAN cache) · 500 GB (vSAN cap) · 500 GB (vSAN cap) |
| esxi02 | 10.10.200.12 | 16 | 32 GB | 1 TB (OS) · 100 GB (vSAN cache) · 500 GB (vSAN cap) · 500 GB (vSAN cap) |
| esxi03 | 10.10.200.13 | 16 | 32 GB | 1 TB (OS) · 100 GB (vSAN cache) · 500 GB (vSAN cap) · 500 GB (vSAN cap) |
| vCenter (VCSA) | 10.10.200.50 | — | — | deploy trên esxi01 |

Lab chạy trên 3 VM:

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/01.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/02.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/03.png)

### 2. Cài đặt ESXi 8.0U3j

Cài đặt ESXi lần lượt trên cả 3 host — quy trình giống nhau, chỉ khác IP và hostname ở bước cấu hình network cuối.

Boot máy chủ từ ISO `VMware-VMvisor-Installer-8.0U3j-25429389.x86_64.iso`. Màn hình welcome của ESXi 8 hiện ra, nhấn **Enter** để bắt đầu.

Chấp nhận EULA (**F11**), chọn disk cài đặt OS — mình chọn disk 1 TB đầu tiên. Đặt password root, xác nhận cài đặt. 

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/04.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/05.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/06.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/07.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/08.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/09.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/10.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/11.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/12.png)

Sau khi boot xong, vào DCUI (**F2** → đăng nhập root) → **Configure Management Network** → **IPv4 Configuration** để đặt IP tĩnh. Mình cấu hình lần lượt cho từng host:

| Host | IP | Subnet Mask | Gateway | Hostname | DNS |
|------|----|-------------|---------|----------|-----|
| esxi01 | 10.10.200.11 | 255.255.255.0 | 10.10.200.1 | esxi01 | 8.8.8.8 |
| esxi02 | 10.10.200.12 | 255.255.255.0 | 10.10.200.1 | esxi02 | 8.8.8.8 |
| esxi03 | 10.10.200.13 | 255.255.255.0 | 10.10.200.1 | esxi03 | 8.8.8.8 |

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/13.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/14.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/15.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/16.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/17.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/18.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/19.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/20.png)


Truy cập `https://10.10.200.11`, `https://10.10.200.12`, `https://10.10.200.13` để kiểm tra web UI của từng host hoạt động bình thường.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/21.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/22.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/23.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/24.png)

### 3. Cài đặt Dell Addon driver cho ESXi

VMware chỉ cung cấp base ESXi image — chưa bao gồm các driver đặc thù của Dell PowerEdge như PERC RAID controller, iDRAC integration hay OpenManage. Dell phát hành riêng addon bundle **Dell_Addon_8.0.3_A08** (build 24859861) — bản mới nhất 2026/06 cho ESXi 8.0U3, download tại [Dell Support Portal](https://www.dell.com/support).

**Bước 1 — Upload addon lên host**

Copy file `Dell_Addon_8.0.3_A08.zip` lên thư mục `/tmp` của host qua SCP:

```bash
scp Dell_Addon_8.0.3_A08.zip root@10.10.200.11:/tmp/
```

**Bước 2 — Đưa host vào Maintenance Mode**

SSH vào host và chạy:

```bash
esxcli system maintenanceMode set --enable true
```

**Bước 3 — Apply addon**

```bash
esxcli software component apply -d /tmp/Dell_Addon_8.0.3_A08.zip
```

Output sẽ liệt kê các component đã được register vào ESXi image. Trên bare-metal PowerEdge, các driver PERC, iDRAC, OpenManage sẽ bind vào phần cứng tương ứng. Trong môi trường nested (ESXi chạy dưới dạng VM), addon vẫn cài thành công nhưng các driver hardware-specific sẽ không bind vào device nào — hoàn toàn bình thường.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/25.png)

**Bước 4 — Reboot và thoát Maintenance Mode**

```bash
reboot
```

Sau khi host boot xong:

```bash
esxcli system maintenanceMode set --enable false
```

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/26.png)

Lặp lại 4 bước trên cho `esxi02` (10.10.200.12) và `esxi03` (10.10.200.13). Khi join cluster vCenter sau này, hardware Dell sẽ được nhận diện đầy đủ trong inventory — iDRAC, storage controller, thông tin firmware.

### 4. Deploy vCenter Server Appliance 8.0.3

Mount ISO `VMware-VCSA-all-8.0.3-25413364.iso`, chạy installer tương ứng với OS:
- **Windows**: `vcsa-ui-installer\win32\installer.exe`
- **macOS**: `vcsa-ui-installer/mac/VMware VCSA Installer.app`
- **Linux**: `vcsa-ui-installer/lin64/installer`

Quá trình deploy VCSA gồm 2 stage.

**Stage 1 — Deploy appliance**

Chọn **Install** → **vCenter Server with Embedded PSC**.

Mình deploy VCSA lên esxi01 — nhập thông tin host đích:
- ESXi Host: `10.10.200.11`
- User: `root` · Password: `<password>`

Đặt tên VM cho VCSA (ví dụ `vcenter`) và password root appliance: `<password>`.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/27.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/28.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/29.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/30.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/31.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/32.png)


Chọn **Deployment Size: Tiny** — phù hợp cho lab, yêu cầu 2 vCPU và 14 GB RAM.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/33.png)

Chọn datastore trên esxi01 để chứa VCSA VM.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/34.png)

Cấu hình network cho VCSA:

| Trường | Giá trị |
|--------|---------|
| IP | 10.10.200.50 |
| Subnet mask | 255.255.255.0 |
| Gateway | 10.10.200.1 |
| DNS | 8.8.8.8 |

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/35.png)

Review và **Finish** — Stage 1 deploy OVA lên esxi01, mất khoảng 10–15 phút.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/36.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/37.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/38.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/38a.png)

**Stage 2 — Cấu hình vCenter**

Installer tự chuyển sang Stage 2 sau khi Stage 1 hoàn tất.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/39.png)

Cấu hình **NTP**: `vn.pool.ntp.org`.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/40.png)

Cấu hình **vCenter Single Sign-On**:

| Trường | Giá trị |
|--------|---------|
| SSO domain name | `anhlx.local` |
| SSO username | `administrator` |
| SSO password | `<password>` |

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/41.png)

Bỏ chọn **CEIP** nếu không muốn gửi telemetry về Broadcom. Review và **Finish** — Stage 2 mất thêm 10–15 phút để khởi động các dịch vụ vCenter.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/42.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/43.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/44.png)

Sau khi hoàn tất, truy cập `https://10.10.200.50` và đăng nhập bằng `administrator@anhlx.local` để vào vSphere Client.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/45.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/46.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/47.png)

### 5. Tạo vSphere Datacenter & Cluster

Mình tạo Datacenter và Cluster để tổ chức và quản lý 3 ESXi host từ vCenter.

**Tạo Datacenter:** Từ vSphere Client, click chuột phải vào vCenter object → **New Datacenter** → đặt tên (ví dụ `DC-Lab`) → **OK**.

**Tạo Cluster:** Click chuột phải vào Datacenter vừa tạo → **New Cluster** → đặt tên (ví dụ `Cluster-01`). Ở wizard, mình tắt DRS, HA và vSAN — sẽ cấu hình vSAN riêng ở section sau. Nhấn **Finish**.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/48.png)

**Add ESXi hosts:** Click chuột phải vào Cluster → **Add Hosts** → lần lượt điền IP và credential của 3 host:

| Host | IP | User | Password |
|------|----|------|----------|
| esxi01 | 10.10.200.11 | root | `<password>` |
| esxi02 | 10.10.200.12 | root | `<password>` |
| esxi03 | 10.10.200.13 | root | `<password>` |

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/49.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/50.png)

vCenter connect và add 3 host vào cluster. Sau khi hoàn tất, inventory hiển thị đủ 3 host ở trạng thái **Connected**.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/51.png)

### 6. Cấu hình vSAN

Mình dùng vSAN **OSA** (Original Storage Architecture) — không yêu cầu NVMe, phù hợp với virtual disk thông thường trong lab. Mỗi host có 3 disk dành cho vSAN, tổ chức thành 1 disk group:

| Tier | Disk | Dung lượng | Vai trò |
|------|------|------------|---------|
| Cache | disk2 | 100 GB | Write buffer — nhỏ hơn capacity là đúng kiến trúc thực tế |
| Capacity | disk3 | 500 GB | Lưu trữ dữ liệu thực tế |
| Capacity | disk4 | 500 GB | Lưu trữ dữ liệu thực tế |

Với 3 host × 2 capacity disk × 500 GB và **FTT=1 (RAID-1)**, tổng raw capacity là 3 TB, dung lượng usable khoảng **~1.5 TB**, chịu được tối đa 1 host failure.

**Enable vSAN:**

Vào **Cluster** → **Configure** → **vSAN** → **Services** → **Configure vSAN**.

Chọn **vSAN HCI** — hosts vừa chạy compute vừa cung cấp storage (hyperconverged). Tiếp theo chọn **Single site vSAN cluster**.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/52.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/53.png)

Mình không enable deduplication/compression để giữ cấu hình đơn giản cho lab.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/54.png)

**Claim disks:**

Mình map disk cho từng host:
- disk2 → **Flash Cache** (Cache Tier)
- disk3 → **Capacity** (Capacity Tier)
- disk4 → **Capacity** (Capacity Tier)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/55.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/56.png)

Review và **Finish**. vSAN cluster khởi tạo và tạo datastore `vsanDatastore`.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/57.png)
![](/assets/img/2026-06-08-vsphere-8-vsan-lab/57a.png)

Nếu wizard cảnh báo thiếu **vSAN** flag trong VMkernel adapter, mình enable flag vSAN trên `vmk0` từ vCenter — vào **Host → Configure → Networking → VMkernel adapters → vmk0 → Edit Settings**, tick **vSAN** trong phần Available services → **OK**. Lặp lại cho esxi02 và esxi03.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/58.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/59.png)

**Kiểm tra vSAN Health:**

Vào **Cluster** → **Monitor** → **vSAN** → **Skyline Health**. Trong nested lab sẽ có một số warning expected — có thể bỏ qua. Miễn là vSAN datastore `vsanDatastore` hiện capacity và cluster Summary hiển thị storage usage là vSAN hoạt động bình thường.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/60.png)

### 7. Active licenses

Mình assign license cho toàn bộ cluster từ một chỗ duy nhất: **Administration → Licensing → Licenses**.

**Bước 1 — Add license key**

Vào **Administration → Licensing → Licenses** → **Add New Licenses** → nhập license key

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/61.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/62.png)


**Bước 2 — Assign license cho vCenter**

Tab **Assets → vCenter Server Systems** → chọn vCenter → **Assign License** → chọn vCenter license → **OK**.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/63.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/63a.png)

**Bước 3 — Assign license cho ESXi hosts**

Tab **Assets → Hosts** → chọn cả 3 host (Ctrl+click) → **Assign License** → chọn ESXi license → **OK**.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/64.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/65.png)

**Bước 4 — Assign license cho vSAN cluster**

Tab **Assets → vSAN Clusters** → chọn `Cluster-01` → **Assign License** → chọn vSAN license → **OK**.

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/66.png)

![](/assets/img/2026-06-08-vsphere-8-vsan-lab/67.png)

Sau khi assign xong, banner "expired or expiring licenses" trên vSphere Client sẽ biến mất.

### 8. Lời kết

Như vậy mình đã hoàn tất setup cụm hypervisor 3 node với bản VMware vSphere 8 mới nhất — ESXi 8.0U3j, vCenter 8.0.3, Dell Addon A08 và vSAN OSA. So với vSphere 7, quy trình triển khai không thay đổi nhiều, nhưng giao diện vCenter 8 được cải thiện rõ rệt — đặc biệt vSAN Health dashboard và cluster configuration wizard trực quan và chi tiết hơn hẳn. Bài tiếp theo mình sẽ khai thác tiếp cụm này để lab vSphere HA, Lifecycle Manager, và thử nghiệm vSAN 8 ESA khi môi trường hỗ trợ NVMe.
