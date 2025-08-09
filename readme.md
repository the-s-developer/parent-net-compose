# AI Network Docker Stack

This stack runs multiple AI services that **share the same network namespace** through a dedicated container (`ai-net`).
Using `network_mode: container:ai-net` allows all services to talk over `localhost` without exposing unnecessary ports to the host.

---

## 📦 Services

### **1. ai-net** (`ai-net.yaml`)

* **Purpose**: Parent network container for the stack.
* **Image**: `alpine:latest`
* **Notes**:

  * **Fixed container name**: `ai-net` is required for other containers to join its stack.
  * Only this container exposes ports to the host.
* **Network**:

  * Connected to external `nat99_net` with static IP `192.168.99.91`.
* **Ports**:

  * `8188:8188` — Only exposed here.

---

### **2. vLLM** (`vllm.yaml`)

* **Purpose**: Runs [vLLM](https://github.com/vllm-project/vllm) as an OpenAI-compatible API server.
* **Image**: `vllm/vllm-openai`
* **Network**: Shares `ai-net` network namespace.
* **Command Example**:

  ```bash
  --model HuggingFaceUser/ModelName --host 0.0.0.0 --port 8000
  ```
* **GPU Support**: All NVIDIA GPUs available.
* **Persistence**: `huggingface_cache` stores model cache.

---

### **3. ComfyUI** (`comfyui.yaml`)

* **Purpose**: [ComfyUI](https://github.com/comfyanonymous/ComfyUI) for AI image workflows.
* **Image**: `yanwk/comfyui-boot:cu126-slim`
* **Network**: Shares `ai-net` network namespace.
* **Persistence**: `comfyui` volume.
* **GPU Support**: All NVIDIA GPUs available.

---

### **4. llama.cpp** (`llamacpp.yaml`)

* **Purpose**: Runs [llama.cpp](https://github.com/ggerganov/llama.cpp) in server mode.
* **Image**: `ghcr.io/ggerganov/llama.cpp:server-cuda-cu121`
* **Network**: Shares `ai-net` network namespace.
* **Command Example**:

  ```bash
  --model /models/your-model.gguf --host 0.0.0.0 --port 8081 --n-gpu-layers -1
  ```
* **Persistence**: `model_storage` stores models.
* **GPU Support**: All NVIDIA GPUs available.

---

### **5. Ollama (CPU)** (`ollama-cpu.yaml`)

* **Purpose**: CPU-only [Ollama](https://ollama.com) instance.
* **Image**: `ollama/ollama`
* **Network**: Shares `ai-net` network namespace.
* **Environment Variables**:

  * `CUDA_VISIBLE_DEVICES=` — disables NVIDIA GPU usage.
  * `ROCM_VISIBLE_DEVICES=` — disables AMD GPU usage.
  * `OLLAMA_HOST=0.0.0.0:11435` — listens on alternate port.
* **Persistence**: `ollama_cpu_data`.

---

### **6. Ollama (GPU)** (`ollama.yaml`)

* **Purpose**: GPU-enabled Ollama instance.
* **Image**: `ollama/ollama`
* **Network**: Shares `ai-net` network namespace.
* **Persistence**: `ollama_gpu_data`.
* **GPU Support**: All NVIDIA GPUs available.

---

### **7. Open WebUI** (`openwebui.yaml`)

* **Purpose**: Web interface for interacting with LLMs.
* **Image**: `ghcr.io/open-webui/open-webui:main`
* **Network**: Shares `ai-net` network namespace.
* **Environment Variables**:

  * `PORT=8082`
  * `OLLAMA_BASE_URL=http://localhost:11434` — connects to Ollama via shared localhost.
* **Persistence**: `openwebui_data`.

---

## 🌐 Networking

* **`ai-net`** is the **network hub**.
* Containers with:

  ```yaml
  network_mode: "container:ai-net"
  ```

  share **the same IP and localhost** as `ai-net`.
* `nat99_net` is an **external bridge network** for static IPs and host access.

---

## 🚀 Deployment

### 1. Create the external network

```bash
docker network create \
  --driver=bridge \
  --subnet=192.168.99.0/24 \
  nat99_net
```

### 2. Start `ai-net` first

```bash
docker compose -f ai-net.yaml up -d
```

### 3. Start the other services

```bash
docker compose -f vllm.yaml up -d
docker compose -f openwebui.yaml up -d
docker compose -f ollama.yaml up -d
docker compose -f ollama-cpu.yaml up -d
docker compose -f llamacpp.yaml up -d
docker compose -f comfyui.yaml up -d
```

---

## 📁 Volumes

| Volume Name        | Purpose                      |
| ------------------ | ---------------------------- |
| huggingface\_cache | HuggingFace model cache      |
| comfyui            | ComfyUI workflows & settings |
| model\_storage     | llama.cpp models             |
| ollama\_cpu\_data  | Ollama CPU data              |
| ollama\_gpu\_data  | Ollama GPU data              |
| openwebui\_data    | Open WebUI data              |

---

## 🖥 GPU Requirements

For GPU-enabled containers:

1. Install **NVIDIA GPU drivers**.
2. Install [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html).
3. Test:

   ```bash
   docker run --rm --gpus all nvidia/cuda:12.2.0-base nvidia-smi
   ```

---

## 📌 Notes

* Only `ai-net` exposes ports to the host; all others use `localhost` internally.
* Fixed container names are **mandatory** when using `network_mode: container:ai-net`.
* Services can talk to each other via `localhost` when sharing `ai-net`.

