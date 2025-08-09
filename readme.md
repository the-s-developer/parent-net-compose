# AI Network Docker Stack

This stack runs multiple AI services that **share the same network namespace** via a special container called **`ai-net`**.
This design is called a **parent network** setup in Docker.

---

## 🔍 How the Parent Network Works

* **`ai-net` is the “parent” network container.**
  Other containers don’t get their own IP — instead, they join `ai-net`’s network namespace using:

  ```yaml
  network_mode: "container:ai-net"
  ```

* **Same IP & localhost for all**
  Every service connected this way uses **the same IP** and can talk to each other through `localhost`.
  For example:

  * `openwebui` → Ollama at `http://localhost:11434`
  * `vllm` → available at `http://localhost:8000`

* **Ports are only exposed on `ai-net`**
  If you want the host (or the outside world) to access a service, you only map the port in `ai-net.yaml`.
  This keeps the stack clean and secure.

* **Static IP with `nat99_net`**
  `ai-net` is assigned a fixed IP (`192.168.99.91`) on an external Docker network, so the host can always reach it.

* **Simplified service communication**
  No need for Docker DNS names — all services see each other as `localhost`.

**Network topology:**

```
        [ Host Machine ]
               │
        ┌──────┴──────┐
        │  ai-net     │ 192.168.99.91 (parent)
        │ (Ports mapped to host)  
        └──────┬──────┘
               │
     ┌─────────┼──────────────────────────────────────────────┐
     │         │         │         │         │         │
[ vLLM ]  [ Ollama ]  [ Ollama-CPU ]  [ OpenWebUI ]  [ llama.cpp ]  [ ComfyUI ]
     share same localhost/IP via `network_mode: container:ai-net`
```

---

## 📦 Services

### **1. ai-net** (`ai-net.yaml`)

* **Purpose**: Parent network container for the stack.
* **Image**: `alpine:latest`
* **Fixed container name**: Required so others can join its network namespace.
* **Network**: Connected to `nat99_net` with static IP `192.168.99.91`.
* **Ports**: Only here are ports mapped to host, e.g. `8188:8188`.

---

### **2. vLLM** (`vllm.yaml`)

* **Purpose**: [vLLM](https://github.com/vllm-project/vllm) as OpenAI-compatible API.
* **Network**: Shares `ai-net`’s network stack.
* **Command Example**:

  ```bash
  --model HuggingFaceUser/ModelName --host 0.0.0.0 --port 8000
  ```
* **GPU**: All NVIDIA GPUs.
* **Persistence**: `huggingface_cache`.

---

### **3. ComfyUI** (`comfyui.yaml`)

* **Purpose**: [ComfyUI](https://github.com/comfyanonymous/ComfyUI) for AI image workflows.
* **Network**: Shares `ai-net`.
* **GPU**: All NVIDIA GPUs.
* **Persistence**: `comfyui`.

---

### **4. llama.cpp** (`llamacpp.yaml`)

* **Purpose**: [llama.cpp](https://github.com/ggerganov/llama.cpp) server.
* **Network**: Shares `ai-net`.
* **Command Example**:

  ```bash
  --model /models/your-model.gguf --host 0.0.0.0 --port 8081 --n-gpu-layers -1
  ```
* **GPU**: All NVIDIA GPUs.
* **Persistence**: `model_storage`.

---

### **5. Ollama (CPU)** (`ollama-cpu.yaml`)

* **Purpose**: CPU-only [Ollama](https://ollama.com).
* **Network**: Shares `ai-net`.
* **Environment**:

  * `CUDA_VISIBLE_DEVICES=` (disable GPU)
  * `ROCM_VISIBLE_DEVICES=` (disable AMD GPU)
  * `OLLAMA_HOST=0.0.0.0:11435` (alternate port)
* **Persistence**: `ollama_cpu_data`.

---

### **6. Ollama (GPU)** (`ollama.yaml`)

* **Purpose**: GPU-enabled Ollama.
* **Network**: Shares `ai-net`.
* **GPU**: All NVIDIA GPUs.
* **Persistence**: `ollama_gpu_data`.

---

### **7. Open WebUI** (`openwebui.yaml`)

* **Purpose**: Web interface for LLMs.
* **Network**: Shares `ai-net`.
* **Environment**:

  * `PORT=8082`
  * `OLLAMA_BASE_URL=http://localhost:11434`
* **Persistence**: `openwebui_data`.

---

## 🚀 Deployment

### 1. Create the external network

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

### 3. Start the rest

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

| Volume Name        | Purpose                 |
| ------------------ | ----------------------- |
| huggingface\_cache | HuggingFace model cache |
| comfyui            | ComfyUI data            |
| model\_storage     | llama.cpp models        |
| ollama\_cpu\_data  | Ollama CPU models       |
| ollama\_gpu\_data  | Ollama GPU models       |
| openwebui\_data    | Open WebUI data         |

---

## 🖥 GPU Requirements

1. Install NVIDIA drivers.
2. Install [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html).
3. Test:

   ```bash
   docker run --rm --gpus all nvidia/cuda:12.2.0-base nvidia-smi
   ```
