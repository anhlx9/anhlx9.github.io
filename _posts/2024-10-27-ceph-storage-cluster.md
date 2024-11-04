---
title: Ceph Storage Cluster
categories:
- System
- Storage
- Linux
- Ceph
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Xây dựng cụm server lưu trữ dữ liệu cho doanh nghiệp với Ceph
---


### 1. Giới thiệu 

- Ảo hóa máy chủ và không gian lưu trữ dữ liệu lớn là hai nhu cầu cần thiết đối với các tổ chức, doanh nghiệp. 

- Có nhiều chủng loại thiết bị phần cứng vật lý được thiết kế dành riêng cho việc lưu trữ, các bạn hãy xem ví dụ ở 2 thiết bị bên dưới. 
    
<img src="/assets/img/2024-10-27-ceph-storage-cluster/000-dell-server.jpg" />
Một máy chủ storage với nhiều disk
  
<img src="/assets/img/2024-10-27-ceph-storage-cluster/000-synology-storage.png"/>
Một thiết bị lưu trữ mạng Synology

- Có thể thấy các thiết bị đều sẽ có giới hạn về slot gắn disk, để có khả năng chịu lỗi khi 1 thiết bị hư hỏng linh kiện điện tử hoặc nâng tổng dung lượng cho cụm lên đến hàng Petabyte trong bài viết này mình sẽ chia sẻ về mô hình Ceph storage cluster.

- Kiến trúc Ceph storage cluster:
  
<img src="/assets/img/2024-10-27-ceph-storage-cluster/000-ceph-diagram.gif"/>

- Một mô hình ví dụ cho production:

<img src="/assets/img/2024-10-27-ceph-storage-cluster/01.png"/>

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Các công nghệ sử dụng](#2-các-công-nghệ-sử-dụng)
- [3. Mô hình triển khai](#3-mô-hình-triển-khai)
- [4. Triển khai Ceph cluster](#4-triển-khai-ceph-cluster)
  - [4.1. Install cephadm , ceph-common](#41-install-cephadm--ceph-common)
  - [4.2. Khởi tạo Ceph cluster](#42-khởi-tạo-ceph-cluster)
  - [4.3. Deploy Ceph Monitor Daemon (mon), Ceph Manager Daemon (mgr)](#43-deploy-ceph-monitor-daemon-mon-ceph-manager-daemon-mgr)
  - [4.4. Deploy Ceph Object Daemon (osd)](#44-deploy-ceph-object-daemon-osd)
- [5. Ceph admin dashboard - Grafana monitor dashboard](#5-ceph-admin-dashboard---grafana-monitor-dashboard)
- [6. Ceph block storage](#6-ceph-block-storage)
  - [6.1. Ví dụ tạo Pool replicate 3, tạo Image size 100GB](#61-ví-dụ-tạo-pool-replicate-3-tạo-image-size-100gb)
  - [6.2. Sử dụng Ceph block storage trên Ubuntu server](#62-sử-dụng-ceph-block-storage-trên-ubuntu-server)
  - [6.3. Sử dụng Ceph block storage trên Windows server](#63-sử-dụng-ceph-block-storage-trên-windows-server)

### 2. Các công nghệ sử dụng
- Server Ubuntu 22.04 
- Ceph version Squid-19.2.0

<img src="/assets/img/2024-10-27-ceph-storage-cluster/000-ceph-releases.png"/>


### 3. Mô hình triển khai 

- Mình sử dụng 6 server Ubuntu 22.04 LTS trong bài lab với cấu hình như sau 

  - 3 x server mon quản lý cluster : 1 disk cài OS
  
<img src="/assets/img/2024-10-27-ceph-storage-cluster/02.png"/>

<img src="/assets/img/2024-10-27-ceph-storage-cluster/03.png"/>

<img src="/assets/img/2024-10-27-ceph-storage-cluster/04.png"/>


  - 3 x server ods làm storage node : 1 disk cài OS , các disk còn lại cấu hình làm ceph devices

<img src="/assets/img/2024-10-27-ceph-storage-cluster/05.png"/>

<img src="/assets/img/2024-10-27-ceph-storage-cluster/06.png"/>

<img src="/assets/img/2024-10-27-ceph-storage-cluster/07.png"/>



### 4. Triển khai Ceph cluster 

- Cài đặt các package common trên tất cả các node
  
``` bash
apt update 

apt install -y podman tree screen net-tools git wget bash-completion software-properties-common jq ntpdate iptables-persistent netfilter-persistent lldpd arp-scan ntp ntpdate

```

#### 4.1. Install cephadm , ceph-common

- Mình dùng Ceph Squid version lastest hiện tại là 19.2.0

- Trên 3 node Ceph-Mon install cephadm , ceph-common

```bash
# On 3 node MON
# Install cephadm  
CEPH_RELEASE=19.2.0
curl --silent --remote-name --location https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
chmod +x cephadm
mv cephadm  /usr/local/bin/

# Install ceph-common
cephadm add-repo --version 19.2.0
cephadm install ceph-common

# Confirm
ceph -v
```

<img src="/assets/img/2024-10-27-ceph-storage-cluster/08.png"/>

#### 4.2. Khởi tạo Ceph cluster

- Tạo thư mục chứa ceph config 
```bash
mkdir -p /etc/ceph
```

- Khởi tạo Ceph cluster , chỉ thực hiện trên 1 node ceph-mon-11

> https://www.ibm.com/docs/en/storage-ceph/5?topic=dashboard-ceph-installation-access 


```bash
# Only on ceph-mon-11
# bootstrap ceph cluster

cephadm bootstrap \
  --allow-fqdn-hostname \
  --mon-ip 10.10.1.11 \
  --dashboard-password-noupdate \
  --initial-dashboard-user admin \
  --initial-dashboard-password adminpwd
```

- Khởi tạo cluster thành công 
  
<img src="/assets/img/2024-10-27-ceph-storage-cluster/09.png"/>

- Giao diện Web quản trị Ceph cluster 

<img src="/assets/img/2024-10-27-ceph-storage-cluster/10.png"/>

<img src="/assets/img/2024-10-27-ceph-storage-cluster/11.png"/>


- Trạng thái hiện tại của cluster 

<img src="/assets/img/2024-10-27-ceph-storage-cluster/12.png"/>


- Tiếp theo chúng ta sẽ add 2 node mon còn lại vào cluster. 
- Ceph adm quản lý cluster thông qua ssh vào các node.
- Trước tiên cần copy ssh key của Ceph lên 2 node mon còn lại.
  
```bash
# From ceph-mon-11 Copy Ceph SSH key to others Control Nodes
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-mon-12
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-mon-13
```
<img src="/assets/img/2024-10-27-ceph-storage-cluster/13.png"/>


- Add node vào cluster 

```bash
# Add Control Nodes to the Ceph cluster
ceph orch host add ceph-mon-12
ceph orch host add ceph-mon-13

# Label the nodes with _admin
ceph orch host label add ceph-mon-11 _admin
ceph orch host label add ceph-mon-12 _admin
ceph orch host label add ceph-mon-13 _admin

# Sync config to to Control Nodes
for i in 10.10.1.11 10.10.1.12 10.10.1.13 ; do rsync -av /etc/ceph/{ceph.conf,ceph.client.admin.keyring,ceph.pub} root@$i:/etc/ceph/ ; done 

# Confirm host
ceph orch host ls --detail
```

<img src="/assets/img/2024-10-27-ceph-storage-cluster/14.png"/>



#### 4.3. Deploy Ceph Monitor Daemon (mon), Ceph Manager Daemon (mgr)

- Gắn nhãn các node làm mon, mgr 

```bash
## Label the nodes with mon,mgr
ceph orch host label add ceph-mon-11 mon
ceph orch host label add ceph-mon-12 mon
ceph orch host label add ceph-mon-13 mon

ceph orch host label add ceph-mon-11 mgr
ceph orch host label add ceph-mon-12 mgr
ceph orch host label add ceph-mon-13 mgr

# Confirm 
ceph orch host ls --detail
```

<img src="/assets/img/2024-10-27-ceph-storage-cluster/15.png"/>

- Deploy các deamon 

```bash
# Apply configs mon , mgr
ceph orch apply mon --placement="3 ceph-mon-11 ceph-mon-13 ceph-mon-13"
ceph orch apply mgr --placement="3 ceph-mon-11 ceph-mon-13 ceph-mon-13"

# Confim 
ceph -s
ceph orch ls
ceph orch ps 
```

<img src="/assets/img/2024-10-27-ceph-storage-cluster/16.png"/>

- Ceph dùng podman để chạy các deamon 

<img src="/assets/img/2024-10-27-ceph-storage-cluster/17.png"/>



#### 4.4. Deploy Ceph Object Daemon (osd)

- Copy ssh key của Ceph lên các OSD node

```bash
# Only On ceph-mon-11
# Copy SSH Key To OSD Nodes
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-osd-14
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-osd-15
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-osd-16

# Add hosts to cluster 
ceph orch host add ceph-osd-14
ceph orch host add ceph-osd-15
ceph orch host add ceph-osd-16

# Give new nodes labels 
ceph orch host label add ceph-osd-14 osd
ceph orch host label add ceph-osd-15 osd
ceph orch host label add ceph-osd-16 osd

# Confirm hosts
ceph orch host ls --detail
```

<img src="/assets/img/2024-10-27-ceph-storage-cluster/19.png"/>

- Kiểm tra các disk có thể sử dụng cho cluster 

> A storage device is considered available if all of the following conditions are met:
> 
> The device must have no partitions.
> 
> The device must not have any LVM state.
> 
> The device must not be mounted.
> 
> The device must not contain a file system.
> 
> The device must not contain a Ceph BlueStore OSD.
> 
> The device must be larger than 5 GB.

```bash
# View all devices on storage nodes
ceph orch device ls --refresh
```

- Các disk trên các OSD node thỏa điều kiện để sử dụng trong Ceph cluster
  
<img src="/assets/img/2024-10-27-ceph-storage-cluster/20.png"/>


- Cấu hình Ceph sử dụng các disk khả dụng  
  
```bash
# Tell Ceph to consume the available and unused storage device 
ceph orch daemon add osd ceph-osd-14:/dev/sdb
ceph orch daemon add osd ceph-osd-14:/dev/sdc
ceph orch daemon add osd ceph-osd-14:/dev/sdd

ceph orch daemon add osd ceph-osd-15:/dev/sdb
ceph orch daemon add osd ceph-osd-15:/dev/sdc
ceph orch daemon add osd ceph-osd-15:/dev/sdd

ceph orch daemon add osd ceph-osd-16:/dev/sdb
ceph orch daemon add osd ceph-osd-16:/dev/sdc
ceph orch daemon add osd ceph-osd-16:/dev/sdd
```

<img src="/assets/img/2024-10-27-ceph-storage-cluster/21.png"/>

<img src="/assets/img/2024-10-27-ceph-storage-cluster/21.1.png"/>

<img src="/assets/img/2024-10-27-ceph-storage-cluster/22.png"/>




### 5. Ceph admin dashboard - Grafana monitor dashboard 

- Ceph orch triển khai các daemon bằng podman container, kiểm tra địa chỉ và port các service
    
<img src="/assets/img/2024-10-27-ceph-storage-cluster/25.png"/>

- Ceph admin dashboard để quản lý Ceph cluster qua giao diện web 
   
<img src="/assets/img/2024-10-27-ceph-storage-cluster/23.png"/>

- Grafana monitor dashboard để giám sát cluster 
  
<img src="/assets/img/2024-10-27-ceph-storage-cluster/24.png"/>


### 6. Ceph block storage

#### 6.1. Ví dụ tạo Pool replicate 3, tạo Image size 100GB 

- Mặc định Ceph có sẵn 1 pool replicate 3 

<img src="/assets/img/2024-10-27-ceph-storage-cluster/26.png"/>

- Tạo RDB pool replicate 3 

```bash
ceph osd pool create production 
ceph osd pool application enable production rbd
rbd pool init -p production
```

<img src="/assets/img/2024-10-27-ceph-storage-cluster/27.png"/>

<img src="/assets/img/2024-10-27-ceph-storage-cluster/28.png"/>


- Tạo RDB image 100 GB cho client 

```bash
rbd create file-share --size 102400 --pool production
rbd ls production
rbd info production/file-share 
```

<img src="/assets/img/2024-10-27-ceph-storage-cluster/29.png"/>
<img src="/assets/img/2024-10-27-ceph-storage-cluster/30.png"/>
<img src="/assets/img/2024-10-27-ceph-storage-cluster/30.1.png"/>



#### 6.2. Sử dụng Ceph block storage trên Ubuntu server

- Trên server client cần install ceph-common
```bash
### Install ceph-common on Client
apt -y install ceph-common
```
- Trên server client tạo file /etc/ceph/ceph.conf (nội dung file như trên ceph admin node)

```bash
root@ubuntu-server:~# cat /etc/ceph/ceph.conf
# minimal ceph.conf for cc6b42ba-943b-11ef-9a15-b1833c256ed8
[global]
	fsid = cc6b42ba-943b-11ef-9a15-b1833c256ed8
	mon_host = [v2:10.10.1.11:3300/0,v1:10.10.1.11:6789/0] [v2:10.10.1.12:3300/0,v1:10.10.1.12:6789/0] [v2:10.10.1.13:3300/0,v1:10.10.1.13:6789/0]

root@ubuntu-server:~#
```

- Trên server client tạo file /etc/ceph/ceph.keyring (nội dung file như trên ceph admin node)
  
```bash
root@ubuntu-server:~# cat /etc/ceph/ceph.keyring 
[client.admin]
	key = AQBdPx5n0agJOxAACDY6rZbpJxFU5Ig7NjdZtw==

root@ubuntu-server:~# 
```


- Map block device trên server client

```bash
root@ubuntu-server:~# ceph osd lspools
1 .mgr
4 production
root@ubuntu-server:~# rbd ls -l production
NAME        SIZE     PARENT  FMT  PROT  LOCK
file-share  100 GiB            2            
root@ubuntu-server:~# 

# map the block device
root@ubuntu-server:~# rbd map file-share -p production
/dev/rbd0

# confirm
root@ubuntu-server:~# rbd showmapped
id  pool        namespace  image       snap  device   
0   production             file-share  -     /dev/rbd0
root@ubuntu-server:~#

# format with ext4
root@ubuntu-server:~# mkfs.ext4 /dev/rbd0

# mount folder 
mkdir -p /opt/ceph-data
mount /dev/rbd0 /opt/ceph-data



# umount 
root@ubuntu-server:~# umount /opt/ceph-data/

# unmap
root@ubuntu-server:~# rbd unmap /dev/rbd0

# delete image 
root@ubuntu-server:~# rbd rm file-share -p production

# delete a pool
# ceph osd pool delete [Pool Name] [Pool Name] ***
root@ubuntu-server:~# ceph osd pool delete production production --yes-i-really-really-mean-it


```

<img src="/assets/img/2024-10-27-ceph-storage-cluster/31.png"/>


#### 6.3. Sử dụng Ceph block storage trên Windows server  

> https://docs.ceph.com/en/reef/install/windows-install/

- Download Ceph cho Windows : https://cloudbase.it/ceph-for-windows/

- Cài đặt và restart server 

<img src="/assets/img/2024-10-27-ceph-storage-cluster/32.png"/>

<img src="/assets/img/2024-10-27-ceph-storage-cluster/33.png"/>


- Tạo file cấu hình Ceph : c:\%ProgramData%\Ceph\ceph.conf

```powershell
[global]
    log to stderr = true
    run dir = C:/ProgramData/Ceph/out
    crash dir = C:/ProgramData/Ceph/out

[client]
    keyring = C:/ProgramData/Ceph/ceph.keyring
    log file = C:/ProgramData/Ceph/out/$name.$pid.log
    admin socket = C:/ProgramData/Ceph/out/$name.$pid.asok

[global]
    mon host = 10.10.1.11,10.10.1.12,10.10.1.13
```

- Tạo file keyring : c:\%ProgramData%\Ceph\ceph.keyring

```powershell
[client.admin]
	key = AQBdPx5n0agJOxAACDY6rZbpJxFU5Ig7NjdZtw==
```

<img src="/assets/img/2024-10-27-ceph-storage-cluster/34.png"/>

- Dùng Windows PowerShell map Ceph block device 

```powershell
 rbd.exe map -n client.admin  production/file-share
 ```
<img src="/assets/img/2024-10-27-ceph-storage-cluster/35.png"/>

- Dùng Disk Manager trên Windows mount và sử dụng 

<img src="/assets/img/2024-10-27-ceph-storage-cluster/36.png"/>



> Nếu bạn thấy bài viết bổ ích , vui lòng like và share tại : [trang Facebook này](https://www.facebook.com/profile.php?id=61567456205328&mibextid=ZbWKwL)
, nó giúp cho tôi có thêm niềm vui khi chia sẻ các bài viết. Xin cảm ơn. 

