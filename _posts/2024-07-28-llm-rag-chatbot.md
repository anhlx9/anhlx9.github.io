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
    - [1. Cài đặt Docker và Docker-compose trên server Ubuntu 22.04](#1-cài-đặt-docker-và-docker-compose-trên-server-ubuntu-2204)
        - [2.1. Thông tin server](#21-thông-tin-server)
        - [2.2. Cài đặt Docker service](#22-cài-đặt-docker-service)
        - [2.3. Cài đặt Docker-compose](#23-cài-đặt-docker-compose)
    - [3. Sử dụng Ollama để tương tác với LLM](#3-sử-dụng-ollama-để-tương-tác-với-llm)
        - [3.1. Ollama là gì?](#31-ollama-là-gì)
        - [3.2. LLaMA 3 là gì?](#32-llama-3-là-gì)
        - [3.3. Chạy Ollama với Docker-compose](#33-chạy-ollama-với-docker-compose)
        - [3.4. Sử dụng LLM Meta Llama 3 model (bản 4.7 GB) trên Ollama](#34-sử-dụng-llm-meta-llama-3-model-bản-47-gb-trên-ollama)
    - [4. Làm việc với LangChain](#4-làm-việc-với-langchain)
        - [4.1. LangChain là gì?](#41-langchain-là-gì)




### 1. Cài đặt Docker và Docker-compose trên server Ubuntu 22.04 

- Trong bài viết này tôi triển khai các thành phần trên container qua docker-compose nhằm dễ quản lí cấu hình.

##### 2.1. Thông tin server
- Tôi thực hiện bài viết này trên máy ảo VMWare với cấu hình :
  - CPU : 16 cores
  - Ram : 32 GB
  - Disk : 100 GB
  - IP : 192.168.161.11/24 
  
![Crepe](/assets/img/2024-07-28-llm-rag-chatbot/01.png)


##### 2.2. Cài đặt Docker service

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt install docker.io -y 
systemctl restart docker.service
systemctl enable docker.service
systemctl status docker.service
docker ps -a 
```

##### 2.3. Cài đặt Docker-compose
```bash
curl -L "https://github.com/docker/compose/releases/download/v2.29.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose
chmod +x  /usr/bin/docker-compose 
docker-compose --version
```
<br>


### 3. Sử dụng Ollama để tương tác với LLM 

##### 3.1. Ollama là gì?

🦙 Ollama là một công cụ ngôn ngữ mạnh mẽ cho phép bạn chạy các mô hình ngôn ngữ lớn trên máy tính cá nhân của mình một cách dễ dàng. Hiện tại, Ollama hỗ trợ các hệ điều hành Mac OS và Linux, và phiên bản Windows đang được phát triển. Với Ollama, bạn có thể chạy các mô hình ngôn ngữ như LLaMA-2, Mistral, Vicuna và nhiều mô hình khác.

##### 3.2. LLaMA 3 là gì?

LLaMA 3 là một mô hình ngôn ngữ lớn (LLM) được phát triển bởi Meta AI, nổi tiếng với khả năng tạo văn bản, dịch ngôn ngữ và viết các loại nội dung sáng tạo khác nhau.


##### 3.3. Chạy Ollama với Docker-compose

- Tạo thư mục dự án : 
```bash
mkdir -p /opt/docker/
touch /opt/docker/docker-compose.yml 
```

- docker-compose cho Ollama :

> [!TIP]

> dữ liệu được mount ra folder /opt/docker/ollama/ trên host để dễ quản lý

```yml
# /opt/docker/docker-compose.yml
services:
  Ollama:
    container_name: Ollama
    image: ollama/ollama:latest
    restart: always
    ports:
      - 11434:11434
    volumes:
      - /opt/docker/ollama/.ollama:/root/.ollama
    environment:
      - OLLAMA_KEEP_ALIVE=24h
      - OLLAMA_HOST=0.0.0.0
```
- Khởi chạy Ollama container :

```bash
docker-compose -f /opt/docker/docker-compose.yml up -d --force-recreate Ollama
```

- Ollama đã chạy và API hoạt động tại 0.0.0.0:11434 

![Crepe](/assets/img/2024-07-28-llm-rag-chatbot/01.png)


##### 3.4. Sử dụng LLM Meta Llama 3 model (bản 4.7 GB) trên Ollama


- Ollama supports a list of models available on [ollama.com/library](https://ollama.com/library 'ollama model library')

- Here are some example models that can be downloaded:

| Model              | Parameters | Size  | Download                       |
| ------------------ | ---------- | ----- | ------------------------------ |
| llama3           | 8B         | 4.7GB | `ollama run llama3`            |
| Llama 3            | 70B        | 40GB  | `ollama run llama3:70b`        |
| Phi 3 Mini         | 3.8B       | 2.3GB | `ollama run phi3`              |
| Phi 3 Medium       | 14B        | 7.9GB | `ollama run phi3:medium`       |
| Gemma 2            | 9B         | 5.5GB | `ollama run gemma2`            |
| Gemma 2            | 27B        | 16GB  | `ollama run gemma2:27b`        |
| Mistral            | 7B         | 4.1GB | `ollama run mistral`           |
| Moondream 2        | 1.4B       | 829MB | `ollama run moondream`         |
| Neural Chat        | 7B         | 4.1GB | `ollama run neural-chat`       |
| Starling           | 7B         | 4.1GB | `ollama run starling-lm`       |
| Code Llama         | 7B         | 3.8GB | `ollama run codellama`         |
| Llama 2 Uncensored | 7B         | 3.8GB | `ollama run llama2-uncensored` |
| LLaVA              | 7B         | 4.5GB | `ollama run llava`             |
| Solar              | 10.7B      | 6.1GB | `ollama run solar`             |

> [!NOTE]
> You should have at least 8 GB of RAM available to run the 7B models, 16 GB to run the 13B models, and 32 GB to run the 33B models.


- Vào Ollama container chạy lệnh : 'ollama run llama3'
```bash
root@Bot-Server:~# docker exec -it Ollama bash
root@49590f2f6645:/# ollama run llama3
```

- Ollama bắt đầu tải xuống LLM llama3 dung lượng 4.7GB : 
  
![Crepe](/assets/img/2024-07-28-llm-rag-chatbot/03.png)


- Sau khi Ollama hoàn tất tải xuống LLM llama3, chúng ta có thể bắt đầu sử dụng LLM qua CLI : 

![Crepe](/assets/img/2024-07-28-llm-rag-chatbot/04.png)


- Thao tác với Ollama qua API : 

```
curl http://localhost:11434/api/generate -d '{"model": "llama3" , "prompt":"giới thiệu ngắn gọn tiểu sử Nguyễn Du" , "stream": false}' | jq
```
> [!TIP]

> sử dụng "stream": false để response là 1 chuổi json.

> Trong ví dụ này, thời gian trả về dữ liệu là hơn 1 phút, khá lâu :( 

![Crepe](/assets/img/2024-07-28-llm-rag-chatbot/05.png)



### 4. Làm việc với LangChain 

##### 4.1. LangChain là gì?

LangChain là một khung mã nguồn mở để xây dựng các ứng dụng dựa trên các mô hình ngôn ngữ lớn (LLM). LLM là các mô hình học sâu lớn được đào tạo trước trên khối lượng lớn dữ liệu có thể tạo ra câu trả lời cho các câu hỏi của người dùng, ví dụ như trả lời câu hỏi hoặc tạo hình ảnh từ lời nhắc dựa trên văn bản. LangChain cung cấp các công cụ và yếu tố trừu tượng để cải thiện khả năng tùy chỉnh, độ chính xác và mức độ liên quan của thông tin do các mô hình tạo ra. Ví dụ: nhà phát triển có thể sử dụng các thành phần LangChain để xây dựng chuỗi nhắc mới hoặc tùy chỉnh các mẫu hiện có. LangChain cũng bao gồm các thành phần cho phép LLM truy cập các tập dữ liệu mới mà không cần đào tạo lại. 


