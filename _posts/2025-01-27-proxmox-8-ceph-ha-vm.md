---
title: Proxmox 8.3 with Ceph Shared Storage, HA VM
categories:
- System
- Proxmox
- Linux
- Ceph
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Ảo hóa hạ tầng máy chủ với Proxmox-v8.3 + Ceph-v19.2.0
---

### 1. Giới thiệu 

- Khi triển khai hạ tầng ảo hóa cho doanh nghiệp, mình luôn ưu tiên sử dụng VMware vSphere do hiệu năng, độ ổn định và các tính năng nâng cao của VMware vSphere luôn vượt trội hơn các giải pháp ảo hóa khác. Chi phí licenses cho ESXi, vCenter, vSan là khá lớn, và hiện nay việc thay đổi các chính sách khi Broadcom mua lại VMware khiến cho xu hướng công nghệ chuyển dịch sang các giải pháp ảo hóa open source khác càng trở nên phổ biến hơn. 
  
- Trong bài viết này, mình sẽ chia sẻ về mô hình ảo hóa hạ tầng với Proxmox VE v8.3 kết hợp Ceph Squid v19.2.0 (internal) làm shared storage. Với những ưu điểm :
  - Miễn phí : do là giải pháp open source nên doanh nghiệp sẽ không phải tốn chi phí đầu tư về licenses.
  - Dễ sử dụng : thao tác quản lý qua WebUi tiện lợi.
  - Khả năng mở rộng : bổ sung thêm các node vật lý vào cluster một cách dễ dàng.
  - Chịu lỗi : các VM được bật tính năng High availability sẽ tự động migrate sang node khác khi node vật lý đang chứa VM bị lỗi.  

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/00.gif"/>

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Cài đặt Proxmox Host - Ảo hóa 1 máy chủ vật lý với Proxmox VE v8.3](#2-cài-đặt-proxmox-host---ảo-hóa-1-máy-chủ-vật-lý-với-proxmox-ve-v83)
- [3. WebUi quản lý Proxmox](#3-webui-quản-lý-proxmox)
- [4. Khởi tạo Proxmox cluster](#4-khởi-tạo-proxmox-cluster)
- [5. Khởi tạo Ceph cluster làm shared storage cho Proxmox cluster](#5-khởi-tạo-ceph-cluster-làm-shared-storage-cho-proxmox-cluster)
- [6. Cấu hình các disk vật lý thành Ceph OSD.](#6-cấu-hình-các-disk-vật-lý-thành-ceph-osd)
- [7. Tạo Ceph Pool relicate chứa VM trên Proxmox cluster](#7-tạo-ceph-pool-relicate-chứa-vm-trên-proxmox-cluster)
- [8. Tạo network cho VM trên Proxmox cluster](#8-tạo-network-cho-vm-trên-proxmox-cluster)
- [9. Tạo VM Linux vả Windows trên Proxmox cluster](#9-tạo-vm-linux-vả-windows-trên-proxmox-cluster)
- [10. Enable High Availability Virtual Machine (HA VM) trên Proxmox cluster](#10-enable-high-availability-virtual-machine-ha-vm-trên-proxmox-cluster)
- [11. Live migrate VM giữa các node PVE trên Proxmox cluster](#11-live-migrate-vm-giữa-các-node-pve-trên-proxmox-cluster)
- [12. Scale-out Proxmox cluster - thêm PVE node vào cluster](#12-scale-out-proxmox-cluster---thêm-pve-node-vào-cluster)
- [13. Lời kết](#13-lời-kết)

<br>

-  Các công nghệ sử dụng : 
   - Proxmox Virtual Environment v8.3
   - Ceph Squid v19.2.0

### 2. Cài đặt Proxmox Host - Ảo hóa 1 máy chủ vật lý với Proxmox VE v8.3 

- Mình Lab trên VMware Workstation 17 dùng 4 server cài Proxmox với cấu hình như sau:

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/01.png"/>

- Khởi động máy chủ boot từ ISO : proxmox-ve_8.3-1.iso
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/02.png"/>


- Chọn phân vùng ổ đĩa để cài đặt, trên thực tế chúng ta sẽ chọn cài đặt Proxmox lên phân vùng Raid 1 đã cấu hình sẵn của server vật lý : 
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/03.png"/>

- Đặt location và time zone cho hệ thống : 
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/04.png"/>

- Đặt mật khẩu root của server PVE : 
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/05.png"/>

- Khai báo network để quản lý cho server PVE : 
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/06.png"/>

- Kiểm tra lại các thông tin cấu hình : 
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/07.png"/>

- Quá trình cài đặt Proxmox VE bắt đầu : 
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/08.png"/>

- Hoàn tất cài đặt Proxmox VE, khởi động lại PVE server : 
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/09.png"/>

- Giao diện console của PVE server : 
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/10.png"/>


### 3. WebUi quản lý Proxmox  

- Đăng nhập WebUi của PVE host : 
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/11.png"/>

- Giao diện WebUi quản lý hệ thống ảo hóa Proxmox trên host PVE-1 : 
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/12.png"/>

- Các thông số monitor cơ bản của host PVE-1 : 
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/13.png"/>



### 4. Khởi tạo Proxmox cluster 

- Trên host PVE-1, tạo cluster để join các PVE-2, và PVE-3 : 
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/14.png"/>

- Khởi tạo cluster thành công : 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/15.png"/>

- Lấy token để join các host PVE còn lại vào cluster : 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/16.png"/>


- Join host PVE-2 vào cluster : 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/17.png"/>

- Host PVE-2 đã join vào cluster thành công: 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/18.png"/>

- Tương tự, join host PVE-3 vào cluster : 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/19.png"/>

- Hoàn tất khởi tạo Proxmox cluster và join các node vật lý vào cụm hạ tầng ảo hóa. Hiện tại cluster có 3 node, do Ceph cluster yêu cầu tối thiểu là 3 node nên mình sẽ triển khai trước 3 node vào cluster, các node còn lại sẽ join sau để scale-out cluster : 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/20.png"/>

- Capacity của Proxmox cluster với 3 node vừa khởi tạo : 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/21.png"/>


### 5. Khởi tạo Ceph cluster làm shared storage cho Proxmox cluster

- Cài đặt Ceph trên tất cả node PVE, các bước thực hiện trên PVE-1 : 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/22.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/23.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/24.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/25.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/26.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/27.png"/>

- Cài đặt Ceph trên PVE-2 : 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/28.png"/>

- Tạo daemon Ceph Mon trên PVE-2 :
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/29.png"/>

- Tạo daemon Ceph Mgr trên PVE-2 :

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/30.png"/>

- Cài đặt Ceph trên PVE-3 : 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/31.png"/>


- Tương tự, tạo daemon Ceph Mon và Ceph Mgr trên PVE-3 :

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/32.png"/>


### 6. Cấu hình các disk vật lý thành Ceph OSD.

- Mình add các disk vào các PVE host để cấu hình làm Ceph OSD : 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/33.png"/>


- Các disk vừa thêm vào đã detect trên Proxmox WebUi : 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/34.png"/>

- Add các disk làm Ceph OSD trên tất cả các node PVE :

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/35.png"/>

- Hoàn tất add Ceph OSD trên các PVE host :

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/36.png"/>


### 7. Tạo Ceph Pool relicate chứa VM trên Proxmox cluster

- Mình tạo pool HA-VM repicate 3 để chứa các VM : 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/37.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/38.png"/>


### 8. Tạo network cho VM trên Proxmox cluster

- Mình tạo thêm 1 subnet sử dụng NIC khác cho các VM hoạt động, trên thực tế chúng ta sẽ cấu hình vlan trunking tạo ra các vlan cho các VM sử dụng. 
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/39.png"/>

- Trong bài lab này, mình qui hoạch 1 subnet 10.50.10.0/24 để quản lý Proxmox cluster và 1 subnet 10.50.20.0/24 cho các VM.

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/40.png"/>


### 9. Tạo VM Linux vả Windows trên Proxmox cluster

- Upload các Image(.iso) để cài đặt OS cho VM. 
 
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/41.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/42.png"/>

- Tạo 1 VM Ubunu server 22.04.
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/43.png"/>

- Load iso đã upload để cài OS cho VM.
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/44.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/45.png"/>

- Chỉ định lưu VM trên Ceph pool để c61u hình HA cho VM.

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/46.png"/>

- Khai báo CPU và Ram cấp cho VM.
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/47.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/48.png"/>

- Chỉ định network sử dụng cho VM.
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/49.png"/>

- Kiểm tra lại các thông số cấu hình tạo VM.
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/50.png"/>

- Sau khi khởi tạo VM, start VM và tiến hành cài đặt OS.
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/51.png"/>

- VM đang được cài đặt OS Ubuntu server 22.04 LTS.
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/52.png"/>


- Hoàn tất cài đặt 1 VM chạy Ubuntu 22.04.
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/53.png"/>


- Tương tự, khởi tạo và cài đặt 1 VM chạy Windows Server 2022.
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/54.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/55.png"/>

- Hoàn tất cài đặt 1 VM chạy Windows Server 2022.
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/56.png"/>


### 10. Enable High Availability Virtual Machine (HA VM) trên Proxmox cluster 

- Trên node PVE-2 mình có 2 VM chạy Ubuntu 22.04 và Windows 2022 như hình bên dưới.
- Mình sẽ cấu hình chịu lỗi cho VM chạy Ubuntu để khi node PVE-2 down, VM Ubuntu sẽ tự động chuyển qua node PVE khác.
- VM Windows 2022 không được bật tính năng HA sẽ không tự động migrate qua node PVE khác. Để khôi phục VM chạy Windows chúng ta sẽ tạo VM trên node PVE khác vào load disk của VM Windows đang được lưu trên Ceph cluster.  

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/57.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/58.png"/>

- Cấu hình HA VM.

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/59.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/60.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/61.png"/>

- Shutdown node PVE-2 để kiểm tra failover, sau vài giây VM 104-Ubuntu đã tự động migrate và start trên node PVE-1.

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/62.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/63.png"/>


### 11. Live migrate VM giữa các node PVE trên Proxmox cluster 

- Trong quá trình vận hành, việc bảo trì các server vật lý đòi hỏi phải di chuyển các VM đang chạy services sang node vật lý khác để đảm bảo up time của services trong VM.

- Mình sẽ demo live migrate VM 104 trên node PVE-1 sang node mới PVE-3 trên Proxmox cluster có Ceph làm shared storage. 

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/64.png"/>

<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/65.png"/>

- Hoàn tất mirgate VM 104 sang node mới, trong suốt quá trình migrate VM vẫn hoạt động bình thường đảm bảo up time cho services trong VM.
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/66.png"/>

### 12. Scale-out Proxmox cluster - thêm PVE node vào cluster

- Trên PVE-1, copy token để join PVE-4 vào Proxmox cluster
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/67.png"/>


- Trên PVE-4, paste token và nhập password để join PVE-4 vào Proxmox cluster
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/68.png"/>

- Hoàn tất thêm node PVE-4 vào Proxmox cluster
  
<img src="/assets/img/2025-01-27-proxmox-8-ceph-ha-vm/69.png"/>

- Cấu hình network và Ceph trên PVE-4 tương tự các node trước.


### 13. Lời kết 

- Như vậy mình đã chia sẻ về cách triển khai một hạ tầng ảo hóa sử dụng công nghệ Proxmox VE cho các doanh nghiệp, các tổ chức.
  
- Với các tính năng như HA VM và Live Migrate VM giúp cho việc vận hành hạ tầng ảo hóa được đảm bảo an toàn, tránh rủi ro khi phần cứng vật lý server bị lỗi ảnh hưởng đến up time của services.

- Trong bài viết trên mình chỉ demo một số tính năng cơ bản, các bạn có thể tìm hiểu sâu thêm về các thiết kế nâng cao qua Google.
