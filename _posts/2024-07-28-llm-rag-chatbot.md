---
layout: post
title: LLM RAG Chatbot
subtitle: Chia sẻ về cách tạo chat bot AI sử dụng LLM và bổ sung thông tin trả lời qua các file tài liệu 
tags: [AI, LLM, System, Linux]
author: Anh Le
comments: false
mathjax: false
---

# Phát triển ứng dụng Chatbot với LLM 

- [Phát triển ứng dụng Chatbot với LLM](#phát-triển-ứng-dụng-chatbot-với-llm)
    - [1. Thông tin server](#1-thông-tin-server)
    - [2. Cài đặt Docker \& Docker-compose trên server Ubuntu 22.04](#2-cài-đặt-docker--docker-compose-trên-server-ubuntu-2204)
        - [2.1. Cài đặt Docker service](#21-cài-đặt-docker-service)
        - [2.2. Cài đặt Docker-compose](#22-cài-đặt-docker-compose)



### 1. Thông tin server
- Tôi thực hiện trên máy ảo với cấu hình :
  - CPU : 16 cores
  - Ram : 32 GB
  - Disk : 100 GB
  - IP : 192.168.161.11/24 
  
![Crepe](/assets/img/2024-07-28-llm-rag-chatbot/01.png)


### 2. Cài đặt Docker & Docker-compose trên server Ubuntu 22.04 

- Trong bài viết này tôi triển khai các thành phần trên container qua docker-compose nhằm dễ quản lí cấu hình.

##### 2.1. Cài đặt Docker service

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt install docker.io -y 
systemctl restart docker.service
systemctl enable docker.service
systemctl status docker.service
docker ps -a 
```

##### 2.2. Cài đặt Docker-compose
```bash
curl -L "https://github.com/docker/compose/releases/download/v2.29.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose
chmod +x  /usr/bin/docker-compose 
docker-compose --version
```
<br>


<!-- 
```bash
pip install langchain
``` -->