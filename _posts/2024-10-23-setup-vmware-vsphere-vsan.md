---
title: Setup VMware vSphere, vSan
categories:
- System
- VMware
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Ảo hóa hạ tầng máy chủ và lưu trữ cho Tổ chức, Doanh nghiệp với vSphere 7
---

### 1. Giới thiệu 

- Các ứng dụng dịch vụ đòi hỏi môi trường triển khai để hoạt động, thường là các máy chủ (server). Trong hệ thống một doanh nghiệp có hàng trăm, hàng ngàn dịch vụ khác nhau, do đó đòi hỏi một lượng lớn máy chủ. Ảo hóa hạ tầng các máy chủ vật lý cung cấp ra thành các máy chủ ảo đem đến lợi ích lớn về việc tận dụng hiệu quả, tối đa tài nguyên phần cứng máy chủ vật lý, từ đó giảm chi phí đầu tư hạ tầng cho doanh nghiệp.

- Trong bài viết này mình chia sẻ về cách triển khai hệ thống ảo hóa với VMware vSphere và vSan, version 7.0. 

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/vmware.gif"/>

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Cài đặt ESXi Host - Ảo hóa 1 máy chủ vật lý với VMware ESXi](#2-cài-đặt-esxi-host---ảo-hóa-1-máy-chủ-vật-lý-với-vmware-esxi)
- [3. Khởi tạo vSphere cluster - Quản lý các máy chủ vật lý được ảo hóa](#3-khởi-tạo-vsphere-cluster---quản-lý-các-máy-chủ-vật-lý-được-ảo-hóa)
- [4. Khởi tạo vSan storage cho cluster - Quản lý không gian lưu trữ tập trung trên các ESXi host](#4-khởi-tạo-vsan-storage-cho-cluster---quản-lý-không-gian-lưu-trữ-tập-trung-trên-các-esxi-host)
- [5. Update license cho vSphere cluster](#5-update-license-cho-vsphere-cluster)
- [6. vSphere High Availability (HA) - chuyển đổi dự phòng máy ảo có downtime](#6-vsphere-high-availability-ha---chuyển-đổi-dự-phòng-máy-ảo-có-downtime)
  - [6.1. Enable vSphere High Availability](#61-enable-vsphere-high-availability)
  - [6.2. Tạo máy ảo và test tính năng vSphere HA](#62-tạo-máy-ảo-và-test-tính-năng-vsphere-ha)
- [7. vSphere Fault Tolerance (TF) - chuyển đổi dự phòng máy ảo không có downtime](#7-vsphere-fault-tolerance-tf---chuyển-đổi-dự-phòng-máy-ảo-không-có-downtime)
  - [7.1. Enable vSphere Fault Tolerance](#71-enable-vsphere-fault-tolerance)
  - [7.2. Test tính năng vSphere Fault Tolerance](#72-test-tính-năng-vsphere-fault-tolerance)
- [8. Lời kết](#8-lời-kết)

<br>

-  Các công nghệ sử dụng : 
   - VMware ESXi 7.0
   - VMware vCenter Server Appliance (VCSA) 7.0
   - VMware vSan 7.0

### 2. Cài đặt ESXi Host - Ảo hóa 1 máy chủ vật lý với VMware ESXi 

- Khởi động máy chủ vật lý boot từ ISO : VMware-ESXi-7.0U3n-21930508.x86_64.iso
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/001.png"/>

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/002.png"/>

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/003.png"/>

- Chọn ổ đã cài đặt ESXi
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/004.png"/>

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/005.png"/>

- Đặt password root của ESXi host

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/006.png"/>

- Tiến hành cài đặt ESXi 
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/007.png"/>

- Quá trình cài đặt ESXi bắt đầu 

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/008.png"/>

- Hoàn tất cài đặt ESXi , khởi động lại máy chủ 

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/009.png"/>

- Cấu hình IP tĩnh cho ESXi host
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/010.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/011.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/012.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/013.png" alt="drawing" />

- Đặt hostname cho máy chủ ESXi
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/014.png" alt="drawing" />

- Lưu cấu hình 

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/015.png" alt="drawing" />

- Máy chủ ESXi khởi động lại 
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/016.png" alt="drawing" />

- Địa chỉ Web UI quản lý ESXi host

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/017.png" alt="drawing" />

- Truy cập giao diện Web quản trị ESXi host
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/018.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/019.png" alt="drawing" />




### 3. Khởi tạo vSphere cluster - Quản lý các máy chủ vật lý được ảo hóa

- Thông số cấu hình các ESXi 7.0 host trong bài Lab

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/01.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/02.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/03.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/04.png" alt="drawing" />

<br>

- Thông số cấu hình máy chủ ảo chạy vCenter 7.0 (VCSA)

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/05.png" alt="drawing" />


- Truy cập giao diện Web quản trị của vCenter , bắt đầu cấu hình quản lý vSphere 

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/06.png" alt="drawing" />



- Tạo Datacenter 

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/07.png" alt="drawing" />
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/08.png" alt="drawing" />


- Tạo Cluster
 
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/09.png" alt="drawing" />
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/10.png" alt="drawing" />


- Thêm ESXi host vào Cluster

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/11.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/12.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/13.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/14.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/15.png" alt="drawing" />

- Tiến trình cấu hình join các ESXi host vào cluster bắt đầu
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/16.png" alt="drawing" />

- Ở đây mình join các ESXi qua Quickstart cluster nên các ESXi sau khi join thành công sẽ ở trạng thái bảo trì (Maintenance Mode)
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/17.png" alt="drawing" />

- Tiếp tục các bước cấu hình khác 
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/18.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/19.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/20.png" alt="drawing" />

- Hoàn tất cấu hình một cụm máy chủ ảo hóa vSphere 
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/21.png" alt="drawing" />




### 4. Khởi tạo vSan storage cho cluster - Quản lý không gian lưu trữ tập trung trên các ESXi host

- Thông số các disk trên 1 ESXi host trong bài Lab
    - 01 disk 700 GiB : mình cài OS ESXi
    - 01 disk 500 GiB : mình cấu hình làm Cache Tier 
    - 03 disk 500 GiB : mình cấu hình làm Capacity Tier 


<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/22.png" alt="drawing" />


- Enable vSan service và cấu hình claim disk trên các ESXi host

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/23.png" alt="drawing" />

- Ở đây mình chỉ dùng Single cluster 
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/24.png" alt="drawing" />

- Các option nâng cao các bạn tìm hiểu thêm để biết cách dùng, ở đây mình không dùng đến 
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/25.png" alt="drawing" />

- Cấu hình claim disk, khai báo disk nào làm Cache tier , disk nào làm Capacity tier trên mỗi host 
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/26.png" alt="drawing" />

- Fault domain mình dùng mặc định 1 miền chịu lỗi 

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/27.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/28.png" alt="drawing" />

- Tiến trình cấu hình vSan trên các ESXi host bắt đầu 
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/29.png" alt="drawing" />

- vSan yêu cầu enable vSan option trong VMkernel adapters
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/30.png" alt="drawing" />

- Cấu hình enable vSan network trên các ESXi host
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/31.png" alt="drawing" />

- Hoàn tất cấu hình vSan cho cluster 

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/32.png" alt="drawing" />

- Update HCL DB without internet :
[download](https://partnerweb.vmware.com/service/vsan/all.json )

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/33.png" alt="drawing" />

- Update vSAN release catalog DB without internet :
[download](https://vcsa.vmware.com/ph/api/v1/results?deploymentId=2d02e861-7e93-4954-9a73-b08692a330d1&collectorId=VsanCloudHealth.6_5&objectId=0c3e9009-ba5d-4e5f6-bae8-f25ec506d219&type=vsan-updates-json )

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/34.png" alt="drawing" />


### 5. Update license cho vSphere cluster  

- Add license 

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/35.png" alt="drawing" />

- Các license sử dụng trong bài lab : ESXi, vCenter, vSan
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/36.png" alt="drawing" />

- Update license cho vCenter 7.0

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/37.png" alt="drawing" />

- Update license cho ESXi 7.0

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/38.png" alt="drawing" />

- Update license cho vSan 7.0

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/39.png" alt="drawing" />



### 6. vSphere High Availability (HA) - chuyển đổi dự phòng máy ảo có downtime

- vSphere High Availability là tính năng failover cho VM. 
- VM sẽ tự chuyển sang ESXi host khác khi ESXi host đang chứa VM gặp sự cố. Do các file cấu hình máy ảo lưu trên storage dùng chung là vSan.
  
- Quá trình chuyển đổi có downtime do VM sẽ restart trên ESXI host mới.
 
#### 6.1. Enable vSphere High Availability

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/40.png" alt="drawing" />

- Các hành động sẽ thực hiện khi sự cố xảy ra trên cụm vSphere, các bạn tim hiểu thêm để biết cách dùng, mình chỉ dùng để demo.

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/41.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/42.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/43.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/44.png" alt="drawing" />

- Hoàn tất cấu hình vSphere HA
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/45.png" alt="drawing" />


#### 6.2. Tạo máy ảo và test tính năng vSphere HA

- Tạo máy ảo từ template 

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/46.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/47.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/48.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/49.png" alt="drawing" />

- Chọn datastore lưu máy ảo là vSan
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/50.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/51.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/52.png" alt="drawing" />

- Cấu hình cấp cho máy ảo 
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/53.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/54.png" alt="drawing" />

- Hoàn tất tạo một máy ảo, máy ảo nhận IP 137, đang chạy trên máy chủ vật lý ESXi host 133
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/55.png" alt="drawing" />

- Test vSphere HA : shutdown hoặc disable network ESXi host , máy ảo sẽ tự restart chuyển qua ESXi host khác available. Máy ảo đã tự chuyển qua máy chủ vật lý ESXi host 132 (máy ảo sẽ phải khởi động lại nên các ứng dụng dịch vụ sẽ bị gián đoạn trong thời gian chuyển đổi dự phòng)

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/56.png" alt="drawing" />



### 7. vSphere Fault Tolerance (TF) - chuyển đổi dự phòng máy ảo không có downtime

- Fault Tolerance là tính năng replicate VM trên 2 ESXi host đồng thời. Sử dụng cho các server chạy các dịch vụ cực kỳ quan trọng không thể gián đoạn.

- Khi VM chạy Fault Tolerance sẽ ảnh hưởng đến hiệu suất của VM và tiêu tốn nhiều tài nguyên của hệ thống hơn, cần cân nhắc trước khi sử dụng.

- Thực tế thì khi chuyển đổi dự phòng VM với Fault Tolerance sẽ mất kết nối vài giây đến máy ảo, tương tự thao tác migate VM.

- Để chạy tính năng Fault Tolerance cho VM cần enable vMotion và Fault Tolerance logging trong VMkernel adapter trên các ESXi host

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/57.png" alt="drawing" />


#### 7.1. Enable vSphere Fault Tolerance

- Để enable tính năng Fault Tolerance cho VM cần power off VM trước

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/58.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/59.png" alt="drawing" />

- Chọn ESXi host chứa VM secondary được replicate 
  
<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/60.png" alt="drawing" />

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/61.png" alt="drawing" />


- Hoàn tất cấu hình Fault Tolerance cho VM

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/62.png" alt="drawing" />


- VM state primary, máy ảo chính đang chạy trên máy chủ vật lý ESXi host 132

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/63.png" alt="drawing" />


- VM state secondary, , máy ảo dự phòng đang trên máy chủ vật lý ESXi host 134

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/64.png" alt="drawing" />


#### 7.2. Test tính năng vSphere Fault Tolerance


- Test vSphere Fault Tolerance : shutdown hoặc disable network ESXi host chứa VM state primary , VM primary sẽ start trên ESXi host khác available.

- Các bước chuyển đổi được hiểu như sau : 
  - Step 1: ESXi host down, VM primary trên 132 mất kết nối
  - Step 2: VM secondary trên 134 chuyển trạng thái thành primary
  - Step 3: VM secondary được start trên ESXi host khác 133 
  - Step 4: VM secondary trên 133 chuyển thành primary
  - Step 5: VM primary trên 134 lại chuyển về trạng thái secondary

<img src="/assets/img/2024-10-23-setup-vmware-vsphere-vsan/65.png" alt="drawing" />


### 8. Lời kết 

- Như vậy mình đã chia sẻ về cách triển khai một hạ tầng ảo hóa và không gian lưu trữ tập trung sử dụng công nghệ VMware vSphere, vSan cho các doanh nghiệp, các tổ chức.
- Trong bài viết trên mình chỉ demo một số tính năng cơ bản, các bạn có thể tìm hiểu sâu thêm về các thiết kế nâng cao qua Google.
- VMware vSphere là công nghệ nổi tiếng trên thế giới được sử dụng rộng rãi trong các doanh nghiệp, có độ ổn định cao, nhược điểm là chi phí cao cho các loại license như mình sử dụng trong bài lab này.


