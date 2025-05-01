---
title: Guacamole Jump Host 
categories:
- System
- Linux
  
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Quản lý truy cập với Guacamole jump host trên nền web
---


### 1. Giới thiệu 

- [1. Giới thiệu](#1-giới-thiệu)
- [2. Triển khai Apache Guacamole jump host](#2-triển-khai-apache-guacamole-jump-host)
  - [2.1. Chuẩn bị](#21-chuẩn-bị)
  - [2.2. Cài đặt MariaDB](#22-cài-đặt-mariadb)
  - [2.3. Build the Guacamole Server from source](#23-build-the-guacamole-server-from-source)
  - [2.4. Cấu hình Guacamole Database Authentication](#24-cấu-hình-guacamole-database-authentication)
  - [2.5. Cài đặt recording storage extension](#25-cài-đặt-recording-storage-extension)
  - [2.6. Cài đặt Guacamole Web Application](#26-cài-đặt-guacamole-web-application)
  - [2.7. SSH Linux Server , RDP Windows Server qua Web](#27-ssh-linux-server--rdp-windows-server-qua-web)
  - [2.8. Xem lại record lịch sử truy cập của user](#28-xem-lại-record-lịch-sử-truy-cập-của-user)
- [4. Lời kết](#4-lời-kết)


Thông thường trong các hệ thống doanh nghiệp, chúng ta triển khai 1 vài server chạy Windows server hoặc Linux server làm jump host, từ đó SSH hoặc RDP tới các server đằng sau, nhằm quản lý truy cập vào hệ thống để đảm bảo security. Với cách triển khai như vậy, chúng ta khó hoặc không thể ghi nhận các thao tác của user đối với hệ thống từ jump host, đặc biệt trên môi trường đồ họa như Windows Desktop hoặc Ubuntu Desktop không thể ghi nhận thao tác click chuột của user. 

Trong bài viết này, mình chia sẻ về cách triển khai Apache Guacamole làm jump host với các ưu điểm : 

- <b>Thao tác trên nền Web.</b>
- <b>Quản lý user.</b>
- <b>Phân quyền truy cập cho user.</b>
- <b>Record session remote của user và xem lại lịch sử truy cập.</b>

<b>Mô hình triển khai</b>

<img src="/assets/img/2025-05-01-guacamole-jump-host/01.png"/>

<b>HA design</b>

<img src="/assets/img/2025-05-01-guacamole-jump-host/02.png"/>


- RDP Windows server trên nền web

<video src="/assets/img/2025-05-01-guacamole-jump-host/rdp-windows.mp4" width="320" height="240" controls></video>

- SSH Linux server trên nền web

<video src="/assets/img/2025-05-01-guacamole-jump-host/ssh-linux.mp4" width="320" height="240" controls></video>

- Xem lại record session 

<video src="/assets/img/2025-05-01-guacamole-jump-host/record-playback.mp4" width="320" height="240" controls></video>



### 2. Triển khai Apache Guacamole jump host

#### 2.1. Chuẩn bị 

- Trong bài viết này, mình sẽ sử dụng Ubuntu server 22.04 cài Guacamole jump host, SSH đến Rhel 9 Server và RDP đến Windows Server 2022

<img src="/assets/img/2025-05-01-guacamole-jump-host/03.png"/>

- Set hostname jump host 

```bash
hostnamectl set-hostname jump101
```
- Cài đặt các gói cần thiết 

```bash
apt update

apt install -y gcc nano vim curl wget g++ libcairo2-dev libjpeg-turbo8-dev libpng-dev libtool-bin libossp-uuid-dev

apt install -y libavcodec-dev libavformat-dev libavutil-dev libswscale-dev build-essential libpango1.0-dev libssh2-1-dev libvncserver-dev libtelnet-dev libpulse-dev libvorbis-dev libwebp-dev

# Install FreeRDP2
add-apt-repository ppa:remmina-ppa-team/remmina-next-daily -y
apt update
apt install freerdp2-dev freerdp2-x11 -y
```

- Cài đặt Java 

```bash
apt install default-jdk -y

java --version

root@jump101:/root/# java --version
openjdk 11.0.26 2025-01-21
OpenJDK Runtime Environment (build 11.0.26+4-post-Ubuntu-1ubuntu122.04)
OpenJDK 64-Bit Server VM (build 11.0.26+4-post-Ubuntu-1ubuntu122.04, mixed mode, sharing)
root@jump101:/root/#
```

- Cài đặt Apache Tomcat

```bash
apt install tomcat9 tomcat9-admin tomcat9-common tomcat9-user -y 

systemctl enable --now tomcat9
systemctl status tomcat9
```

<img src="/assets/img/2025-05-01-guacamole-jump-host/04.png"/>

#### 2.2. Cài đặt MariaDB 

- Cài đặt mariadb-server

```bash
apt -y install mariadb-server
```

- Cấu hình cơ bản MariaDB 

```bash
tee /etc/mysql/my.cnf <<EOF
[server]
[mysqld]
pid-file                = /run/mysqld/mysqld.pid
basedir                 = /usr
bind-address            = 0.0.0.0
port = 3306
# default max_connections value 151 is not enough on Openstack Env
max_connections = 500
expire_logs_days        = 10
character-set-server  = utf8mb4
collation-server      = utf8mb4_general_ci
[embedded]
[mariadb]
[mariadb-10.6]
[client-server]
socket = /run/mysqld/mysqld.sock
EOF
```


- Đặt mật khẩu root DB

```bash
mysql_secure_installation

Change the root password? [Y/n] 
New password: 123456


systemctl restart mariadb
systemctl enable mariadb
journalctl -xeu mariadb.service

root@jump101:/root/# mysql -u root -p123456 -e "SELECT host, user FROM mysql.user  ;"
+-----------+-------------+
| Host      | User        |
+-----------+-------------+
| localhost | mariadb.sys |
| localhost | mysql       |
| localhost | root        |
+-----------+-------------+
root@jump101:/root/#


root@jump101:/root/# netstat -nlpt | grep mariadb
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      12129/mariadbd
root@jump101:/root/#
```

<img src="/assets/img/2025-05-01-guacamole-jump-host/05.png"/>

#### 2.3. Build the Guacamole Server from source

- Download và build Guacamole Server

```bash
# download Guacamole Server
cd ~/
VER=1.5.4
wget https://archive.apache.org/dist/guacamole/$VER/source/guacamole-server-$VER.tar.gz
tar xzf ~/guacamole-server-*.tar.gz

# build Guacamole Server
cd ~/guacamole-server-*/
./configure  --with-init-dir=/etc/init.d
```

<img src="/assets/img/2025-05-01-guacamole-jump-host/06.png"/>

```bash
make 
make install

ldconfig
mkdir  -p /etc/guacamole/{extensions,lib}
tree  /etc/guacamole/
```

<img src="/assets/img/2025-05-01-guacamole-jump-host/07.png"/>


- Tạo guacd.conf

```bash  
tee /etc/guacamole/guacd.conf <<EOF
[daemon]
pid_file = /var/run/guacd.pid
#log_level = debug

[server]
#bind_host = localhost
bind_host = 127.0.0.1
bind_port = 4822

#[ssl]
#server_certificate = /etc/ssl/certs/guacd.crt
#server_key = /etc/ssl/private/guacd.key
EOF
```

- Khởi động dịch vụ guacd
```bash
systemctl daemon-reload
systemctl restart guacd
systemctl enable guacd
systemctl status guacd
```

<img src="/assets/img/2025-05-01-guacamole-jump-host/08.png"/>

#### 2.4. Cấu hình Guacamole Database Authentication

- Cài đặt lib và extention kết nối database

```bash
# MySQL Connector/J (Java Connector)
cd ~/
CON_VER=8.3.0
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-$CON_VER.tar.gz
tar -xf mysql-connector-j-$CON_VER.tar.gz
cp mysql-connector-j-$CON_VER/mysql-connector-j-$CON_VER.jar /etc/guacamole/lib/

# JDBC auth plugin for Guacamole
cd ~/
VER=1.5.4
wget https://downloads.apache.org/guacamole/$VER/binary/guacamole-auth-jdbc-$VER.tar.gz
tar -xf guacamole-auth-jdbc-$VER.tar.gz
mv guacamole-auth-jdbc-$VER/mysql/guacamole-auth-jdbc-mysql-$VER.jar /etc/guacamole/extensions/
```

<img src="/assets/img/2025-05-01-guacamole-jump-host/09.png"/>

- Tạo database cho Guacamole server 

```bash
mysql -u root -p123456

CREATE DATABASE guacamole_db;
CREATE USER 'guacamole_user'@'localhost' IDENTIFIED BY 'Passw0rd!';
GRANT SELECT,INSERT,UPDATE,DELETE ON guacamole_db.* TO 'guacamole_user'@'localhost';
FLUSH PRIVILEGES;
QUIT
```

- Import schema cho Guacamole

```bash
cd guacamole-auth-jdbc-*/mysql/schema
cat *.sql | mysql -u root -pPassw0rd! guacamole_db
```

<img src="/assets/img/2025-05-01-guacamole-jump-host/10.png"/>


#### 2.5. Cài đặt recording storage extension

- Cài đặt extention lưu file record 

```bash
cd ~
VER=1.5.4
wget https://downloads.apache.org/guacamole/$VER/binary/guacamole-history-recording-storage-$VER.tar.gz
tar -xf guacamole-history-recording-storage-$VER.tar.gz

mv guacamole-history-recording-storage-1.5.4/guacamole-history-recording-storage-1.5.4.jar /etc/guacamole/extensions/
```

<img src="/assets/img/2025-05-01-guacamole-jump-host/11.png"/>

- Thư mục default record

```bash
mkdir -p /var/lib/guacamole/recordings
chown tomcat:tomcat /var/lib/guacamole/recordings
chmod 2750 /var/lib/guacamole/recordings
```

<img src="/assets/img/2025-05-01-guacamole-jump-host/12.png"/>

#### 2.6. Cài đặt Guacamole Web Application

- Download và cài Guacamole Client

```bash
# Install Guacamole Client
cd ~
VER=1.5.4
wget https://archive.apache.org/dist/guacamole/$VER/binary/guacamole-$VER.war
mv guacamole-$VER.war /var/lib/tomcat9/webapps/guacamole.war

# config Guacamole client connect to the Guacamole server (guacd)
echo "GUACAMOLE_HOME=/etc/guacamole" | tee -a /etc/default/tomcat
echo "export GUACAMOLE_HOME=/etc/guacamole" | tee -a /etc/profile

tee /etc/guacamole/guacamole.properties <<EOF
guacd-hostname: localhost
guacd-port:     4822

### MySQL properties
mysql-hostname: 127.0.0.1
mysql-port: 3306
mysql-database: guacamole_db
mysql-username: guacamole_user
mysql-password: Passw0rd!

EOF


systemctl restart tomcat9 guacd
```

<img src="/assets/img/2025-05-01-guacamole-jump-host/13.png"/>

#### 2.7. SSH Linux Server , RDP Windows Server qua Web

- Truy cập Guacamole Web 
> http://172.16.10.101:8080/guacamole

> admin default : guacadmin\guacadmin

<img src="/assets/img/2025-05-01-guacamole-jump-host/14.png"/>

- Tạo connection SSH Linux server

```bash
Recording path:	${HISTORY_PATH}/${HISTORY_UUID}/
Recording name:	session-${GUAC_USERNAME}-${GUAC_DATE}-${GUAC_TIME}
```

<img src="/assets/img/2025-05-01-guacamole-jump-host/15.png"/>

<img src="/assets/img/2025-05-01-guacamole-jump-host/16.png"/>

- Tạo connection RDP Windows server  

<img src="/assets/img/2025-05-01-guacamole-jump-host/17.png"/>

<img src="/assets/img/2025-05-01-guacamole-jump-host/18.png"/>


- Tạo và phân quyền user01 chỉ được phép remote Linux server 102

<img src="/assets/img/2025-05-01-guacamole-jump-host/19.png"/>

- Tạo và phân quyền user02 chỉ được phép remote Windows server 103

<img src="/assets/img/2025-05-01-guacamole-jump-host/20.png"/>

- Kiểm tra truy cập , user01 chỉ thấy connection Linux server 102

<img src="/assets/img/2025-05-01-guacamole-jump-host/21.png"/>

<img src="/assets/img/2025-05-01-guacamole-jump-host/22.png"/>

- Kiểm tra truy cập , user02 chỉ thấy connection Windows server 103

<img src="/assets/img/2025-05-01-guacamole-jump-host/23.png"/>

<img src="/assets/img/2025-05-01-guacamole-jump-host/24.png"/>

#### 2.8. Xem lại record lịch sử truy cập của user 

- Các account có quyền admin sẽ xem lại được lịch sử truy cập 

<img src="/assets/img/2025-05-01-guacamole-jump-host/25.png"/>

<img src="/assets/img/2025-05-01-guacamole-jump-host/26.png"/>

- Các file record được lưu trên server, để play được cần convert qua định dạng .m4v 

```bash
# convert to .m4v
guacenc -s 1280x720 -r 20000000 -f /var/lib/guacamole/recordings/1efef3b7-049e-390c-be74-effe86e1ab83/session-anhlx-20250501-031132
```

<img src="/assets/img/2025-05-01-guacamole-jump-host/27.png"/>

<img src="/assets/img/2025-05-01-guacamole-jump-host/28.png"/>


 ### 4. Lời kết 

- Bài viết này mình đã chia sẻ về cách triển khai Apache Guacamole jump host mã nguồn mở , miễn phí, được sử dụng rộng rãi, phù hợp với doanh nghiệp. Cho phép quản lý truy cập và record các thao tác trên hệ thống nhằm đảm bảo security và trích suất lịch sử, evidence khi có sự cố xảy ra.
- Trên đây chỉ là bài demo nên mình dùng VM và triển khai cơ bản môhình stand alone. Chúc bạn thành công. 
  