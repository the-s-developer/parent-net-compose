
# AI Network Docker Stack

This stack runs multiple AI services that **share the same network namespace** via a special container called **`ai-net`**.
This pattern is a **parent network** design in Docker.

## 🔍 How the Parent Network Works

* **`ai-net` is the parent network container.**
  Other services join its network namespace using:

  ```yaml
  network_mode: "container:ai-net"
  ```
* **Same IP & localhost for all**
  All services share the **same IP** as `ai-net` and talk to each other via `localhost`:

  * `openwebui` → `http://localhost:11434` (Ollama)
  * `vllm` → `http://localhost:8000`
* **Port exposure only on `ai-net`**
  If the host should access a service, map ports **only in `ai-net.yaml`**.
* **Static IP via `nat99_net`**
  `ai-net` has a fixed IP on an external Docker network for predictable access.

```
[ Host ] ── ports mapped ─▶ [ ai-net (parent) 192.168.99.91 ]
                                  │ (shared localhost/IP)
             vLLM  Ollama  Ollama-CPU  OpenWebUI  llama.cpp  ComfyUI
```

---

## 📦 Services

### **ai-net** (`ai-net.yaml`)

* Parent network container; fixed name `ai-net` (required).
* Connected to external `nat99_net` with static IP `192.168.99.91`.
* Only here you expose host ports (e.g., `8188:8188`).

### **vLLM** (`vllm.yaml`)

* OpenAI-compatible API via `vllm/vllm-openai`.
* Shares `ai-net` network.
* Example command:

  ```
  --model HuggingFaceUser/ModelName --host 0.0.0.0 --port 8000
  ```
* GPU enabled; cache on `huggingface_cache`.

### **ComfyUI** (`comfyui.yaml`)

* `yanwk/comfyui-boot:cu126-slim`.
* Shares `ai-net` network.
* GPU enabled; data on `comfyui`.

### **llama.cpp** (`llamacpp.yaml`)

* `ghcr.io/ggerganov/llama.cpp:server-cuda-cu121`.
* Shares `ai-net` network.
* Example command:

  ```
  --model /models/your-model.gguf --host 0.0.0.0 --port 8081 --n-gpu-layers -1
  ```
* Models on `model_storage`; GPU enabled.

### **Ollama (CPU)** (`ollama-cpu.yaml`)

* `ollama/ollama` (CPU-only).
* Shares `ai-net` network.
* Env:

  * `CUDA_VISIBLE_DEVICES=`
  * `ROCM_VISIBLE_DEVICES=`
  * `OLLAMA_HOST=0.0.0.0:11435`
* Data on `ollama_cpu_data`.

### **Ollama (GPU)** (`ollama.yaml`)

* `ollama/ollama` (GPU).
* Shares `ai-net` network.
* Data on `ollama_gpu_data`.

### **Open WebUI** (`openwebui.yaml`)

* `ghcr.io/open-webui/open-webui:main`.
* Shares `ai-net` network.
* Env:

  * `PORT=8082`
  * `OLLAMA_BASE_URL=http://localhost:11434`
* Data on `openwebui_data`.

---

## 🌐 Networking

* `network_mode: "container:ai-net"` → services **share localhost & IP** with `ai-net`.
* `nat99_net` is **external** and must exist before starting `ai-net`.
* You can run `nat99_net` as **bridge** on a non-conflicting subnet, or as **macvlan** if you want containers to appear as real hosts on your LAN.

---

## 🚀 Deployment

### 1) Create the external network

#### Option A — Bridge (use a subnet that does **not** clash with your host)

```bash
docker network create --driver=bridge --subnet=192.168.77.0/24 nat99_net
```

> If you previously used `192.168.99.0/24` and your host is also on that LAN, switch to a different subnet (e.g., `192.168.77.0/24`) to avoid conflicts.

#### Option B — Macvlan (containers get real IPs on your LAN)

Replace `enp7s0f3u2` with your physical NIC if different:

```bash
docker network create -d macvlan \
  --subnet=192.168.99.0/24 --gateway=192.168.99.1 \
  -o parent=enp7s0f3u2 nat99_net
```

> If you choose **macvlan**, also configure a **host macvlan interface** (next section) so the host can talk to containers directly.

---

### 2) Start `ai-net` first

```bash
docker compose -f ai-net.yaml up -d
```

### 3) Start the other services

```bash
docker compose -f vllm.yaml up -d
docker compose -f openwebui.yaml up -d
docker compose -f ollama.yaml up -d
docker compose -f ollama-cpu.yaml up -d
docker compose -f llamacpp.yaml up -d
docker compose -f comfyui.yaml up -d
```

---

## 🖥 Make Host → Macvlan Access **Persistent** (Survives Reboots)

When using **macvlan**, the commands below create a host-side interface so the host can reach containers on the same LAN.
Runtime commands (ip link/addr) **do not persist** across reboots. Pick one method:

### Method A — systemd unit (recommended)

Create a unit:

```bash
sudo tee /etc/systemd/system/macvlan0.service >/dev/null <<'EOF'
[Unit]
Description=Create macvlan0 for Docker nat99_net
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/sbin/ip link add macvlan0 link enp7s0f3u2 type macvlan mode bridge
ExecStart=/sbin/ip addr add 192.168.99.200/24 dev macvlan0
ExecStart=/sbin/ip link set macvlan0 up
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now macvlan0.service
```

* Change `enp7s0f3u2` and `192.168.99.200/24` to match your NIC and a **free IP** on your LAN.

### Method B — Netplan (Ubuntu 18.04+)

```bash
sudo tee /etc/netplan/99-macvlan.yaml >/dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp7s0f3u2:
      dhcp4: no
  macvlans:
    macvlan0:
      mode: bridge
      link: enp7s0f3u2
      addresses: [192.168.99.200/24]
EOF

sudo netplan apply
```

### Method C — `/etc/network/interfaces` (Debian/Ubuntu legacy)

```bash
sudo tee -a /etc/network/interfaces >/dev/null <<'EOF'
auto macvlan0
iface macvlan0 inet static
    pre-up ip link add macvlan0 link enp7s0f3u2 type macvlan mode bridge
    address 192.168.99.200
    netmask 255.255.255.0
EOF

sudo ifup macvlan0
```

**Test after reboot:**

```bash
ping -c1 192.168.99.91          # ai-net
nc -vz 192.168.99.91 8188       # ai-net mapped port
```

> ⚠️ Ensure `192.168.99.200` is unused. Adjust for your LAN.
> If you used **bridge** (not macvlan), you **don’t** need a host macvlan interface—just use port mappings on `ai-net`.

---

## 📁 Volumes

| Volume Name        | Purpose                 |
| ------------------ | ----------------------- |
| huggingface\_cache | HuggingFace model cache |
| comfyui            | ComfyUI data            |
| model\_storage     | llama.cpp models        |
| ollama\_cpu\_data  | Ollama CPU data         |
| ollama\_gpu\_data  | Ollama GPU data         |
| openwebui\_data    | Open WebUI data         |

---

## 🖥 GPU Requirements

1. Install NVIDIA drivers on the host.
2. Install NVIDIA Container Toolkit.
3. Verify:

   ```bash
   docker run --rm --gpus all nvidia/cuda:12.2.0-base nvidia-smi
   ```

---

## 🧩 Tips & Troubleshooting

* **Subnet conflict**: Don’t use the same subnet for Docker bridge as your host NIC unless using macvlan.
* **Orphans warning**: Use `--remove-orphans` if you renamed/removed services.
* **NVIDIA errors (NVML mismatch)**: Align host driver and container toolkit versions; test with `nvidia-smi` inside `nvidia/cuda` image.

