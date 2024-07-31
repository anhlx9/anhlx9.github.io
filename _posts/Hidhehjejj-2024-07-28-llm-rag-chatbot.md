---
layout: post
title: LLM RAG Chatbot với LLaMA 3 model 
subtitle: Chia sẻ về cách tạo chat bot AI sử dụng LLM và bổ sung thông tin trả lời qua các file tài liệu 
tags: [AI, LLM, System, Linux]
author: Anh Le
comments: false
mathjax: false
---

# Phát triển ứng dụng Chatbot AI với LLM Meta LLaMA 3

- [Phát triển ứng dụng Chatbot AI với LLM Meta LLaMA 3](#phát-triển-ứng-dụng-chatbot-ai-với-llm-meta-llama-3)
    - [1. Cài đặt Docker và Docker-compose trên server Ubuntu 22.04](#1-cài-đặt-docker-và-docker-compose-trên-server-ubuntu-2204)
        - [2.1. Thông tin server](#21-thông-tin-server)
        - [2.2. Cài đặt Docker service](#22-cài-đặt-docker-service)
        - [2.3. Cài đặt Docker-compose](#23-cài-đặt-docker-compose)
    - [3. Sử dụng Ollama để tương tác với LLM](#3-sử-dụng-ollama-để-tương-tác-với-llm)
        - [3.1. Ollama là gì?](#31-ollama-là-gì)
        - [3.2. LLaMA 3 là gì?](#32-llama-3-là-gì)
        - [3.3. Chạy Ollama với Docker-compose](#33-chạy-ollama-với-docker-compose)
        - [3.4. Sử dụng LLM Meta Llama 3 model (bản 4.7 GB) trên Ollama](#34-sử-dụng-llm-meta-llama-3-model-bản-47-gb-trên-ollama)
        - [3.5. Export và Import LLM model trên Ollama](#35-export-và-import-llm-model-trên-ollama)
    - [4. Các thành phần khi làm việc với LLM bằng Python code](#4-các-thành-phần-khi-làm-việc-với-llm-bằng-python-code)
        - [4.1. LangChain là gì?](#41-langchain-là-gì)
        - [4.2. ChromaDB là gì?](#42-chromadb-là-gì)
        - [4.3. Streamlit là gì?](#43-streamlit-là-gì)
    - [5. Tạo ứng dụng Chatbot LLM với Python code](#5-tạo-ứng-dụng-chatbot-llm-với-python-code)
        - [5.1. Cài đặt các gói cần thiết](#51-cài-đặt-các-gói-cần-thiết)




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



> [!TIP] dữ liệu được mount ra folder /opt/docker/ollama/ trên host để dễ quản lý

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

![Crepe](/assets/img/2024-07-28-llm-rag-chatbot/02.png)


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

> [!TIP] sử dụng "stream": false để response là 1 chuổi json.


![Crepe](/assets/img/2024-07-28-llm-rag-chatbot/05.png)

##### 3.5. Export và Import LLM model trên Ollama
- Do dung lượng LLM khá lớn, tôi sẽ export data ra disk để lưu trữ và import sử dụng cho các lần sau mà không cần download lại ,
  
- Export LLM :  
  - Tôi sử dụng script tại https://gist.github.com/supersonictw/f6cf5e599377132fe5e180b3d495c553
> [!TIP] chỉnh sửa OLLAMA_HOME phù hợp với đường dẫn trên máy của bạn. 
> 
> Sau khi chạy script sẽ export ra 3 file : 
>   - model.bin : data của LLM, với model llama3	tôi đang dùng dung lượng khoảng 4.7GB
>   - Modelfile : file dùng để tạo model trong Ollama
>   - source.txt : file meta

- Import LLM : 
  - Copy các file đã export trước đó vào path mount trong container, ở bài viết này là /opt/docker/ollama/ 
  - Vào Ollama container tạo model từ file export đã copy : 

```bash
root@Bot-Server:~# tree -sh /opt/docker/ollama/.ollama/
[4.0K]  /opt/docker/ollama/.ollama/
├── [ 387]  id_ed25519
├── [  81]  id_ed25519.pub
├── [4.0K]  llama3-model-backup
│   ├── [ 12K]  Modelfile
│   ├── [4.3G]  model.bin
│   └── [  40]  source.txt
└── [4.0K]  models
    └── [4.0K]  blobs

3 directories, 5 files
root@Bot-Server:~# 
root@Bot-Server:~# docker ps 
CONTAINER ID   IMAGE                  COMMAND               CREATED          STATUS          PORTS                                           NAMES
2c3a019f5573   ollama/ollama:latest   "/bin/ollama serve"   14 minutes ago   Up 14 minutes   0.0.0.0:11434->11434/tcp, :::11434->11434/tcp   Ollama
root@Bot-Server:~# 
root@Bot-Server:~# docker exec -it Ollama bash

root@2c3a019f5573:~# cd /root/.ollama/llama3-model-backup/
root@2c3a019f5573:~/.ollama/llama3-model-backup# ll
total 4552000
drwxrwxr-x 2 root root       4096 Jul 29 02:40 ./
drwxr-xr-x 4 root root       4096 Jul 29 02:41 ../
-rw-r--r-- 1 root root      12704 Jul 26 07:04 Modelfile
-rw-r--r-- 1 root root 4661211424 Jul 26 07:04 model.bin
-rw-r--r-- 1 root root         40 Jul 26 07:04 source.txt
root@2c3a019f5573:~/.ollama/llama3-model-backup# ollama create llama3:latest
transferring model data 
using existing layer sha256:6a0746a1ec1aef3e7ec53868f220ff6e389f6f8ef87a01d77c96807de94ca2aa 
creating new layer sha256:b73a2906c0224dcb54cece1820b54f1540411f8e81aca3f7c41271c908ccb311 
creating new layer sha256:8ab4849b038cf0abc5b1c9b8ee1443dca6b93a045c2272180d985126eb40bf6f 
creating new layer sha256:83af8bbae28affc119c2de6f7189ded5b6e8215f989b2ede9b506943db6dd82c 
writing manifest 
success 
root@2c3a019f5573:~/.ollama/llama3-model-backup# ollama list
NAME         	ID          	SIZE  	MODIFIED           
llama3:latest	dc1bbceb3d6d	4.7 GB	About a minute ago	
root@2c3a019f5573:~/.ollama/llama3-model-backup# 
```



### 4. Các thành phần khi làm việc với LLM bằng Python code 

##### 4.1. LangChain là gì?

LangChain là một khung mã nguồn mở để xây dựng các ứng dụng dựa trên các mô hình ngôn ngữ lớn (LLM). LLM là các mô hình học sâu lớn được đào tạo trước trên khối lượng lớn dữ liệu có thể tạo ra câu trả lời cho các câu hỏi của người dùng, ví dụ như trả lời câu hỏi hoặc tạo hình ảnh từ lời nhắc dựa trên văn bản. LangChain cung cấp các công cụ và yếu tố trừu tượng để cải thiện khả năng tùy chỉnh, độ chính xác và mức độ liên quan của thông tin do các mô hình tạo ra. Ví dụ: nhà phát triển có thể sử dụng các thành phần LangChain để xây dựng chuỗi nhắc mới hoặc tùy chỉnh các mẫu hiện có. LangChain cũng bao gồm các thành phần cho phép LLM truy cập các tập dữ liệu mới mà không cần đào tạo lại. 


##### 4.2. ChromaDB là gì?

ChromaDB: Là một cơ sơ dữ liệu vector mã nguồn mở dễ sử dụng và tích hợp sâu rộng với hệ sinh thái Python. 
Trong bài viết này tôi sử dụng LangChain trích xuất văn bản từ các tài liệu (.pdf, .txt, .docx, ..) thành các vector và lưu vào ChromaDB phục vụ cho việc tăng cường thông tin trả về cho người dùng .



##### 4.3. Streamlit là gì?

Streamlit là một framework web được viết bằng Python cho phép người dùng dễ dàng tạo các ứng dụng web tương tác trực quan với chỉ một vài dòng mã. Với Streamlit, người dùng chỉ cần tạo một tệp mã Python và sử dụng các thành phần có sẵn của Streamlit để xây dựng giao diện người dùng và xử lý dữ liệu. Streamlit cung cấp nhiều thành phần sẵn có để giúp người dùng tạo ra các ứng dụng web có thể tương tác với dữ liệu thời gian thực, bao gồm cả hình ảnh và video.

Trong trường hợp này, tôi không có chuyên môn sâu trong lĩnh vực lập trình, do đó Streamlit hoàn toàn phù hợp để tôi demo một Web UI cho ứng dụng Chatbot chúng ta đang nói đến.


### 5. Tạo ứng dụng Chatbot LLM với Python code

##### 5.1. Cài đặt các gói cần thiết 

- Tạo folder cho ứng dụng Chatbot
```bash 
mkdir -p /opt/chatbot/docs
touch /opt/chatbot/requirements.txt
touch /opt/chatbot/app.py

root@Bot-Server:~# tree /opt/chatbot/
/opt/chatbot/
├── app.py
├── docs
└── requirements.txt

1 directory, 2 files
root@Bot-Server:~# 

```

- Cài đặt package cần thiết : 
```bash

root@Bot-Server:~# apt install -y python3 python3-pip

root@Bot-Server:~# cat /opt/chatbot/requirements.txt

langchain
langchain-community
langchain-core
langchain-ollama
langchain-chroma
sentence-transformers
pypdf
docx2txt
streamlit

root@Bot-Server:~# 


root@Bot-Server:~# pip install -r /opt/chatbot/requirements.txt

```


- Code app.py 
```python

root@Bot-Server:/opt/chatbot# cat app.py 
#!/bin/env python3

# /opt/chatbot/app.py


import os
import sys
from dotenv import load_dotenv
from langchain.chains import ConversationalRetrievalChain
from langchain.text_splitter import CharacterTextSplitter
from langchain_community.document_loaders import PyPDFLoader
from langchain_community.document_loaders import Docx2txtLoader
from langchain_community.document_loaders import TextLoader
from langchain_community.embeddings.sentence_transformer import SentenceTransformerEmbeddings
from langchain_community.vectorstores import Chroma

import streamlit as st
from streamlit_chat import message

documents = []
# Create a List of Documents from all of our files in the ./docs folder
for file in os.listdir("docs"):
    if file.endswith(".pdf"):
        pdf_path = "./docs/" + file
        loader = PyPDFLoader(pdf_path)
        documents.extend(loader.load())
    elif file.endswith('.docx') or file.endswith('.doc'):
        doc_path = "./docs/" + file
        loader = Docx2txtLoader(doc_path)
        documents.extend(loader.load())
    elif file.endswith('.txt'):
        text_path = "./docs/" + file
        loader = TextLoader(text_path)
        documents.extend(loader.load())

# Split the documents into smaller chunks
text_splitter = CharacterTextSplitter(chunk_size=1000, chunk_overlap=10)
documents = text_splitter.split_documents(documents)

# create the open-source embedding function
embedding_function = SentenceTransformerEmbeddings(model_name="all-MiniLM-L6-v2")

# Convert the document chunks to embedding and save them to the vector store
vectordb = Chroma.from_documents(documents, embedding_function, persist_directory="./chroma_db")
vectordb.persist()

# query it
# query = "Who is Juan Garcia"
query = "hướng dẫn cài ubuntu"

# load from disk
db3 = Chroma(persist_directory="./chroma_db", embedding_function=embedding_function)
docs = db3.similarity_search(query)
print(docs[0].page_content)


root@Bot-Server:/opt/chatbot# 

```












































```bash
root@Bot-Server:~# cat /opt/chatbot/docs/langchain_chroma.txt
Basic Example
In this basic example, we take the most recent State of the Union Address, split it into chunks, embed it using an open-source embedding model, load it into Chroma, and then query it.

# import
from langchain_chroma import Chroma
from langchain_community.document_loaders import TextLoader
from langchain_community.embeddings.sentence_transformer import (
    SentenceTransformerEmbeddings,
)
from langchain_text_splitters import CharacterTextSplitter

# load the document and split it into chunks
loader = TextLoader("../../how_to/state_of_the_union.txt")
documents = loader.load()

# split it into chunks
text_splitter = CharacterTextSplitter(chunk_size=1000, chunk_overlap=0)
docs = text_splitter.split_documents(documents)

# create the open-source embedding function
embedding_function = SentenceTransformerEmbeddings(model_name="all-MiniLM-L6-v2")

# load it into Chroma
db = Chroma.from_documents(docs, embedding_function)

# query it
query = "What did the president say about Ketanji Brown Jackson"
docs = db.similarity_search(query)

# print results
print(docs[0].page_content)

root@Bot-Server:~# 
```
















