# AI Network Docker Stack

This repository contains multiple `docker-compose` configuration files for running an **AI service network** with shared networking and GPU acceleration.
A dedicated container (`ai-net`) acts as the **parent network namespace**, allowing AI services to communicate via `localhost` without exposing unnecessary ports to the host.

---

## 📦 Services Overview

### 1. **ai-net** (`ai-net.yaml`)

* **Purpose**: Central network namespace for all AI containers.
* **Image**: `alpine:latest`
* **Special Notes**:

  * Fixed container name: `ai-net` (required for network sharing).
  * Only this container maps ports to the host.
  * Uses the external Docker network `nat99_net` with a static IP.

---

### 2. **vLLM** (`vllm.yaml`)

* **Purpose**: Runs [vLLM](https://github.com/vllm-project/vllm) as an OpenAI-compatible API.
* **Image**: `vllm/vllm-openai`
* **Network**: Shares network stack with `ai-net`.
* **Command Example**:

  ```bash
  --model HuggingFaceUser/ModelName --host 0.0.0.0 --port 8000
  ```
* **GPU Support**: NVIDIA runtime enabled.
* **Persistence**: Model cache stored in `huggingface_cache`.

---

### 3. **Open WebUI** (`openwebui.yaml`)

* **Purpose**: Web interface for interacting with LLMs.
* **Image**: `ghcr.io/open-webui/open-webui:main`
* **Network**: Shares `ai-net` stack.
* **Environment Variables**:

  * `PORT=8082`
  * `OLLAMA_BASE_URL=http://localhost:11434` (connects to Ollama in same network)
* **Persistence**: Data stored in `openwebui_data`.

---

### 4. **Ollama (GPU)** (`ollama.yaml`)

* **Purpose**: [Ollama](https://ollama.com) server with GPU acceleration.
* **Image**: `ollama/ollama`
* **Network**: Shares `ai-net` stack.
* **GPU**: All NVIDIA GPUs enabled.
* **Persistence**: `ollama_gpu_data` volume.

---

### 5. **Ollama (CPU)** (`ollama-cpu.yaml`)

* **Purpose**: CPU-only Ollama instance.
* **Image**: `ollama/ollama`
* **Network**: Shares `ai-net` stack.
* **Environment**:

  * `CUDA_VISIBLE_DEVICES=` (disables GPU)
  * `ROCM_VISIBLE_DEVICES=` (disables AMD GPU)
  * `OLLAMA_HOST=0.0.0.0:11435` (alternate port)
* **Persistence**: `ollama_cpu_data`.

---

### 6. **llama.cpp** (`llamacpp.yaml`)

* **Purpose**: [llama.cpp](https://github.com/ggerganov/llama.cpp) server.
* **Image**: `ghcr.io/ggerganov/llama.cpp:server-cuda-cu121`
* **Network**: Shares `ai-net` stack.
* **Command Example**:

  ```bash
  --model /models/your-model.gguf --host 0.0.0.0 --port 8081 --n-gpu-layers -1
  ```
* **GPU**: NVIDIA runtime.
* **Persistence**: Models in `model_storage`.

---

### 7. **ComfyUI (Standalone)** (`comfyui.yaml`)

* **Purpose**: [ComfyUI](https://github.com/comfyanonymous/ComfyUI) image generation workflow.
* **Image**: `yanwk/comfyui-boot:cu126-slim`
* **Network**: Uses `nat99_net` with unique static IP.
* **Ports**: Exposes `8188` to host.
* **Persistence**: `comfyui` (S3FS driver for remote storage).

---

### 8. **ComfyUI (Inside ai-net)** (`comf.yaml`)

* **Purpose**: ComfyUI running inside `ai-net` network namespace.
* **Image**: `yanwk/comfyui-boot:cu126-slim`
* **Network**: `network_mode: "container:ai-net"`
* **GPU**: NVIDIA runtime.
* **Persistence**: `comfyui`.

---

## 🌐 Networking Architecture

* **Centralized Network**:
  Services using `network_mode: "container:ai-net"` share the same localhost and IP.
* **External Network (`nat99_net`)**:
  Allows `ai-net` and standalone containers to have static IPs.

---

## 🚀 Deployment

### 1. Create External Network

```bash
docker network create \
  --driver=bridge \
  --subnet=192.168.99.0/24 \
  nat99_net
```

### 2. Start `ai-net`

```bash
docker compose -f ai-net.yaml up -d
```

### 3. Start Other Services

```bash
docker compose -f vllm.yaml up -d
docker compose -f openwebui.yaml up -d
docker compose -f ollama.yaml up -d
docker compose -f ollama-cpu.yaml up -d
docker compose -f llamacpp.yaml up -d
docker compose -f comf.yaml up -d
```

For standalone ComfyUI:

```bash
docker compose -f comfyui.yaml up -d
```

---

## 🖥 Enabling Server Access to `nat99_net`

By default, `nat99_net` is internal to Docker.
If you want the **host server** (or other machines) to reach containers in `nat99_net`, follow these steps:

### 1. Add a Host Interface for `nat99_net`

```bash
sudo ip link add br-nat99 link docker0 type macvlan mode bridge
sudo ip addr add 192.168.99.1/24 dev br-nat99
sudo ip link set br-nat99 up
```

Here `192.168.99.1` is the host’s IP in that subnet.

---

### 2. Enable IP Forwarding & NAT (Optional for external access)

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Replace eth0 with your internet-facing interface
sudo iptables -t nat -A POSTROUTING -s 192.168.99.0/24 -o eth0 -j MASQUERADE
```

---

### 3. Test Connectivity

```bash
ping 192.168.99.91   # ai-net container IP
curl http://192.168.99.91:8188
```

If successful, your host can directly access all containers in `nat99_net`.

---

## 📁 Volumes

| Volume Name        | Purpose                  |
| ------------------ | ------------------------ |
| huggingface\_cache | HuggingFace model cache  |
| openwebui\_data    | Open WebUI data storage  |
| ollama\_gpu\_data  | Ollama GPU model storage |
| ollama\_cpu\_data  | Ollama CPU model storage |
| model\_storage     | llama.cpp model storage  |
| comfyui            | ComfyUI workflows & data |

---

## 🖥 GPU Requirements

For GPU-enabled services:

* Install NVIDIA drivers on the host.
* Install [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html).
* Docker Compose GPU config:

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all
          capabilities: [gpu]
```

---

## 📌 Notes

* **Fixed Container Names**: Required when using `network_mode: "container:ai-net"`.
* **Port Mapping**: Only `ai-net` or standalone services expose ports to the host.
* **Service-to-Service Communication**: Use `localhost` when sharing the `ai-net` stack.

