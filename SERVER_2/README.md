# 🖥️ Server 2 — Standalone vLLM Worker
This folder contains a standalone vLLM worker that can run on a separate GPU machine and be added to the main stack's Nginx load balancer. This allows you to scale out LLM inference across multiple servers.

## 📐 How It Fits Into the Architecture


```bash
 Main Server                          Remote Server (Server 2)

┌──────────────────┐                 ┌──────────────────────┐

│  Nginx           │                 │                      │

│  ┌────────────┐  │   HTTP :8000    │  ┌────────────────┐  │

│  │ upstream    │──┼────────────────▶│  │  vLLM Worker   │  │

│  │ vllm\_cluster│  │                 │  │  (GPU)         │  │

│  └─────┬──────┘  │                 │  └────────────────┘  │

│        │         │                 └──────────────────────┘

│        ▼         │

│  Local vLLM      │

│  (GPU)           │

└──────────────────┘
```
Nginx on the main server distributes /v1/ API requests across both the local vLLM instance and this remote worker.

## ✅ Prerequisites
- Docker \& Docker Compose (v2+)
- NVIDIA GPU with drivers installed
- NVIDIA Container Toolkit (nvidia-docker2 or nvidia-container-toolkit)
- Network connectivity to the main server (port 8000 must be accessible)
- Hugging Face access token (same as main server)

## ⚙️ Setup Instructions

**Step 1 — Copy Files to Remote Machine**

Copy this server_2/ folder to your remote GPU server:
```bash
scp -r server_2/ user@remote-server-ip:/path/to/server_2/
```
**Step 2 — Create the `.env` File**

On the remote server, create a .env file inside the server_2/ directory:
```bash
# Must be the SAME model as the main server
MODEL_NAME=your_model_name_here

# Your Hugging Face access token
HUGGING_FACE_HUB_TOKEN=hf_your_token_here
```

    ⚠️ Important: The `MODEL_NAME` must be identical to the one running on the main server. Nginx load-balances requests across all workers, so they must all serve the same model.
***Step 3 — Launch the Worker***
```bash
cd server_2/

docker compose up -d
```
    ⏳ First-time startup will take time to download model weights. Monitor progress:

```bash
docker compose logs -f
```
**Step 4 — Verify It's Running**
```bash
# Check container status
docker compose ps

# Test the API endpoint locally
curl http://localhost:8000/v1/models
```
You should see a JSON response listing the model name. If you see this, the worker is ready.

## 🔗 Connect to the Main Server

Once the worker is running, go back to the main server and register it in Nginx.

**Edit** `nginx.conf` on the Main Server

Find the upstream `vllm_cluster` block and add the remote server's IP:

```bash
upstream vllm_cluster {
    server vllm:8000;                    # Local vLLM (Server 1)
    server <SERVER_2_IP>:8000;           # Remote vLLM (Server 2) ← add this line
}
```
Replace `<SERVER_2_IP>` with the actual IP address of the remote machine (e.g., `192.168.1.100`).

**Restart Nginx on the Main Server**
```bash
docker compose restart nginx
```
**Verify Load Balancing**

From the main server, you can check that Nginx is routing to both backends:
```bash
# Should return responses from both servers
curl -k https://localhost/v1/models
```

## 📁 File Structure
```bash
server_2/
├── docker-compose.yml    # Standalone vLLM worker service
├── .env                  # Environment variables (create this)
└── README.md             # You are here
```
## 🔍 docker-compose.yml Breakdown
# ⚙️ Settings Reference

| Setting                        | Purpose                                                                 |
|--------------------------------|-------------------------------------------------------------------------|
| image: vllm/vllm-openai:latest | Official vLLM image with OpenAI-compatible API                          |
| runtime: nvidia                 | Enables GPU passthrough to the container                                |
| shm_size: '10.24gb'             | Shared memory for GPU operations (prevents OOM crashes)                 |
| ports: "8000:8000"              | Exposes vLLM API so Nginx on the main server can reach it               |
| HF_HUB_OFFLINE=1                | After first download, forces offline mode (uses cached weights)         |
| --trust-remote-code             | Required by some models that include custom Python code                 |
| volumes: ~/.cache/huggingface   | Persists downloaded model weights across container restarts             |

## 🛠️ Adding Even More Servers

You can repeat this process for Server 3, Server 4, etc.:
- **Copy** this folder to each new GPU machine.
- **Configure** .env with the same model name and HF token.
- **Start** the worker with docker compose up -d.
- **Register** the new server IP in nginx.conf on the main server:
```bash
upstream vllm_cluster {
    server vllm:8000;                    # Server 1 (local)
    server 192.168.1.100:8000;           # Server 2
    server 192.168.1.101:8000;           # Server 3
    server 192.168.1.102:8000;           # Server 4
}
```
- **Restart** Nginx: docker compose restart nginx
## 🐛 Troubleshooting
| Problem                        | Cause                          | Solution                                                             |
|--------------------------------|--------------------------------|----------------------------------------------------------------------|
| Container exits immediately    | GPU not detected                | Run `nvidia-smi` on the host to verify GPU drivers                   |
| Model download fails           | Invalid HF token                | Check `HUGGING_FACE_HUB_TOKEN` in `.env`                             |
| Main server can't reach worker | Port 8000 blocked by firewall   | Open port 8000: `sudo ufw allow 8000`                                |
| Nginx returns 502 Bad Gateway  | Worker not ready or crashed     | Check worker logs: `docker compose logs -f`                          |
| OOM errors during inference    | Model too large for GPU VRAM    | Try a smaller model or add `--max-model-len 4096` to the command     |
| Different responses from workers| Model mismatch                 | Ensure `MODEL_NAME` is identical on all servers                      

## 🛑 Stopping the Worker
```bash
# Stop the worker
docker compose down

# Stop and remove cached model weights — USE WITH CAUTION
docker compose down -v
```

## 📄 License
MIT, Feel free to contact if you face any issue at shergillkuldeep@outlook.com
