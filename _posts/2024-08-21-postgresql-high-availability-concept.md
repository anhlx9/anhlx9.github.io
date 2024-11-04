---
title: PostgreSQL - HA Concept
categories:
- Database
- System
- Linux
feature_image: "/assets/post-banner.jpg"
feature_text: |
  ### PostgreSQL - High Availability Database Concept
---

### 1. Giới thiệu 

- Trong các bài viết khác, mình sử dụng PostgreSQL làm database do đó để giảm bớt nội dung kiến thức trong các bài viết, mình sẽ tách riêng bài viết xây dựng hệ thống database với PostgreSQL có tính sẵn sàng cao để khi tích hợp các hệ thống với nhau chúng ta có 1 mô hình lớn tổng thể các dịch vụ có độ ổn định và tính sẵn sàng cao phục vụ cho hoạt động của doanh nghiệp.

- PostgreSQL là 1 hệ cơ sở dữ liệu nguồn mở có tính ổn định cao. Các hướng dẫn cài đặt, quản trị,... đã có khá nhiều bài viết trên Internet. Giới hạn trong bài viết này, mình chỉ chia sẻ về cách mình triển khai 1 hệ cơ sở dữ liệu sử dụng PostgresSQL có khả năng chịu lỗi và rất đơn giản trong việc triển khai. Nếu cần đáp ứng CCU lớn, concept này hoàn toàn có thể được scale lớn hơn để phục vụ.

<img src="/assets/img/2024-08-21-postgresql-high-availability-concept/postgresql-failover.gif"/>


 
- [1. Giới thiệu](#1-giới-thiệu)
- [2. Các công nghệ sử dụng](#2-các-công-nghệ-sử-dụng)
- [3. Mô hình triển khai PostgreSQL Master-Slave](#3-mô-hình-triển-khai-postgresql-master-slave)
- [4. Các bước cài đặt và cấu hình](#4-các-bước-cài-đặt-và-cấu-hình)
  - [4.1. Cài đặt Docker service và Docker-compose](#41-cài-đặt-docker-service-và-docker-compose)
  - [4.2. Cấu hình PostgreSQL Replicate Master/Slave sử dụng Docker-compose](#42-cấu-hình-postgresql-replicate-masterslave-sử-dụng-docker-compose)
  - [4.3. Cài đặt và cấu hình Keepalived tự động chuyển đổi dự phòng cho PostgreSQL Replicate Master/Slave](#43-cài-đặt-và-cấu-hình-keepalived-tự-động-chuyển-đổi-dự-phòng-cho-postgresql-replicate-masterslave)
 
### 2. Các công nghệ sử dụng
- Server Ubuntu 22.04 
- Docker service , docker-compose
- PostgreSQL 
- Keepalived 
- Shell script

Như danh sách trên có thể thấy, chúng ta sẽ kết hợp nhiều thành phần với các vai trò khác nhau để tạo thành 1 solution cần thiết. Mình sử dụng OS Ubuntu server 22.04 LTS vì hiện tại đây là version thông dụng và stable, các bản Ubuntu 23 và 24 hiện tại ý kiến cá nhân mình thấy vẫn còn chưa rộng rãi các package và chưa stable. Trong bài viết này mình sẽ triển khai trên Docker, nếu cần chịu tải cao hơn các bạn có thể triển khai với systemd.


### 3. Mô hình triển khai PostgreSQL Master-Slave 

- Chúng ta sử dụng Keepalived để tạo Virtual IP (VIP) cung cấp endpoint cho kết nối database, cơ chế Mater-Slave hay Active-Standby giúp giảm thiểu tình trạng confict dữ liệu khi write đồng thời. 

- Khi server hoặc instance PostgreSQL gặp sự cố  (cụ thể trong trường hợp này là server Down), sau khoảng 5 giây Keepalived sẽ tự động switch VIP từ Master sang Slave đồng thời mình sử dụng option trong Keepalived là "notify" để gọi chạy 1 file Shell script thực hiện các công việc check STATE của Keepalive, start PostgreSQL container với cấu hình Master và trigger Slave lên làm Master. Khi server Up được ghi nhận trở thành Slave cũng sẽ dùng option "notify" chạy Shell script thực hiện start PostgreSQL container với cấu hình là Slave và đồng bộ data từ Master.

- Như giải thích bên trên, Shell script được mình sử dụng làm kịch bản control 2 server khi có sự cố, đảm bảo dịch vụ database luôn Up giảm tối đa Downtime. Ở trường hợp dịch vụ của mình vận hành khá stable khi sử dụng thực tế, mình không cần phải xử lý khi server gặp sự cố cứ để nó tự chuyển đổi dự phòng. 

### 4. Các bước cài đặt và cấu hình 

#### 4.1. Cài đặt Docker service và Docker-compose

Note:
> Thực hiện trên cả 2 server 

- Cài đặt Docker service
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository -y "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt install docker.io -y 
systemctl restart docker.service
systemctl enable docker.service
systemctl status docker.service
docker ps -a 
```

- Cài đặt Docker-compose
```bash
curl -L "https://github.com/docker/compose/releases/download/v2.29.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose
chmod +x  /usr/bin/docker-compose 
docker-compose --version
```

<img src="/assets/img/2024-08-21-postgresql-high-availability-concept/01.png"/>



#### 4.2. Cấu hình PostgreSQL Replicate Master/Slave sử dụng Docker-compose


Note:
> Thực hiện trên cả 2 server. 

> Mỗi server đều có 2 file yaml để start container PostgresSQL Master và Slave. 


- Tạo thư mục để mount PostgreSQL data trong container ra host. 

```bash
mkdir -p /opt/docker/postgresql/data
chown -R 1001.1001 /opt/docker/postgresql/

touch /opt/docker/postgresql/docker-compose-postgresql-master.yaml
touch /opt/docker/postgresql/docker-compose-postgresql-slave.yaml

```
<br>

- Nội dung file docker-compose-postgresql-master.yaml trên server PostgreSQL-01

```yml
# docker-compose-postgresql-master.yaml on PostgreSQL-01

services:

  postgresql:
    image: bitnami/postgresql:latest
    #image: bitnami/postgresql:12.4.0
    container_name: postgresql
    hostname: "postgresql"
    restart: always
    environment:

      ##### start container as Master
      - POSTGRESQL_REPLICATION_MODE=master
      - POSTGRESQL_REPLICATION_USER=repl_user
      - POSTGRESQL_REPLICATION_PASSWORD=replpwd
      - POSTGRESQL_USERNAME=gitlab
      - POSTGRESQL_PASSWORD=7JG9z9L*e*jy5
      - POSTGRESQL_DATABASE=gitlab_production
      - POSTGRESQL_POSTGRES_PASSWORD=5U2vE*18O52JVnpB

    volumes:
      - "/opt/docker/postgresql/data:/bitnami/postgresql/data"
    ports:
      - 5432:5432
    extra_hosts:
           - postgresql-01:192.168.161.11
           - postgresql-02:192.168.161.12
```

<br>

- Nội dung file docker-compose-postgresql-slave.yaml trên server PostgreSQL-01

```yml
# docker-compose-postgresql-slave.yaml on PostgreSQL-01

services:

  postgresql:
    image: bitnami/postgresql:latest
    #image: bitnami/postgresql:12.4.0
    container_name: postgresql
    hostname: "postgresql"
    restart: always
    environment:

      ##### start container as Slave
      - POSTGRESQL_REPLICATION_MODE=slave
      - POSTGRESQL_REPLICATION_USER=repl_user
      - POSTGRESQL_REPLICATION_PASSWORD=replpwd
      - POSTGRESQL_MASTER_HOST=postgresql-02
      - POSTGRESQL_MASTER_PORT_NUMBER=5432
      - POSTGRESQL_PASSWORD=5U2vE*18O52JVnpB

    volumes:
      - "/opt/docker/postgresql/data:/bitnami/postgresql/data"
    ports:
      - 5432:5432
    extra_hosts:
           - postgresql-01:192.168.161.11
           - postgresql-02:192.168.161.12
```

<br>

- Nội dung file docker-compose-postgresql-master.yaml trên server PostgreSQL-02

```yml
# docker-compose-postgresql-master.yaml on PostgreSQL-02

services:

  postgresql:
    image: bitnami/postgresql:latest
    #image: bitnami/postgresql:12.4.0
    container_name: postgresql
    hostname: "postgresql"
    restart: always
    environment:

      ##### start container as Master
      - POSTGRESQL_REPLICATION_MODE=master
      - POSTGRESQL_REPLICATION_USER=repl_user
      - POSTGRESQL_REPLICATION_PASSWORD=replpwd
      - POSTGRESQL_USERNAME=gitlab
      - POSTGRESQL_PASSWORD=7JG9z9L*e*jy5
      - POSTGRESQL_DATABASE=gitlab_production
      - POSTGRESQL_POSTGRES_PASSWORD=5U2vE*18O52JVnpB

    volumes:
      - "/opt/docker/postgresql/data:/bitnami/postgresql/data"
    ports:
      - 5432:5432
    extra_hosts:
           - postgresql-01:192.168.161.11
           - postgresql-02:192.168.161.12

```

<br>

- Nội dung file docker-compose-postgresql-slave.yaml trên server PostgreSQL-02

```yml
# docker-compose-postgresql-slave.yaml on PostgreSQL-02

services:

  postgresql:
    image: bitnami/postgresql:latest
    #image: bitnami/postgresql:12.4.0
    container_name: postgresql
    hostname: "postgresql"
    restart: always
    environment:

      ##### start container as Slave
      - POSTGRESQL_REPLICATION_MODE=slave
      - POSTGRESQL_REPLICATION_USER=repl_user
      - POSTGRESQL_REPLICATION_PASSWORD=replpwd
      - POSTGRESQL_MASTER_HOST=postgresql-01
      - POSTGRESQL_MASTER_PORT_NUMBER=5432
      - POSTGRESQL_PASSWORD=5U2vE*18O52JVnpB

    volumes:
      - "/opt/docker/postgresql/data:/bitnami/postgresql/data"
    ports:
      - 5432:5432
    extra_hosts:
           - postgresql-01:192.168.161.11
           - postgresql-02:192.168.161.12

```
<br>

<img src="/assets/img/2024-08-21-postgresql-high-availability-concept/02.png"/>
<br>
<img src="/assets/img/2024-08-21-postgresql-high-availability-concept/03.png"/>


- Khởi chạy container PostgreSQL làm Master trên server PostgreSQL-01

```bash
docker-compose -f /opt/docker/postgresql/docker-compose-postgresql-master.yaml up -d --force-recreate postgresql
```


- Khởi chạy container PostgreSQL làm Slave trên server PostgreSQL-02
  
```bash
docker-compose -f /opt/docker/postgresql/docker-compose-postgresql-slave.yaml up -d --force-recreate postgresql
```

<img src="/assets/img/2024-08-21-postgresql-high-availability-concept/04.png"/>
<br>
<img src="/assets/img/2024-08-21-postgresql-high-availability-concept/05.png"/>




- Trên server PostgreSQL-01 làm Master kiểm tra replicate : 

```bash
psql -U postgres -p 5432

postgres=# select usename,application_name,client_addr,backend_start,state,sync_state from pg_stat_replication ;
```

<img src="/assets/img/2024-08-21-postgresql-high-availability-concept/06.png"/>



- Trên server PostgreSQL-02 làm Slave kiểm tra database đã được đồng bộ : 

<img src="/assets/img/2024-08-21-postgresql-high-availability-concept/07.png"/>



#### 4.3. Cài đặt và cấu hình Keepalived tự động chuyển đổi dự phòng cho PostgreSQL Replicate Master/Slave 

Note:
> Thực hiện trên cả 2 server. 

```bash
apt install keepalived -y 
```

- Tạo file keepalived_control_failover.sh trên server PostgreSQL-01

```bash
#!/bin/bash

# /etc/keepalived/keepalived_control_failover.sh
# keepalived_control_failover.sh On server PostgreSQL-01


STATE=$3
peer='192.168.161.12'
ipaddr=`/sbin/ifconfig ens33 | awk -F ' *|:' '/inet /{print $3}'`
logs='/etc/keepalived/keepalived_control_failover.log'
echo "$(date +'%Y-%m-%d %H:%M:%S') $(hostname) $(hostname -I), STATE: ${STATE} " >>  $logs

sendTelegram(){
        curl -s -X POST --data chat_id=-xxxxx --data text="$1" "https://api.telegram.org/botxxxxxxx:Axxxxx/sendMessage" 
}


case $STATE in
        "MASTER") echo "$(date +'%Y-%m-%d %H:%M:%S') trigger postgresql container to master" >>  $logs
                  /usr/bin/docker exec postgresql touch /tmp/postgresql.trigger.5432
                  echo "$(date +'%Y-%m-%d %H:%M:%S') recreate postgresql container with Master config" >>  $logs
                  /usr/bin/docker-compose -f /opt/docker/postgresql/docker-compose-postgresql-master.yaml up -d --force-recreate postgresql
                  sendTelegram "❗ PostgreSQL Failover Switching ❗ %0A alert on $(hostname) %0A - Hostname : $(hostname) %0A - STATE : ${STATE} %0A - IP : ${ipaddr} "
                  exit 0
                  ;;

        "BACKUP") echo "$(date +'%Y-%m-%d %H:%M:%S') stop postgresql container" >>  $logs
                  /usr/bin/docker stop postgresql
                  echo "$(date +'%Y-%m-%d %H:%M:%S') rsync data from Master " >>  $logs
                  echo 'xxxxx' | rsync -av root@$peer:/opt/docker/postgresql/data/ /opt/docker/postgresql/data/
                  echo "$(date +'%Y-%m-%d %H:%M:%S') recreate postgresql container with Slave config" >>  $logs
                  /usr/bin/docker-compose -f /opt/docker/postgresql/docker-compose-postgresql-slave.yaml up -d --force-recreate postgresql
                  sendTelegram "❗ PostgreSQL Failover Switching ❗ %0A alert on $(hostname) %0A - Hostname : $(hostname) %0A - STATE : ${STATE} %0A - IP : ${ipaddr} "
                  exit 0
                  ;;

        *)        echo "unknown state"
                  echo "$(date +'%Y-%m-%d %H:%M:%S') unknown state to process!!!" >>  $logs
                  exit 0
                  ;;
esac
```

- Tạo file keepalived_control_failover.sh trên server PostgreSQL-02

```bash
#!/bin/bash

# /etc/keepalived/keepalived_control_failover.sh
# keepalived_control_failover.sh On server PostgreSQL-02


STATE=$3
peer='192.168.161.11'
ipaddr=`/sbin/ifconfig ens33 | awk -F ' *|:' '/inet /{print $3}'`
logs='/etc/keepalived/keepalived_control_failover.log'
echo "$(date +'%Y-%m-%d %H:%M:%S') $(hostname) $(hostname -I), STATE: ${STATE} " >>  $logs

sendTelegram(){
        curl -s -X POST --data chat_id=-xxxxx --data text="$1" "https://api.telegram.org/botxxxxxxx:Axxxxx/sendMessage" 
}


case $STATE in
        "MASTER") echo "$(date +'%Y-%m-%d %H:%M:%S') trigger postgresql container to master" >>  $logs
                  /usr/bin/docker exec postgresql touch /tmp/postgresql.trigger.5432
                  echo "$(date +'%Y-%m-%d %H:%M:%S') recreate postgresql container with Master config" >>  $logs
                  /usr/bin/docker-compose -f /opt/docker/postgresql/docker-compose-postgresql-master.yaml up -d --force-recreate postgresql
                  sendTelegram "❗ PostgreSQL Failover Switching ❗ %0A alert on $(hostname) %0A - Hostname : $(hostname) %0A - STATE : ${STATE} %0A - IP : ${ipaddr} "
                  exit 0
                  ;;

        "BACKUP") echo "$(date +'%Y-%m-%d %H:%M:%S') stop postgresql container" >>  $logs
                  /usr/bin/docker stop postgresql
                  echo "$(date +'%Y-%m-%d %H:%M:%S') rsync data from Master " >>  $logs
                  echo 'xxxxx' | rsync -av root@$peer:/opt/docker/postgresql/data/ /opt/docker/postgresql/data/
                  echo "$(date +'%Y-%m-%d %H:%M:%S') recreate postgresql container with Slave config" >>  $logs
                  /usr/bin/docker-compose -f /opt/docker/postgresql/docker-compose-postgresql-slave.yaml up -d --force-recreate postgresql
                  sendTelegram "❗ PostgreSQL Failover Switching ❗ %0A alert on $(hostname) %0A - Hostname : $(hostname) %0A - STATE : ${STATE} %0A - IP : ${ipaddr} "
                  exit 0
                  ;;

        *)        echo "unknown state"
                  echo "$(date +'%Y-%m-%d %H:%M:%S') unknown state to process!!!" >>  $logs
                  exit 0
                  ;;
esac
```

- Tạo file cấu hình Keepalived trên server PostgreSQL-01

```bash
# /etc/keepalived/keepalived.conf 
global_defs {
  enable_script_security
  script_user root 
}

vrrp_script chk_postgresql {
    script "/usr/bin/nc -zv localhost 5432"
    interval 2
    weight 3
}

vrrp_instance VIP_1 {
    interface ens33
    state MASTER
    virtual_router_id 60
    priority 100
    authentication {
        auth_type PASS
        auth_pass 3Bj1KiCoLBYbmUxy
    }
    virtual_ipaddress {
        192.168.161.10/24
    }
    track_script {
        chk_postgresql
    }

    notify "/etc/keepalived/keepalived_control_failover.sh"
}
```

- Tạo file cấu hình Keepalived trên server PostgreSQL-02

```bash
# /etc/keepalived/keepalived.conf 

global_defs {
  enable_script_security
  script_user root 
}

vrrp_script chk_postgresql {
    script "/usr/bin/nc -zv localhost 5432"
    interval 2
    weight 3
}

vrrp_instance VIP_1 {
    interface ens33
    state BACKUP
    virtual_router_id 60
    priority 100
    authentication {
        auth_type PASS
        auth_pass 3Bj1KiCoLBYbmUxy
    }
    virtual_ipaddress {
        192.168.161.10/24
    }
    track_script {
        chk_postgresql
    }

    notify "/etc/keepalived/keepalived_control_failover.sh"
}
```

- Cấp quyền thực thi cho script và start Keepalived service 

```bash
chmod +x /etc/keepalived/keepalived_control_failover.sh

systemctl enable keepalived.service
systemctl restart keepalived.service

journalctl -u keepalived | tail -n 100
```


- Thông báo khi hệ thống tự động chuyển đổi dự phòng qua Telegram : 

<img src="/assets/img/2024-08-21-postgresql-high-availability-concept/08.png"/>


- Bạn hãy Down/Up server để test failover và theo dõi quá trình tự động chuyển đổi dự phòng cho PostgreSQL Database.

<img src="/assets/img/2024-08-21-postgresql-high-availability-concept/09.png"/>


> Nếu bạn thấy bài viết bổ ích , vui lòng like và share tại : [trang Facebook này](https://www.facebook.com/profile.php?id=61567456205328&mibextid=ZbWKwL)
, nó giúp cho tôi có thêm niềm vui khi chia sẻ các bài viết. Xin cảm ơn. 


