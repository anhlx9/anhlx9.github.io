---
title: RAG Chatbot với DeepSeek-R1 LLM
categories:
- System
- LLM
- DeepSeek
  
feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Chạy mô hình ngôn ngữ lớn DeepSeek-R1 cục bộ và tạo RAG chatbot tích hợp vào Website
---


### 1. Giới thiệu 

- [1. Giới thiệu](#1-giới-thiệu)
    - [1.1. LLM là gì?](#11-llm-là-gì)
    - [1.2. DeepSeek-R1 là gì?](#12-deepseek-r1-là-gì)
    - [1.3. RAG là gì?](#13-rag-là-gì)
    - [1.4. Ollama là gì?](#14-ollama-là-gì)
    - [1.5. AnythingLLM là gì?](#15-anythingllm-là-gì)
- [2. Cài đặt Docker và Docker-compose trên server Ubuntu 22.04](#2-cài-đặt-docker-và-docker-compose-trên-server-ubuntu-2204)
    - [2.1. Cài đặt Docker service](#21-cài-đặt-docker-service)
    - [2.2. Cài đặt Docker-compose](#22-cài-đặt-docker-compose)
- [3. Sử dụng Ollama để tương tác với LLM](#3-sử-dụng-ollama-để-tương-tác-với-llm)
    - [3.1. Chạy Ollama với Docker-compose](#31-chạy-ollama-với-docker-compose)
    - [3.2. Sử dụng DeepSeek-R1:8b model trên Ollama](#32-sử-dụng-deepseek-r18b-model-trên-ollama)
    - [3.3. Tạo RAG chatbot tích hợp vào trang Web](#33-tạo-rag-chatbot-tích-hợp-vào-trang-web)
- [4. Lời kết](#4-lời-kết)


Dạo này gia đình tôi có con nhỏ, từ lúc có con bé thời gian sinh hoạt của cả nhà bị xáo trộn cả. Đêm nay cũng vậy, dậy lúc nửa đêm mãi chẳng vào giấc lại được, sẵn đọc báo thấy nói nhiều về DeepSeek nên tôi lại lọ mọ tải về dùng thử xem hay ho thế nào mà dân tình rộn ràng thế . 

Tôi sẽ tạo một Chatbot AI sử dụng kỹ thuật truy suất thông tin tăng cường Retrieval-Augmented Generation (RAG) và tích hợp cửa sổ chat với DeepSeek vào Website. 

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/15.png"/>

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/06.gif"/>


Trong bài viết này, tôi dùng VM chạy Ubuntu server 22.04 với cấu hình như sau

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/01.png"/>


Chatbot AI là một ứng dụng trí tuệ nhân tạo được thiết kế nhằm thực hiện các cuộc trò chuyện với con người. Dựa trên các thuật toán và xử lý ngôn ngữ tự nhiên (NLP), Chatbot AI có khả năng phản hồi câu hỏi, giải quyết vấn đề, thực hiện các tác vụ đơn giản, và cung cấp tư vấn về sản phẩm/dịch vụ, đóng vai trò quan trọng trong việc cung cấp thông tin và hỗ trợ khách hàng. 

Cùng với sự tiến bộ của công nghệ, Chatbot AI ngày càng được cải tiến với khả năng hiểu và đáp ứng các yêu cầu của người dùng, tạo ra trải nghiệm tương tác thông minh và thuận tiện.

<b>Ưu điểm : </b>
- Tiết kiệm chi phí: Chatbot AI có khả năng thấu hiểu đa ngôn ngữ, vì vậy bạn không cần phải bỏ ra quá nhiều chi phí nhân sự. Đồng thời, với khả năng làm việc không ngừng nghỉ, người trợ lý này có thể túc trực 24/7 nhằm giải đáp câu hỏi, cung cấp lời khuyên và hỗ trợ khách hàng trong quá trình mua sắm hoặc sử dụng sản phẩm/dịch vụ ở bất cứ khi nào và ở bất cứ nơi đâu.
- Tăng khả năng phản hồi: Như đã đề cập, Chatbot AI được lập trình tự động và hoạt động 24/7, vì vậy bạn không cần phải lo về khả năng bỏ lỡ tin nhắn. Người bạn này sẽ hỗ trợ bạn trong việc phản hồi khách hàng, giúp họ an tâm khi mua sắm hoặc sử dụng sản phẩm/dịch vụ của doanh nghiệp. 


##### 1.1. LLM là gì?

Các mô hình ngôn ngữ lớn (LLM - Large language model) là một loại mô hình ngôn ngữ được đào tạo bằng cách sử dụng các kỹ thuật học sâu trên tập dữ liệu văn bản khổng lồ. Các mô hình này có khả năng tạo văn bản tương tự như con người và thực hiện các tác vụ xử lý ngôn ngữ tự nhiên khác nhau.

Một mô hình ngôn ngữ có thể có độ phức tạp khác nhau, từ các mô hình n-gram đơn giản đến các mô hình mạng mô phỏng hệ thần kinh của con người vô cùng phức tạp. Tuy nhiên, thuật ngữ Large language model” thường dùng để chỉ các mô hình sử dụng kỹ thuật học sâu và có số lượng tham số lớn, có thể từ hàng tỷ đến hàng nghìn tỷ. Những mô hình này có thể phát hiện các quy luật phức tạp trong ngôn ngữ và tạo ra các văn bản y hệt con người.


##### 1.2. DeepSeek-R1 là gì?

DeepSeek-R1 là một mô hình ngôn ngữ lớn (LLM) được phát triển bởi Trung Quốc vừa ra mắt, đang rất nổi tiếng, được giới thiệu là 1 model được xây dựng và phát triển với chi phí cực thấp, mã nguồn mở và hiệu suất vượt trội.


##### 1.3. RAG là gì?

Retrieval Augmented Generation (RAG) là 1 kỹ thuật truy suất thông tin tăng cường qua các nguồn tài liệu (website, .pdf, .docx, .txt, ...) nhằm trả về kết quả cho người dùng những thông tin chính xác hoặc đặc thù riêng về 1 sản phẩm, 1 doanh nghiệp, hay 1 sự vật nào đó mà thông tin trong LLM chưa đủ hoặc chưa chính xác.

Mục đích chính của RAG là nâng cao khả năng của mô hình ngôn ngữ lớn (LLM), đặc biệt là trong các nhiệm vụ đòi hỏi sự hiểu biết sâu sắc và tạo ra câu trả lời phù hợp với ngữ cảnh.

##### 1.4. Ollama là gì?

🦙 Ollama là một công cụ ngôn ngữ mạnh mẽ cho phép bạn chạy các mô hình ngôn ngữ lớn trên máy tính cá nhân của mình một cách dễ dàng. Hiện tại, Ollama hỗ trợ các hệ điều hành Mac OS và Linux, và phiên bản Windows đang được phát triển. Với Ollama, bạn có thể chạy các mô hình ngôn ngữ như DeepSeek-R1, LLaMA-3, Mistral, Vicuna và nhiều mô hình khác.

##### 1.5. AnythingLLM là gì?

Anything LLM là 1 công cụ mạnh mẽ cho phép bạn trò chuyện với các mô hình ngôn ngữ lớn (LLM)). Được thiết kế để mang lại trải nghiệm tốt nhất cho người dùng, Anything LLM cung cấp khả năng tùy chỉnh, tích hợp vector database và hỗ trợ nhiều người dùng.

Phiên bản mới nhất của Anything LLM đã được cải tiến đáng kể với hỗ trợ các tính năng mạnh mẽ và giao diện trực quan hơn. Phiên bản này có khả năng tự cập nhật, hỗ trợ môi trường Doanh nghiệp và cho phép sử dụng chatbot riêng tư. Điều này giúp người dùng tận hưởng trải nghiệm tốt nhất và linh hoạt trong việc sử dụng Anything LLM.

### 2. Cài đặt Docker và Docker-compose trên server Ubuntu 22.04 

- Trong bài viết này tôi triển khai các thành phần trên container qua docker-compose nhằm dễ quản lí cấu hình.


##### 2.1. Cài đặt Docker service

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository -y "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt install docker.io -y 
systemctl enable docker.service
systemctl status docker.service
docker ps -a 
```

##### 2.2. Cài đặt Docker-compose
```bash
curl -L "https://github.com/docker/compose/releases/download/v2.32.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose
chmod +x  /usr/bin/docker-compose 
docker-compose --version
```

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/02.png"/>

### 3. Sử dụng Ollama để tương tác với LLM 

##### 3.1. Chạy Ollama với Docker-compose

- Tạo thư mục dự án : 
```bash
mkdir -p /opt/docker/
touch /opt/docker/docker-compose.yml 
```

- docker-compose cho Ollama :

> Note : Dữ liệu được mount ra folder /opt/docker/ollama/ trên host để dễ quản lý

```yml
# /opt/docker/docker-compose.yml
services:
  ollama:
    container_name: ollama
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
docker-compose -f /opt/docker/docker-compose.yml up -d --force-recreate ollama
```

- Ollama đã chạy và API hoạt động tại port 11434 

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/03.png"/>


##### 3.2. Sử dụng DeepSeek-R1:8b model trên Ollama

- Ollama supports a list of models available on [ollama.com/library](https://ollama.com/library 'ollama model library')

> Note:  You should have at least 8 GB of RAM available to run the 7B models, 16 GB to run the 13B models, and 32 GB to run the 33B models.

- Vào Ollama container tải xuống model deepseek-r1:8b
  
```bash
root@RAG:~# docker exec -it ollama bash
root@452c134dc4ae:/# ollama pull deepseek-r1:8b
```

- Ollama bắt đầu tải xuống LLM DeepSeek-R1 bản 8 tỷ tham số với dung lượng 4.9GB.

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/04.png"/>

- Hoàn tất tải xuống LLM DeepSeek-R1:8b
  
<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/05.png"/>

- Sau khi Ollama hoàn tất tải xuống LLM DeepSeek-R1, chúng ta có thể bắt đầu sử dụng LLM qua CLI.
- 
```bash
root@452c134dc4ae:/# ollama run deepseek-r1:8b
```
<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/06.png"/>

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/06.gif"/>


- Thao tác với Ollama qua API : 

```bash
root@RAG:~# curl http://localhost:11434/api/generate -d '{"model": "deepseek-r1:8b" , "prompt":"Thủ đô Trung Quốc ? trả lời bằng tiếng Việt." , "stream": false}' | jq
```

> Note : sử dụng "stream": false để response là 1 chuỗi json.

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/07.png"/>


##### 3.3. Tạo RAG chatbot tích hợp vào trang Web

- Ở đây mình chỉ demo việc tạo 1 mã java script embed cửa sổ chat với DeepSeek-R1 vào 1 trang html.
  
- Mình sẽ start container Nginx làm Web server và AnythingLLM để tương tác với model DeepSeek-R1. Đồng thời AnythingLLM có chức năng thêm các dữ liệu bổ sung từ các nguồn khác như Web, PDF, TXT,... để tạo ra RAG chatbot.

- docker-compose cho nginx  :

```yml
# /opt/docker/docker-compose.yml
services:
  nginx:
    container_name: nginx
    image: nginx:alpine
    restart: always
    ports:
      - "8080:80"
    volumes:
      - /opt/docker/nginx/html:/usr/share/nginx/html:z

```

- docker-compose cho AnythingLLM  :

```yml
# /opt/docker/docker-compose.yml
services:
  anythingllm:
    image: mintplexlabs/anythingllm
    container_name: anythingllm
    restart: always
    cap_add:
      - SYS_ADMIN
    ports:
    - "3001:3001"
    environment:
    # Adjust for your environment
      - STORAGE_DIR=/app/server/storage
      - JWT_SECRET="rWURbsxox849ZD"
      - LLM_PROVIDER=ollama
      - OLLAMA_BASE_PATH=http://10.50.10.21:11434
      - OLLAMA_MODEL_PREF=deepseek-r1:8b
      - OLLAMA_MODEL_TOKEN_LIMIT=4096
      - EMBEDDING_ENGINE=ollama
      - EMBEDDING_BASE_PATH=http://10.50.10.21:11434
      - EMBEDDING_MODEL_PREF=mxbai-embed-large:latest
      - EMBEDDING_MODEL_MAX_CHUNK_LENGTH=8192
      - VECTOR_DB=lancedb
      - WHISPER_PROVIDER=local
      - TTS_PROVIDER=native
      - PASSWORDMINCHAR=8
      - AGENT_SERPER_DEV_KEY="123456"
      - AGENT_SERPLY_API_KEY="123456789"
    volumes:
      - anythingllm_storage:/app/server/storage
 
volumes:
  anythingllm_storage:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /opt/docker/anythingllm/storage
```

> Note:
> - LLM_PROVIDER: sử dụng ollama
> - OLLAMA_BASE_PATH: endpoint ollama để tạo sinh văn bản trả lời người dùng 
> - OLLAMA_MODEL_PREF: model trên ollama để tạo sinh văn bản trả lời người dùng , ở đây mình sử dụng deepseek-r1:8b


> Vector embedding là một phương thức chuyển đổi các dạng dữ liệu thành số nhằm bóc tách ngữ nghĩa và quan hệ của chúng. Chúng biểu diễn các dữ liệu thành các điểm trong không gian đa chiều, các điểm gần nhau hơn sẽ giống nhau về ngữ nghĩa hơn.
> 
> - EMBEDDING_BASE_PATH: endpoint ollama để embedding
> - EMBEDDING_MODEL_PREF: model trên ollama để embedding, ở đây sử dụng model mxbai-embed-large

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/08.png"/>

- Trong AnythingLLM mình dùng model mxbai-embed-large để embedding nên mình cần vào ollama download thêm model mxbai-embed-large.

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/09.png"/>


- Truy cập WebUi của AnythingLLM kiểm tra tương tác với model Deepseek-R1.

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/10.png"/>


- Mình sẽ tăng cường thông tin trả lời cho chatbot từ 1 file giới thiệu bản thân  

> https://anhle.com.vn/

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/11.png"/>

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/12.png"/>

- Lấy code embed
  
<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/13.png"/>

- Mình sẽ tích hợp cửa sổ chat với DeepSeek-R1 vào 1 trang index.html , ví dụ ở đây là trang index của Nginx.

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/14.png"/>

- Dữ liệu của Deepseek được huấn luyện đến năm 2023

<img src="/assets/img/2025-01-31-rag-chatbot-with-deepseek-llm/15.png"/>


- Nội dung file index.html 

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Welcome to nginx!</title>
  <style>
      body {
          width: 35em;
          margin: 0 auto;
          font-family: Tahoma, Verdana, Arial, sans-serif;
          background-color:#95afc0;
      }
  </style>
  </head>
  <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>

    <script
      data-chat-icon="support"
      data-assistant-name="Deepseek-R1 Chatbot"
      data-greeting="Need help?"
      data-no-sponsor="true"
      data-assistant-icon="https://repository-images.githubusercontent.com/920179950/55571654-d64c-4d4f-aee7-a516a5d9949e"
      data-brand-image-url="https://repository-images.githubusercontent.com/920179950/55571654-d64c-4d4f-aee7-a516a5d9949e"
      data-embed-id="e2d1b2a4-9707-47ce-8fbd-b0080515d062"
      data-base-api-url="http://10.50.10.21:3001/api/embed"
      src="http://10.50.10.21:3001/embed/anythingllm-chat-widget.min.js"
    ></script>

  </body>
</html>

```

 ### 4. Lời kết 

- Bài viết này mình đã chia sẻ về cách chạy mô hình ngôn ngữ lớn cục bộ và tăng cường thông tin trả lời qua nguồn dữ liệu bổ sung từ các nguồn website, file pdf, txt, md ...
- Trên đây chỉ là bài nghiên cứu và demo nên mình dùng VM không sử dụng GPU nên tốc độ xử lý rất chậm, tuy nhiên nếu bạn thử nghiệm chạy trên các hệ thống chuyên dụng cho tác vụ AI với GPU sẽ gặt hái được những thành tụ kinh ngạc. Chúc bạn thành công. 
  