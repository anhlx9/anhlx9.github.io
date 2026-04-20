---
title: "AI Platform cho System Engineer - Ollama + Dify + n8n trên Docker"
categories:
- System
- LLM
- Automation

feature_image: "/assets/postbanner.jpg"
feature_text: |
  ### Xây dựng nền tảng AI Assistant tự host cho System Engineer với Ollama, Dify và n8n
---

### Mục lục

- [Mục lục](#mục-lục)
- [1. Giới thiệu \& Ý tưởng Lab](#1-giới-thiệu--ý-tưởng-lab)
- [2. Kiến trúc tổng quan](#2-kiến-trúc-tổng-quan)
- [3. Chuẩn bị VM \& Cài đặt Ubuntu Server 24.04](#3-chuẩn-bị-vm--cài-đặt-ubuntu-server-2404)
- [4. Cài đặt Docker \& Docker Compose](#4-cài-đặt-docker--docker-compose)
- [5. Triển khai Ollama \& Custom model từ Hugging Face](#5-triển-khai-ollama--custom-model-từ-hugging-face)
- [6. Triển khai Dify - AI Application Platform](#6-triển-khai-dify---ai-application-platform)
- [7. Triển khai n8n - Workflow Automation](#7-triển-khai-n8n---workflow-automation)
- [8. Tích hợp Ollama vào Dify](#8-tích-hợp-ollama-vào-dify)
- [9. Tích hợp Ollama/Dify vào n8n](#9-tích-hợp-ollamadify-vào-n8n)
- [10. Use Case 1 - Chatbot hỗ trợ SysAdmin](#10-use-case-1---chatbot-hỗ-trợ-sysadmin)
- [11. Use Case 2 - Tự động phân tích log \& tạo ticket](#11-use-case-2---tự-động-phân-tích-log--tạo-ticket)
- [12. Use Case 3 - Sinh script Bash/Ansible tự động](#12-use-case-3---sinh-script-bashansible-tự-động)
- [13. Use Case 4 - Tạo tài liệu kỹ thuật tự động](#13-use-case-4---tạo-tài-liệu-kỹ-thuật-tự-động)
- [14. Tối ưu hiệu năng \& Monitoring](#14-tối-ưu-hiệu-năng--monitoring)
- [15. Kết luận](#15-kết-luận)

---

### 1. Giới thiệu & Ý tưởng Lab

Trong công việc hàng ngày, System Engineer thường phải xử lý nhiều tác vụ lặp đi lặp lại: viết script, phân tích log, troubleshoot sự cố, tạo tài liệu kỹ thuật, hỗ trợ NOC/SOC... Việc tích hợp AI vào quy trình làm việc giúp tăng năng suất đáng kể.

Bài lab này xây dựng một **AI Platform tự host hoàn toàn nội bộ** (không cần internet, không gửi dữ liệu ra ngoài) trên một VM duy nhất, bao gồm 3 thành phần chính:

| Thành phần | Vai trò | Port |
|---|---|---|
| **Ollama** | Chạy LLM model cục bộ (Text GGUF từ Hugging Face + Vision từ Ollama Library) | `11434` |
| **Dify** | Nền tảng xây dựng AI app, chatbot, agent, RAG pipeline | `80` |
| **n8n** | Nền tảng workflow automation, kết nối AI với hệ thống giám sát/ticketing | `5678` |

**Chiến lược Dual-Model: Text + Vision**

Thay vì dùng 1 model text cho mọi tác vụ, lab này triển khai **2 model chuyên biệt** trên cùng Ollama — 1 model text mạnh cho code/analysis và 1 model vision cho xử lý hình ảnh:

| Model | Base | Quantize | Size | RAM | Vai trò |
|---|---|---|---|---|---|
| **sysadmin-coder** | Qwen2.5-Coder:**14b** | Q5_K_M | ~10 GB | ~13 GB | **Text** - suy luận, thiết kế, debug hệ thống; sinh script, phân tích log, viết tài liệu |
| **sysadmin-vision** | Llama 3.2 Vision:**11b** | Q4 (Ollama) | ~7.9 GB | ~9 GB | **Vision** - xử lý hình ảnh từ screenshot, dashboard, diagram, OCR log |

> **Tại sao dual-model Text + Vision?**
> - `sysadmin-coder` (14b): chuyên **suy luận, thiết kế, debug hệ thống** — sinh script Bash/Ansible/Terraform, phân tích log sâu, chatbot Q&A kỹ thuật (~5-10 token/s trên CPU)
> - `sysadmin-vision` (11b): chuyên **xử lý hình ảnh** — nhận diện lỗi từ screenshot, đọc dashboard Grafana/Zabbix, đọc diagram topology, OCR log từ ảnh chụp
> - Với 32GB RAM: cả 2 model được load đồng thời vào RAM (`OLLAMA_MAX_LOADED_MODELS=2`) — không có swap delay khi chuyển giữa text và vision
> - Dify/n8n cho phép chọn model khác nhau cho mỗi app/workflow

**Phân công model theo Use Case:**

| Use Case | Model | Lý do |
|---|---|---|
| Chatbot SysAdmin Q&A | `sysadmin-coder` (14b) | Suy luận, trả lời kỹ thuật chính xác |
| Sinh script Bash/Ansible | `sysadmin-coder` (14b) | Thiết kế script, có error handling |
| Phân tích log text | `sysadmin-coder` (14b) | Debug sâu, context 128K |
| Phân loại alert & triage | `sysadmin-coder` (14b) | Suy luận nguyên nhân, output JSON |
| Tạo tài liệu kỹ thuật | `sysadmin-coder` (14b) | Output dài, cần chất lượng cao |
| Phân tích screenshot lỗi | `sysadmin-vision` (11b) | Nhìn ảnh Grafana/error → đề xuất fix |
| Đọc diagram/topology | `sysadmin-vision` (11b) | Upload sơ đồ mạng → AI mô tả kiến trúc |
| OCR log từ ảnh | `sysadmin-vision` (11b) | Chụp màn hình log → AI đọc và phân tích |
| Nhận diện phần cứng từ ảnh | `sysadmin-vision` (11b) | Chụp rack/switch → AI nhận diện thiết bị |

**Tại sao chọn 2 model này?**

| Tiêu chí | sysadmin-coder (Qwen2.5-Coder:14b) | sysadmin-vision (Llama 3.2 Vision:11b) |
|---|---|---|
| Suy luận, debug hệ thống | **Xuất sắc** - reasoning sâu | Không chuyên |
| Thiết kế & sinh script Bash/Ansible | **Xuất sắc** (HumanEval 92.9%) | Không chuyên |
| Phân tích log text | **Rất tốt** - context 128K | Không chuyên |
| Phân tích screenshot/dashboard | Không hỗ trợ | **Xuất sắc** - native multimodal Meta |
| Đọc diagram/topology | Không hỗ trợ | **Rất tốt** - reasoning vượt LLaVA |
| OCR log từ ảnh chụp | Không hỗ trợ | **Rất tốt** - đọc text từ ảnh chính xác |
| Tốc độ inference (CPU) | 5-10 tok/s | 5-8 tok/s |
| Context window | **128K** | **128K** |
| RAM cần thiết | ~13 GB (Q5_K_M) | ~9 GB (Q4 Ollama) |
| Ngôn ngữ đầu ra | Tiếng Việt ổn định | **Tiếng Việt ổn định** (Meta, đa ngôn ngữ) |
| Nguồn | GGUF bartowski (Hugging Face) | `ollama pull` (Ollama Library) |

> **Tip:** Nếu VM chỉ có 16GB RAM, chỉ dùng `sysadmin-coder` (14b) cho text — bỏ vision model

**Mục tiêu lab:**
- Chatbot SysAdmin hỏi đáp về Linux, Windows Server, networking, troubleshooting
- Tự động phân tích log từ syslog/Prometheus alert → đề xuất fix
- Sinh script Bash/Ansible/Terraform từ mô tả bằng ngôn ngữ tự nhiên
- Tạo tài liệu kỹ thuật (runbook, post-mortem) tự động
- Workflow tự động: Alert → AI phân tích → Tạo ticket → Gửi thông báo
- **Phân tích screenshot** Grafana/error page → AI nhìn ảnh và đề xuất giải pháp
- **Đọc diagram/topology** → AI mô tả kiến trúc mạng từ ảnh
- **OCR log từ ảnh** → Chụp màn hình log → AI đọc và phân tích

---

### 2. Kiến trúc tổng quan

```
┌─────────────────────────────────────────────────────────────────┐
│                   VM: aiops (Ubuntu 24.04)                      │
│              16 CPU │ 32 GB RAM │ 100 GB Disk                   │
│                    10.10.200.11                                 │
│                                                                 │
│  ┌─────────────┐   ┌──────────────────┐   ┌──────────────────┐  │
│  │   Ollama    │   │      Dify        │   │       n8n        │  │
│  │  :11434     │   │      :80         │   │      :5678       │  │
│  │             │   │                  │   │                  │  │
│  │ ┌─────────┐ │   │ ┌──────────────┐ │   │ ┌──────────────┐ │  │
│  │ │Text 14b │ │◄──┤ │ Script Gen   │ │   │ │ Alert Triage │ │  │
│  │ │ (Q5_K_M)│ │   │ │ Deep Analyze │ │   │ │ Chatbot Q&A  │ │  │
│  │ ├─────────┤ │   │ │ Tech Docs    │ │   │ ├──────────────┤ │  │
│  │ │Vision 11b││◄──┤ ├──────────────┤ │   │ │ Screenshot   │ │  │
│  │ │(Llama3.2)││   │ │ Chatbot Q&A  │ │   │ │ Analysis     │ │  │
│  │ └─────────┘ │   │ │ OCR Ảnh      │ │   │ └──────┬───────┘ │  │
│  └──────▲──────┘   └──────────────────┘   └────────┼─────────┘  │
│         │                                          │            │
│         └──────────────────────────────────────────┘            │
│                     Ollama API (:11434)                         │
│                                                                 │
│  ┌─────────┐  ┌───────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │ Docker  │  │ PostgreSQL│  │  Redis   │  │   Weaviate/      │ │
│  │ Engine  │  │ (Dify DB) │  │ (Cache)  │  │   Qdrant (RAG)   │ │
│  └─────────┘  └───────────┘  └──────────┘  └──────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Luồng hoạt động:**
1. **User → Dify**: Truy cập giao diện web Dify để chat, hỏi đáp với AI chatbot
2. **Dify → Ollama**: Dify gửi prompt/ảnh đến Ollama API — text query dùng `sysadmin-coder` (14b), ảnh dùng `sysadmin-vision` (11b)
3. **n8n → Ollama/Dify**: n8n nhận webhook từ hệ thống giám sát (Prometheus, Zabbix...), gọi AI phân tích, rồi tạo ticket/gửi thông báo
4. **Dify RAG**: Upload tài liệu kỹ thuật (runbook, KB) → Dify vector hóa và lưu vào vector database → AI trả lời dựa trên knowledge base nội bộ

---

### 3. Chuẩn bị VM & Cài đặt Ubuntu Server 24.04

**Thông số VM trên vSphere:**

| Thông số | Giá trị |
|---|---|
| VM Name | aiops |
| CPU | 16 vCPU |
| Memory | 32 GB |
| Hard Disk | 100 GB |
| Network | 10.10.200.11 |

> **Lưu ý:** 100 GB disk cần phân bổ hợp lý: ~25GB cho OS + Docker images, ~22GB cho model text (14b GGUF) + vision (11b Ollama), ~30GB cho Dify data/vector DB, ~23GB còn lại cho log/data.

**Cài đặt Ubuntu Server 24.04 LTS** với cấu hình cơ bản:

```bash
# Cập nhật hệ thống
sudo apt update && sudo apt upgrade -y

# Cài đặt các package cần thiết
sudo apt install -y curl wget git vi htop net-tools ca-certificates gnupg lsb-release

# Cấu hình hostname
sudo hostnamectl set-hostname aiops
```

**Cấu hình swap (khuyến nghị cho LLM inference):**

```bash
# Tạo swap 8GB (hỗ trợ khi model cần thêm RAM)
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Thêm vào fstab để tự mount khi reboot
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Giảm swappiness (ưu tiên dùng RAM)
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

### 4. Cài đặt Docker & Docker Compose

```bash
# Thêm Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Thêm Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Cài đặt Docker Engine + Docker Compose
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Thêm user hiện tại vào group docker
sudo usermod -aG docker $USER
newgrp docker

# Kiểm tra
docker --version
docker compose version
```

**Cấu hình Docker daemon (tối ưu cho production):**

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "default-address-pools": [
    {
      "base": "172.20.0.0/16",
      "size": 24
    }
  ]
}
EOF

sudo systemctl restart docker
```

---

### 5. Triển khai Ollama & Custom model từ Hugging Face

**Tạo thư mục project:**

```bash
sudo mkdir -p /opt/ai-platform/{ollama,dify,n8n}
sudo chown -R $USER:$USER /opt/ai-platform
cd /opt/ai-platform
```

**Tạo docker-compose cho Ollama:**

```bash
cat <<'EOF' > /opt/ai-platform/ollama/docker-compose.yml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_NUM_PARALLEL=2
      - OLLAMA_MAX_LOADED_MODELS=1
      - OLLAMA_KEEP_ALIVE=10m
    networks:
      - ai-network

volumes:
  ollama_data:

networks:
  ai-network:
    name: ai-network
    driver: bridge
EOF
```

> **Lưu ý:** Đây là config khởi tạo Ollama ban đầu (chưa mount models, chưa cấu hình tài nguyên). Config đầy đủ với volume mount, CPU limit và 2 model loaded sẽ được cập nhật ở **Bước 3** bên dưới.

**Khởi chạy Ollama:**

```bash
cd /opt/ai-platform/ollama
docker compose up -d

# Kiểm tra Ollama đã chạy
docker logs ollama -f
curl http://localhost:11434
# Output: "Ollama is running"
```

**Tải model GGUF từ Hugging Face và tạo custom model:**

Thay vì pull model mặc định từ Ollama library, ta tải bản GGUF quantized từ Hugging Face cho **text model** (kiểm soát mức quantize), và dùng `ollama pull` cho **vision model** (Llama 3.2 Vision đã được tối ưu sẵn trong Ollama Library).

> **Các mức quantize phổ biến (cho GGUF):**
>
> | Quantize | Đặc điểm | Khi nào dùng |
> |---|---|---|
> | Q8_0 | Gần gốc, nặng | Khi có GPU hoặc RAM dư thừa |
> | **Q5_K_M** | Cân bằng chất lượng/tốc độ | **Khuyến nghị cho model text (14b)** |
> | Q4_K_M | Nhẹ, nhanh | Khi RAM hạn chế |
> | Q3_K_M | Rất nhẹ, giảm chất lượng | Chỉ khi RAM rất hạn chế |

**Bước 1: Tải text model GGUF từ Hugging Face + Pull vision model từ Ollama**

```bash
# Tạo thư mục chứa model GGUF (cho text model)
mkdir -p /opt/ai-platform/models

# Cài hf CLI (Ubuntu 24.04 dùng pipx thay vì pip)
sudo apt install -y pipx
pipx install huggingface-hub
pipx ensurepath
source ~/.bashrc

# === Model TEXT (14b Q5_K_M ~10GB) - GGUF từ Hugging Face ===
hf download bartowski/Qwen2.5-Coder-14B-Instruct-GGUF \
  Qwen2.5-Coder-14B-Instruct-Q5_K_M.gguf \
  --local-dir /opt/ai-platform/models

# Kiểm tra file đã tải
ls -lh /opt/ai-platform/models/
# Qwen2.5-Coder-14B-Instruct-Q5_K_M.gguf         ~10 GB (text)

# === Model VISION (Llama 3.2 Vision 11b ~7.9GB) - từ Ollama Library ===
# Llama 3.2 Vision: native multimodal của Meta, không bias tiếng Trung, reasoning vượt LLaVA
docker exec -it ollama ollama pull llama3.2-vision
```

> **Lưu ý:** Nếu không cài được `hf`, dùng `wget` trực tiếp:
> ```bash
> wget -P /opt/ai-platform/models/ \
>   "https://huggingface.co/bartowski/Qwen2.5-Coder-14B-Instruct-GGUF/resolve/main/Qwen2.5-Coder-14B-Instruct-Q5_K_M.gguf"
> ```

> **Tại sao vision model dùng `ollama pull` thay vì GGUF?**
> Vision model cần cả language GGUF + vision projector (mmproj), phải match chính xác phiên bản. Dùng `ollama pull` đảm bảo 2 thành phần tương thích, đã được test kỹ, tránh lỗi runtime crash. **Lý do chọn Llama 3.2 Vision:** Native multimodal của Meta, không bias tiếng Trung, reasoning vượt trội hơn LLaVA, context 128K, hỗ trợ tiếng Việt tốt. Text model dùng GGUF từ HF vì chỉ có 1 file đơn giản, dễ kiểm soát quantize.


**Bước 2: Tạo 2 Modelfile tùy chỉnh**

**Modelfile cho `sysadmin-coder` (Text - 14b):**

```bash
cat <<'EOF' > /opt/ai-platform/models/Modelfile-sysadmin-coder
# Model TEXT: Qwen2.5-Coder 14b Q5_K_M
# Dùng cho: sinh script, phân tích log, viết tài liệu, chatbot Q&A
FROM /models/Qwen2.5-Coder-14B-Instruct-Q5_K_M.gguf

PARAMETER temperature 0.3
PARAMETER top_p 0.9
PARAMETER num_ctx 8192
PARAMETER num_thread 8
PARAMETER stop "<|im_end|>"
PARAMETER stop "<|endoftext|>"

SYSTEM """
Bạn là System Engineer AI Assistant cấp cao, thành thạo Linux và Windows Server.
Chuyên sinh script (Bash, PowerShell, Ansible, Terraform), phân tích log sâu, và viết tài liệu kỹ thuật.
Trả lời bằng tiếng Việt. Cung cấp lệnh/script cụ thể, có error handling.
Cảnh báo nếu lệnh nguy hiểm. Luôn đề xuất best practice và bảo mật.
"""

TEMPLATE """{{- if .System }}<|im_start|>system
{{ .System }}<|im_end|>
{{ end }}
{{- range .Messages }}<|im_start|>{{ .Role }}
{{ .Content }}<|im_end|>
{{ end }}<|im_start|>assistant
"""
EOF
```

**Modelfile cho `sysadmin-vision` (Vision - Llama 3.2 Vision 11b):**

```bash
cat <<'EOF' > /opt/ai-platform/models/Modelfile-sysadmin-vision
# Model VISION: Llama 3.2 Vision (11b) - từ Ollama Library
# Dùng cho: phân tích screenshot, đọc diagram, OCR log từ ảnh
# Native multimodal của Meta, không bias tiếng Trung, hỗ trợ tiếng Việt tốt
FROM llama3.2-vision

PARAMETER temperature 0.5
PARAMETER top_p 0.8
PARAMETER num_ctx 4096
PARAMETER num_thread 8

SYSTEM """
Bạn là System Engineer Vision Assistant, chuyên phân tích hình ảnh kỹ thuật.
Khi nhận ảnh screenshot, dashboard, diagram hoặc log: mô tả chi tiết những gì thấy,
nhận diện lỗi/cảnh báo, và đề xuất giải pháp cụ thể.
Trả lời bằng tiếng Việt. Nếu ảnh là dashboard/log: phân tích metric, chỉ ra vấn đề.
"""
EOF
```


> **Sự khác biệt giữa 2 Modelfile:**
>
> | Tham số | sysadmin-coder (Text 14b) | sysadmin-vision (Vision 11b) |
> |---|---|---|
> | Base model | Qwen2.5-Coder-14B (GGUF từ HF) | Llama 3.2 Vision 11b (`ollama pull`) |
> | `FROM` | Trỏ file GGUF cụ thể | `FROM llama3.2-vision` (model đã pull) |
> | `temperature` | 0.3 (chính xác cho code) | 0.5 (linh hoạt cho mô tả ảnh) |
> | `num_ctx` | 8192 (8K - cân bằng RAM/context) | 4096 (4K - đủ cho ảnh + prompt) |
> | `num_thread` | 8 (dùng 8/16 core) | 8 (dùng 8/16 core) |
> | System prompt | Suy luận, thiết kế, debug; sinh script, phân tích log | Xử lý hình ảnh: screenshot, dashboard, diagram, OCR |
> | Khả năng | Text only | **Text + Image input** |

**Bước 3: Mount thư mục models vào container Ollama**

Cập nhật docker-compose để mount thư mục chứa GGUF:

```bash
cat <<'EOF' > /opt/ai-platform/ollama/docker-compose.yml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
      - /opt/ai-platform/models:/models
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_NUM_PARALLEL=1
      - OLLAMA_MAX_LOADED_MODELS=2
      - OLLAMA_KEEP_ALIVE=60m
    deploy:
      resources:
        limits:
          cpus: '10'
    networks:
      - ai-network

volumes:
  ollama_data:

networks:
  ai-network:
    name: ai-network
    driver: bridge
EOF
```

> **Lưu ý cấu hình Ollama:**
> - `OLLAMA_MAX_LOADED_MODELS=2` — load cả 2 model (`sysadmin-coder` + `sysadmin-vision`) vào RAM đồng thời, không có swap delay (~3-5s) khi chuyển model
> - `OLLAMA_NUM_PARALLEL=1` — giảm xuống 1 request đồng thời để tiết kiệm KV cache RAM (đủ cho lab 1-2 user)
> - `OLLAMA_KEEP_ALIVE=60m` — giữ model trong RAM lâu hơn, tránh reload
> - `deploy.resources.limits.cpus: '10'` — hard cap CPU ở tầng Docker, Ollama chỉ dùng tối đa 10/16 core, dành 6 core cho OS/Dify/n8n — **tránh VM bị treo khi inference**
> - Không đặt `deploy.resources.limits.memory` — để Ollama dùng toàn bộ RAM host tự do
>
> **Tính toán RAM với 2 model loaded:** ~13GB (text) + ~9GB (vision) + ~6GB (services/OS) = **~28GB/32GB** → còn ~4GB free + 8GB swap làm đệm an toàn.

```bash
# Restart Ollama với volume mới
cd /opt/ai-platform/ollama
docker compose up -d
```

> **⚠️ Lưu ý hiệu năng CPU:** Khi Ollama inference, llama.cpp mặc định dùng **tất cả CPU core** có sẵn — trên VM 16 core sẽ thấy `~1500% CPU` trong `top`. Để tránh treo VM, cần giới hạn ở 2 tầng:
> 1. `PARAMETER num_thread 8` trong Modelfile — giới hạn inference thread của llama.cpp
> 2. `deploy.resources.limits.cpus: '10'` trong docker-compose — hard cap ở tầng Docker
> Kết hợp 2 giới hạn này đảm bảo Ollama dùng tối đa ~8 core thực tế, dành 6-8 core cho OS/Dify/n8n.

**Bước 4: Tạo 2 custom model trong Ollama**

```bash
# Tạo model TEXT (14b) - cho code, log, tài liệu
docker exec -it ollama ollama create sysadmin-coder -f /models/Modelfile-sysadmin-coder

# Tạo model VISION (11b) - cho xử lý hình ảnh screenshot, dashboard, diagram
docker exec -it ollama ollama create sysadmin-vision -f /models/Modelfile-sysadmin-vision

# Kiểm tra cả 2 model đã tạo
docker exec -it ollama ollama list
```


**Bước 5: Test cả 2 model**

```bash
# Test model TEXT (14b) - sinh script phức tạp
docker exec -it ollama ollama run sysadmin-coder "Viết script bash kiểm tra disk usage trên Linux, cảnh báo khi partition vượt 80%"

# Test model VISION (11b) - mô tả ảnh (test text mode)
docker exec -it ollama ollama run sysadmin-vision "Mô tả chi tiết những gì bạn thấy trong một Grafana dashboard điển hình có CPU, RAM, Disk"

# So sánh tốc độ qua API
echo "=== TEXT (14b) ==="
time curl -s http://localhost:11434/api/generate -d '{
  "model": "sysadmin-coder",
  "prompt": "Liệt kê 5 lệnh kiểm tra disk trên Linux",
  "stream": false
}' | python3 -c "import sys,json; print(json.load(sys.stdin)['response'][:200])"

echo "=== VISION (11b) ==="
time curl -s http://localhost:11434/api/generate -d '{
  "model": "sysadmin-vision",
  "prompt": "Liệt kê 5 lệnh kiểm tra disk trên Linux",
  "stream": false
}' | python3 -c "import sys,json; print(json.load(sys.stdin)['response'][:200])"
```

> **Kiểm tra resource tiêu thụ:**
> ```bash
> docker stats ollama
> # Khi text model loaded: ~12-14GB RAM
> # Khi vision model loaded: ~8-10GB RAM
> ```

> **⚠️ Troubleshooting: Lỗi "model requires more system memory"**
>
> Nếu gặp lỗi:
> ```
> Error: model requires more system memory (33.3 GiB) than is available (25.1 GiB)
> ```
> **Nguyên nhân:** `num_ctx` quá lớn (ví dụ 32768) kết hợp `OLLAMA_NUM_PARALLEL` cao → Ollama cấp phát KV cache = `num_ctx × NUM_PARALLEL × kích thước mỗi token` → vượt RAM.
>
> **Cách fix:**
> ```bash
> # 1. Giảm num_ctx trong Modelfile (đã fix ở trên: 8192 cho text, 4096 cho vision)
> # 2. Giảm OLLAMA_NUM_PARALLEL xuống 1 (đã fix ở docker-compose)
> # 3. Không đặt deploy.resources.limits.memory cứng trong docker-compose
> # 4. Tạo lại model sau khi sửa Modelfile:
> docker exec -it ollama ollama create sysadmin-coder -f /models/Modelfile-sysadmin-coder
> docker exec -it ollama ollama create sysadmin-vision -f /models/Modelfile-sysadmin-vision
> ```
>
> **Công thức ước tính RAM:**
> RAM ≈ Model weights + (num_ctx × NUM_PARALLEL × KV_size_per_token)
> Ví dụ 14b Q5_K_M: 10GB + (32768 × 4 × ~180B) ≈ 10GB + 23GB = **33GB** → OOM!
> Sau fix: 10GB + (8192 × 2 × ~180B) ≈ 10GB + 2.8GB = **~13GB** → OK ✓

> **⚠️ Troubleshooting: Vision model GGUF crash (segfault)**
>
> Nếu bạn thử tải vision model GGUF từ Hugging Face (ví dụ Qwen3-VL, LLaVA) và gặp lỗi:
> ```
> model runner has unexpectedly stopped (exit status 2)
> ```
> Kèm register dump trong log (`docker logs ollama`) — đây là **segfault** trong llama.cpp runner.
>
> **Nguyên nhân:** Một số kiến trúc vision model (đặc biệt Qwen3-VL) chưa được Ollama hỗ trợ ổn định. File GGUF + mmproj có thể tải đúng, `ollama show` hiện đúng kiến trúc, nhưng runner crash khi inference.
>
> **Giải pháp:** Dùng vision model đã được test kỹ trong Ollama library:
> ```bash
> ollama pull llama3.2-vision   # Llama 3.2 Vision 11b - native multimodal Meta, tiếng Việt tốt
> ollama pull llava              # LLaVA 1.6 - nhẹ hơn (7b), nếu RAM hạn chế
> ollama pull llava:13b          # LLaVA 13b - chất lượng cao hơn LLaVA 7b
> ```
> Sau đó tạo custom model với `FROM llama3.2-vision` (hoặc `FROM llava`) trong Modelfile thay vì `FROM .`.
> **Lưu ý:** Tránh dùng MiniCPM-V, InternVL, hoặc các vision model base Qwen/Baidu — thường fallback về tiếng Trung.

**(Tùy chọn) Tạo thêm model embedding cho RAG:**

```bash
# Pull embedding model (nhỏ, dùng model có sẵn của Ollama)
docker exec -it ollama ollama pull nomic-embed-text
```

---

### 6. Triển khai Dify - AI Application Platform

Dify là nền tảng mã nguồn mở cho phép xây dựng AI application (chatbot, agent, RAG, workflow) với giao diện kéo thả trực quan.

**Clone Dify và cấu hình:**

```bash
cd /opt/ai-platform
git clone https://github.com/langgenius/dify.git
cd dify/docker
```

**Chỉnh sửa file `.env`:**

```bash
cp .env.example .env

# Tạo secret key random
SECRET_KEY=$(openssl rand -hex 32)

# Sửa các biến quan trọng trong .env
sed -i "s|^SECRET_KEY=.*|SECRET_KEY=${SECRET_KEY}|" .env
sed -i "s|^INIT_PASSWORD=.*|INIT_PASSWORD=YourStrongPassword123!|" .env
sed -i "s|^STORAGE_TYPE=.*|STORAGE_TYPE=local|" .env
sed -i "s|^VECTOR_STORE=.*|VECTOR_STORE=weaviate|" .env
sed -i "s|^EXPOSE_NGINX_PORT=.*|EXPOSE_NGINX_PORT=80|" .env
sed -i "s|^EXPOSE_NGINX_SSL_PORT=.*|EXPOSE_NGINX_SSL_PORT=443|" .env
```

**Khởi chạy Dify:**

```bash
docker compose up -d

# Kiểm tra tất cả container đã chạy
docker compose ps
```

> Dify sẽ khởi chạy nhiều container: `api`, `worker`, `web`, `nginx`, `db` (PostgreSQL), `redis`, `weaviate`, `sandbox`, `ssrf_proxy`.

**Chờ khoảng 1-2 phút** cho tất cả service khởi động xong, sau đó truy cập:

```
http://<IP-VM>:80
```

- Lần đầu tiên sẽ hiện trang **đăng ký admin account**
- Tạo tài khoản admin với email và password

---

### 7. Triển khai n8n - Workflow Automation

n8n là nền tảng workflow automation mã nguồn mở, hỗ trợ hàng trăm integration node, bao gồm AI/LLM nodes.

```bash
cat <<'EOF' > /opt/ai-platform/n8n/docker-compose.yml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
    environment:
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://10.10.200.11:5678/
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=YourN8nPassword123!
      - GENERIC_TIMEZONE=Asia/Ho_Chi_Minh
      - N8N_AI_ENABLED=true
    networks:
      - ai-network

volumes:
  n8n_data:

networks:
  ai-network:
    external: true
    name: ai-network
EOF
```

> **Lưu ý:** `N8N_AI_ENABLED=true` kích hoạt AI nodes trong n8n (LangChain, AI Agent, etc.). Network `ai-network` khai báo `external: true` vì đã được tạo bởi Ollama compose.

**Khởi chạy n8n:**

```bash
cd /opt/ai-platform/n8n
docker compose up -d

# Kiểm tra
docker logs n8n -f
```

Truy cập n8n tại: `http://10.10.200.11:5678`

- Đăng nhập bằng `admin` / `YourN8nPassword123!`

---

### 8. Tích hợp Ollama vào Dify

**Bước 1: Thêm Model Provider - sysadmin-coder (14b)**

1. Đăng nhập Dify → vào **Settings** (icon bánh răng góc trên phải)
2. Chọn **Model Provider** → tìm **Ollama**
3. Click **Setup** và điền:
   - **Model Name:** `sysadmin-coder`
   - **Base URL:** `http://ollama:11434` (nếu Dify chạy cùng Docker network) hoặc `http://<IP-VM>:11434`
   - **Model Type:** LLM
   - **Context Size:** `8192` (8K - theo cấu hình trong Modelfile)

4. Click **Save** → Dify sẽ test kết nối đến Ollama

**Bước 2: Thêm Model Provider - Vision (11b)**

1. Quay lại **Model Provider** → **Ollama** → **Add Model**
2. Điền:
   - **Model Name:** `sysadmin-vision`
   - **Base URL:** `http://ollama:11434`
   - **Model Type:** LLM
   - **Model Features:** ☑ Vision (tick chọn hỗ trợ ảnh)
   - **Context Size:** `4096` (4K)

**Bước 3: Thêm Embedding Model (cho RAG)**

1. Tiếp tục **Add Model**:
   - **Model Name:** `nomic-embed-text`
   - **Base URL:** `http://ollama:11434`
   - **Model Type:** Text Embedding

**Bước 4: Cấu hình System Model**

1. Vào **Settings** → **Model Provider** → **System Model Settings**
2. Chọn:
   - **System Reasoning Model:** `sysadmin-coder` (Ollama) — dùng model text mạnh làm mặc định cho hệ thống
   - **Embedding Model:** `nomic-embed-text` (Ollama)

> **Lưu ý:** System Model chỉ là mặc định. Khi tạo từng App, ta sẽ chọn model phù hợp (coder cho text, vision cho phân tích ảnh).

> **Lưu ý Docker Network:** Dify dùng Docker network riêng. Để Dify kết nối được Ollama, cần đảm bảo cùng network hoặc dùng IP host:
> ```bash
> # Cách 1: Kết nối container Dify vào ai-network
> docker network connect ai-network dify-api-1
> docker network connect ai-network dify-worker-1
>
> # Cách 2: Dùng host IP
> # Base URL: http://10.10.200.11:11434
> ```

---

### 9. Tích hợp Ollama/Dify vào n8n

**Cách 1: Kết nối trực tiếp n8n → Ollama**

1. Trong n8n, tạo **Credential** mới:
   - Type: **Ollama API**
   - Base URL: `http://ollama:11434` (cùng Docker network)

2. Sử dụng các AI nodes:
   - **Ollama Chat Model**: Chọn `sysadmin-coder` cho phân tích text, `sysadmin-vision` cho phân tích ảnh
   - **AI Agent**: Tạo agent với tool-calling
   - **AI Chain**: Xây dựng LangChain pipeline

**Cách 2: Kết nối n8n → Dify API**

1. Trong Dify, tạo một **App** (ví dụ chatbot SysAdmin)
2. Vào **API Access** → copy **API Key** và **API Endpoint**
3. Trong n8n, dùng **HTTP Request** node:
   - Method: `POST`
   - URL: `http://<IP-VM>:80/v1/chat-messages`
   - Headers: `Authorization: Bearer <DIFY-API-KEY>`
   - Body:
   ```json
   {
     "inputs": {},
     "query": "{{ $json.message }}",
     "response_mode": "blocking",
     "user": "n8n-automation"
   }
   ```

---

### 10. Use Case 1 - Chatbot hỗ trợ SysAdmin

Tạo chatbot trên Dify có khả năng trả lời câu hỏi về system administration, tích hợp knowledge base nội bộ.

**Bước 1: Tạo Knowledge Base**

1. Trong Dify → **Knowledge** → **Create Knowledge**
2. Upload các tài liệu kỹ thuật:
   - Runbook nội bộ (`.md`, `.pdf`, `.docx`)
   - Cấu hình chuẩn (Ansible playbooks, Terraform configs)
   - SOP xử lý sự cố
   - Man pages, cheat sheets
3. Dify sẽ tự động chunk và embedding tài liệu vào vector database

**Bước 2: Tạo Chatbot App**

1. **Studio** → **Create App** → chọn **Chatbot**
2. Đặt tên: `SysAdmin Assistant`
3. Cấu hình **System Prompt:**

```
Bạn là một System Engineer Assistant chuyên nghiệp, thành thạo cả Linux và Windows Server. Nhiệm vụ của bạn:

1. Trả lời câu hỏi về Linux & Windows Server administration, networking, security
2. Viết và giải thích script (Bash, PowerShell, Python, Ansible, Terraform)
3. Phân tích log và đề xuất giải pháp khắc phục sự cố (syslog, Event Viewer, dmesg...)
4. Hướng dẫn cấu hình dịch vụ Linux: Docker, Kubernetes, HAProxy, Nginx, Ceph, etc.
5. Hướng dẫn cấu hình dịch vụ Windows: Active Directory, DNS, DHCP, GPO, IIS, Hyper-V, WSUS, etc.
6. Tạo tài liệu kỹ thuật theo yêu cầu

Quy tắc:
- Luôn trả lời bằng tiếng Việt
- Cung cấp lệnh/script cụ thể, có thể chạy ngay
- Giải thích từng bước rõ ràng
- Cảnh báo nếu lệnh có thể gây nguy hiểm (rm -rf, fdisk, iptables flush, Format-Volume, Remove-ADUser...)
- Đề xuất best practice và bảo mật
- Nếu không chắc chắn, nói rõ và đề xuất cách kiểm tra thêm
```

4. **Context** → thêm Knowledge Base đã tạo ở bước 1
5. **Model:** chọn `sysadmin-coder` (Ollama) — dùng model text mạnh cho chatbot Q&A, trả lời chính xác và chi tiết
6. **Publish** → lấy URL để truy cập chatbot

> **Tại sao dùng `sysadmin-coder` (14b) cho chatbot?** Model 14b cho output chính xác hơn, kết hợp với RAG từ Knowledge Base cho trải nghiệm tốt nhất. Không cần model nhanh vì chatbot cho phép chờ vài giây.

**Test chatbot:**
- "Hướng dẫn cấu hình HAProxy load balancing cho 3 backend server"
- "Phân tích log này và cho biết nguyên nhân: `kernel: [UFW BLOCK] IN=ens192 OUT= ...`"
- "Viết Ansible playbook cài đặt Docker trên 10 server Ubuntu"
- "Viết PowerShell script join domain hàng loạt cho 20 máy Windows Server 2022"
- "Hướng dẫn cấu hình Active Directory với 2 site và replication"
- "Phân tích Event ID 4625 liên tục trong Security log của Domain Controller"

---

### 11. Use Case 2 - Tự động phân tích log & tạo ticket

Xây dựng workflow n8n: nhận alert từ Prometheus Alertmanager → AI phân tích → tạo ticket.

**Workflow n8n:**

```
[Webhook] → [Set Fields] → [Ollama Chat] → [IF severity] → [Create Ticket / Send Telegram]
```

**Bước 1: Tạo Webhook node**

1. Tạo workflow mới trong n8n
2. Thêm node **Webhook**:
   - HTTP Method: `POST`
   - Path: `alert-ai-analysis`
   - Copy webhook URL: `http://<IP-VM>:5678/webhook/alert-ai-analysis`

**Bước 2: Thêm Ollama Chat Model node**

1. Thêm node **Basic LLM Chain** hoặc **AI Agent**
2. Kết nối với **Ollama Chat Model**:
   - Model: `sysadmin-coder` — dùng model text cho alert triage (phân tích log và output JSON)
3. Prompt template:

```
Bạn là AI phân tích alert cho hệ thống giám sát. Phân tích alert sau và trả về JSON:

Alert: {{ $json.alertname }}
Severity: {{ $json.severity }}
Instance: {{ $json.instance }}
Description: {{ $json.description }}
Value: {{ $json.value }}

Trả về JSON format:
{
  "summary": "Tóm tắt ngắn gọn",
  "root_cause": "Nguyên nhân có thể",
  "impact": "Ảnh hưởng đến hệ thống",
  "action_required": ["Bước 1", "Bước 2", "..."],
  "priority": "critical/high/medium/low",
  "auto_fix_script": "Script tự động fix nếu có thể (hoặc null)"
}
```

**Bước 3: Cấu hình Alertmanager gửi webhook**

```yaml
# alertmanager.yml
route:
  receiver: 'n8n-ai'
  
receivers:
  - name: 'n8n-ai'
    webhook_configs:
      - url: 'http://10.10.200.11:5678/webhook/alert-ai-analysis'
        send_resolved: true
```

**Bước 4: Thêm logic xử lý kết quả**

- Node **IF**: Kiểm tra priority
  - `critical/high` → Gọi `sysadmin-coder` (14b) phân tích sâu + `sysadmin-vision` (11b) nếu có ảnh đính kèm → Gửi Telegram + tạo ticket kèm root cause analysis
  - `medium/low` → Ghi log + gửi email summary

> **Dual-model trong workflow:** `sysadmin-coder` (14b) phân tích log text và phân loại alert. Khi alert có đính kèm screenshot (Grafana, Zabbix screen capture), gọi thêm `sysadmin-vision` (11b) để AI nhìn ảnh và bổ sung phân tích.

---

### 12. Use Case 3 - Sinh script Bash/Ansible tự động

**Workflow Dify: Script Generator Agent**

1. Tạo **Agent** mới trong Dify
2. Cấu hình:
   - **Model:** `sysadmin-coder` — dùng model text (14b) vì cần sinh code chính xác, có error handling
   - **Agent Mode:** Function Calling
3. System Prompt:

```
Bạn là Script Generator cho System Engineer. Khi user mô tả yêu cầu:

1. Hỏi rõ: target OS, số lượng server, yêu cầu cụ thể
2. Sinh script hoàn chỉnh (Bash/Ansible/Terraform tùy yêu cầu)
3. Thêm error handling, logging, và validation
4. Giải thích từng phần quan trọng
5. Cung cấp hướng dẫn chạy script

Output format:
- Script có comment rõ ràng
- Có biến cấu hình ở đầu file
- Có dry-run mode nếu có thể
- Tuân thủ best practices (shellcheck, ansible-lint)
```

**Ví dụ prompt:**
> "Tạo Ansible playbook triển khai cluster Kubernetes 3 node (1 master + 2 worker) trên Ubuntu 24.04 với Cilium CNI"

**Tích hợp vào n8n workflow:**

```
[Telegram Bot] → [Dify API] → [Parse Response] → [Send Script to Telegram/Email]
```

---

### 13. Use Case 4 - Tạo tài liệu kỹ thuật tự động

**Workflow: Tự động tạo Post-Mortem Report**

1. Input: Mô tả sự cố (thời gian, hệ thống affected, symptom)
2. AI sinh ra post-mortem template đầy đủ:
   - Timeline sự kiện
   - Root Cause Analysis
   - Impact Assessment
   - Action Items
   - Prevention measures

**Prompt template trong Dify:**

```
Tạo Post-Mortem Report theo template sau dựa trên thông tin sự cố:

## Post-Mortem Report
**Incident ID:** [Auto-generate]
**Date:** [Current date]
**Author:** System AI Assistant
**Status:** Draft - Cần review bởi engineer

### 1. Tóm tắt sự cố
### 2. Timeline
### 3. Root Cause Analysis (5 Whys)
### 4. Impact
### 5. Mitigation & Resolution
### 6. Action Items
### 7. Lessons Learned

Thông tin sự cố: {user_input}
```

---

### 14. Tối ưu hiệu năng & Monitoring

**Kiểm tra resource usage:**

```bash
# Tổng quan tất cả container
docker stats --no-stream

# Kiểm tra disk usage
docker system df -v

# Kiểm tra Ollama model loaded
curl -s http://localhost:11434/api/ps | python3 -m json.tool
```

**Tối ưu Ollama cho CPU-only inference (config tham khảo cho VM <32GB RAM):**

> **Lưu ý:** Config bên dưới là ví dụ tham khảo cho VM có RAM hạn chế (16-24GB). **Với lab 32GB này**, config thực tế đã được đặt ở Bước 3 Section 5 (`NUM_PARALLEL=1, MAX_LOADED_MODELS=2, KEEP_ALIVE=60m`).

```bash
# Tham khảo: cấu hình bảo thủ cho VM <32GB RAM
docker exec -it ollama bash
export OLLAMA_NUM_PARALLEL=1        # 1 request đồng thời, tiết kiệm KV cache RAM
export OLLAMA_MAX_LOADED_MODELS=1   # Load 1 model tại 1 thời điểm, tự swap khi cần
```

**Hoặc chỉnh docker-compose Ollama (cho VM <32GB RAM):**

```yaml
environment:
  - OLLAMA_HOST=0.0.0.0
  - OLLAMA_NUM_PARALLEL=1
  - OLLAMA_MAX_LOADED_MODELS=1      # Ollama tự swap model khi gọi model khác
  - OLLAMA_KEEP_ALIVE=10m           # Unload model sau 10 phút không dùng
```

> **Tip: Load cả 2 model đồng thời để loại bỏ swap delay (~3-5s)**
> Với VM 32GB RAM, có thể load cả 2 model vào RAM cùng lúc — không cần chờ swap khi chuyển model:
> ```yaml
> # Cập nhật docker-compose Ollama (environment):
>   - OLLAMA_MAX_LOADED_MODELS=2    # Load cả 2 model đồng thời
>   - OLLAMA_NUM_PARALLEL=1         # Giảm xuống 1 để tiết kiệm KV cache RAM
>   - OLLAMA_KEEP_ALIVE=60m         # Giữ model lâu hơn trong RAM
> ```
> Sau khi restart Ollama, pre-warm cả 2 model:
> ```bash
> # Pre-load cả 2 model vào RAM ngay lập tức
> curl -s http://localhost:11434/api/generate -d '{"model":"sysadmin-coder","prompt":"","keep_alive":"60m"}' > /dev/null
> curl -s http://localhost:11434/api/generate -d '{"model":"sysadmin-vision","prompt":"","keep_alive":"60m"}' > /dev/null
> # Kiểm tra cả 2 đang loaded trong RAM
> curl -s http://localhost:11434/api/ps | python3 -m json.tool
> ```
> **Tính toán RAM:** ~13GB (text) + ~9GB (vision) + ~6GB (services/OS) = **~28GB** → còn ~4GB free + 8GB swap làm đệm an toàn.
> **Đánh đổi:** `OLLAMA_NUM_PARALLEL=1` (1 request đồng thời) thay vì 2 — đủ cho lab 1-2 user. Nếu cần nhiều user đồng thời, giữ `MAX_LOADED_MODELS=1`.

**Monitoring với ctop:**

```bash
# Cài ctop để monitor container realtime
sudo wget https://github.com/bcicen/ctop/releases/download/v0.7.7/ctop-0.7.7-linux-amd64 -O /usr/local/bin/ctop
sudo chmod +x /usr/local/bin/ctop
ctop
```

**Phân bổ tài nguyên ước tính:**

| Service | RAM (idle) | RAM (active) | CPU |
|---|---|---|---|
| Ollama + sysadmin-coder (14b Q5_K_M) | ~2 GB | ~13 GB | **~8 core** (num_thread=8) |
| Ollama + sysadmin-vision (Llama 3.2 Vision 11b) | ~1 GB | ~9 GB | 4-8 cores |
| Dify (all containers) | ~2 GB | ~4 GB | 2-4 cores |
| n8n | ~256 MB | ~512 MB | 1-2 cores |
| OS + Docker | ~1.5 GB | ~2 GB | - |
| **Tổng (1 model loaded + services)** | **~7 GB** | **~19 GB** | **16 cores** |
| **Tổng (2 model loaded đồng thời)** | **~7 GB** | **~28 GB** | **16 cores** |

> Với `OLLAMA_MAX_LOADED_MODELS=1` (mặc định): swap khi đổi model mất ~3-5s, an toàn nhất, headroom ~13GB. Với `OLLAMA_MAX_LOADED_MODELS=2`: cả 2 model loaded sẵn trong RAM, **không có swap delay**, tổng ~28GB/32GB — phù hợp lab 32GB có swap 8GB làm đệm.
> `num_ctx` được đặt 8192 (text) và 4096 (vision) để cân bằng giữa chất lượng và RAM. Nếu cần context dài hơn, tăng `num_ctx` nhưng phải giảm `OLLAMA_NUM_PARALLEL`.

**Backup dữ liệu:**

```bash
# Backup volumes
mkdir -p ~/backup

# Backup Dify data
docker compose -f /opt/ai-platform/dify/docker/docker-compose.yml exec db pg_dump -U postgres dify > ~/backup/dify-db-$(date +%Y%m%d).sql

# Backup n8n data
docker cp n8n:/home/node/.n8n ~/backup/n8n-data-$(date +%Y%m%d)

# Backup Ollama models
docker cp ollama:/root/.ollama ~/backup/ollama-data-$(date +%Y%m%d)
```

---

### 15. Kết luận

Bài lab đã hướng dẫn xây dựng một **AI Platform hoàn chỉnh tự host** trên một VM duy nhất (16 CPU, 32GB RAM, 100GB disk) với:

| Thành phần | URL | Chức năng |
|---|---|---|
| Ollama | `http://<IP>:11434` | LLM inference engine (text + vision) |
| Dify | `http://<IP>:80` | AI app builder (chatbot, agent, RAG) |
| n8n | `http://<IP>:5678` | Workflow automation + AI integration |

**Điểm mạnh của kiến trúc:**
- **100% self-hosted** - Không phụ thuộc API bên ngoài, dữ liệu không rời khỏi nội bộ
- **Chiến lược dual-model Text + Vision** - `sysadmin-coder` (14b) cho code/log/tài liệu/chatbot, `sysadmin-vision` (Llama 3.2 Vision 11b) cho phân tích ảnh/screenshot/diagram/OCR
- **Linh hoạt nguồn model** - Text model: GGUF từ Hugging Face (kiểm soát quantize), Vision model: `ollama pull llama3.2-vision` (native multimodal Meta, tiếng Việt ổn định), tùy chỉnh Modelfile riêng cho từng use case, chạy tốt trên CPU-only
- **Dify** - Giao diện trực quan, dễ tạo chatbot/agent mà không cần code
- **n8n** - Kết nối AI với hệ thống monitoring/ticketing/communication hiện có, routing text/vision model theo tác vụ
- **Chi phí bằng 0** - Tất cả đều open-source, chỉ cần 1 VM

**Hướng phát triển tiếp:**
- Thêm GPU passthrough (nếu có) để tăng tốc inference 5-10x
- Tích hợp với Grafana OnCall, PagerDuty, Jira
- Xây dựng knowledge base từ wiki/confluence nội bộ
- Fine-tune model trên dữ liệu sự cố nội bộ
- Deploy High Availability với nhiều node Ollama
- Thêm model chuyên biệt (code review, security audit) khi có thêm RAM
- Thêm workflow vision: chụp màn hình Grafana tự động → AI phân tích dashboard → báo cáo weekly
