# InferX | llm-stack — Open WebUI + vLLM + Nginx (SSL)
A self-hosted, GPU-accelerated LLM inference stack running behind HTTPS. This setup gives you a ChatGPT-like web interface powered by your own models via vLLM, secured with SSL through Nginx, and backed by PostgreSQL for persistent storage.

**"Infer" (LLM inference) + "X" (scalable/extensible)**

## Architecture
```bash

                         ┌──────────────────────────────────────────────┐
                         │              Docker Network                  │
 User (Browser)          │                                              │
      │                  │   ┌────────┐       ┌──────────────┐          │
      │  HTTPS :443      │   │ Nginx  │──────▶│  Open WebUI  │         │
      └── ──────────────▶│   │ (SSL)  │       │   (:8080)    │         │
                         │   │        │       └──────┬───────┘          │
      HTTP :80 ──301────▶│   │        │              │                 │
                         │   │        │       ┌──────▼───────┐          │
                         │   │        │       │  PostgreSQL  │          │
                         │   │        │       │    (:5432)   │          │
                         │   │        │       └──────────────┘          │
                         │   │        │                                 │
                         │   │        │──/v1/─▶┌─────────────┐         │
                         │   │        │        │    vLLM      │         │
                         │   └────────┘        │  (:8000)     │         │
                         │                     │  [GPU]       │         │
                         │                     └─────────────┘          │
                         │                            ▲                 │
                         │                            │ (optional)      │
                         │                     ┌─────────────┐          │
                         │                     │  Server 2    │         │
                         │                     │  vLLM Worker │         │
                         │                     │  (Remote GPU)│         │
                         │                     └─────────────┘          │
                         └──────────────────────────────────────────────┘

```

## 📁 Project Structure
```bash
project-root/
├── docker-compose.yml        # Main stack (Postgres + Open WebUI + vLLM + Nginx)
├── nginx.conf                # Nginx reverse proxy config (SSL + routing)
├── .env                      # Environment variables (passwords, tokens, model)
├── certs/                    # SSL certificate files
│   ├── fullchain.pem         # SSL certificate (replace the dummy file)
│   └── privkey.pem           # SSL private key  (replace the dummy file)
├── server_2/                 # Optional: standalone vLLM worker on a second GPU server
│   ├── docker-compose.yml
│   └── README.md
└── README.md 

```

## ✅ Prerequisites
- Docker & Docker Compose (v2+)
- NVIDIA GPU with drivers installed
- NVIDIA Container Toolkit (nvidia-docker2 or nvidia-container-toolkit)
- SSL Certificate files (fullchain.pem and privkey.pem)
- Hugging Face account with an access token (for gated models like Llama, Mistral, etc.)

## ⚙️ Setup Instructions

### Step 1 — SSL Certificates
The certs/ folder contains dummy placeholder files. You must replace them with your real SSL certificate files.

```bash
# Delete the dummy files
rm certs/fullchain.pem certs/privkey.pem

# Copy your real SSL files into the certs folder
# IMPORTANT: File names MUST match exactly — fullchain.pem and privkey.pem
cp /path/to/your/fullchain.pem certs/fullchain.pem
cp /path/to/your/privkey.pem   certs/privkey.pem
```
- `Note :` If you don't have SSL certs and are testing locally, you can generate self-signed certs:
    ```bash
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout certs/privkey.pem \
    -out certs/fullchain.pem \
    -subj "/CN=localhost"
    ```
### Step 2 — Environment Variables
Open the `.env` file and fill in your values:
```bash
# PostgreSQL password (choose any secure password)
POSTGRES_PASSWORD=your_secure_password_here

# The Hugging Face model to serve (e.g., meta-llama/Llama-3.1-8B-Instruct)
MODEL_NAME=your_model_name_here

# Your Hugging Face access token (required for gated models)
HUGGING_FACE_HUB_TOKEN=hf_your_token_here
```
| Variable              | Description                          | Example                          |
|-----------------------|--------------------------------------|----------------------------------|
| POSTGRES_PASSWORD     | Password for the PostgreSQL database | MySecureP@ss123                  |
| MODEL_NAME            | Full Hugging Face model ID to serve  | meta-llama/Llama-3.1-8B-Instruct |
| HUGGING_FACE_HUB_TOKEN| API token from huggingface.co/settings/tokens | hf_abc123...                     |

## Step 3 — Validate Configuration

Before starting the stack, verify that all environment variables are correctly passed and the YAML is valid:
```bash
docker compose config
```
This will print the fully resolved `docker-compose.yml` with all `${VARIABLES}` replaced by their actual values. Review it carefully — if you see any `${...}` still present, your .env file is not being picked up.

### Step 4 — Launch the Stack
```bash
docker compose up -d
```
    ⏳ First-time startup will take significant time — vLLM needs to download the model weights from Hugging Face (can be several GB). Subsequent restarts will use the cached weights.

Monitor the logs to track progress:
```bash
# Watch all services
docker compose logs -f

# Watch only vLLM (model download progress)
docker compose logs -f vllm
```
### Step 5 — Access the Web UI

Open your browser and navigate to:
```bash
https://localhost:443
```
    Note: HTTP requests on port 80 are automatically redirected to HTTPS on port 443.

First-Time Login
- Open WebUI will prompt you to create an account on the first visit.
- The first account you create becomes the admin account.
- Use this admin ID and password for all subsequent logins.

    ⚠️ Remember your first-time credentials — this becomes the admin account and cannot be easily reset.

## 🔌 Adding More GPU Servers (Scaling Out)

You can distribute inference across multiple GPU machines using Nginx load balancing.

### On the Remote Server
- Navigate to the server_2/ folder and follow its README.
- Start the standalone vLLM worker on the remote machine.

### On the Main Server
Edit `nginx.conf` and uncomment/add the remote server in the upstream block:

```bash
upstream vllm_cluster {
    server vllm:8000;                    # Local vLLM (Server 1)
    server 192.168.1.100:8000;           # Remote vLLM (Server 2)
    # server 192.168.1.101:8000;         # Add more servers as needed
}
```
Then reload Nginx:
```bash
docker compose restart nginx
```
    For detailed instructions, see the server_2/README.md.

## 🛑 Stopping the Stack
```bash
# Stop all containers
docker compose down

# Stop and remove all data (model cache, database, uploads) — USE WITH CAUTION
docker compose down -v
```
## 🔧 Useful Commands

| Command                     | Description                                      |
|-----------------------------|--------------------------------------------------|
| docker compose config       | Validate YAML and check resolved environment variables |
| docker compose up -d        | Start all services in background                 |
| docker compose down         | Stop all services                                |
| docker compose logs -f      | Follow logs for all services                     |
| docker compose logs -f vllm | Follow vLLM logs only (model loading progress)   |
| docker compose ps           | Check status of all containers                   |
| docker compose restart nginx| Restart Nginx after config change                |

## 🐛 Troubleshooting

| Problem                          | Cause                                   | Solution                                                                 |
|----------------------------------|-----------------------------------------|--------------------------------------------------------------------------|
| Nginx won't start                | Invalid nginx.conf syntax                | Run `docker compose logs nginx` and check for config errors              |
| vLLM crashes on startup          | Not enough GPU VRAM for the model        | Try a smaller model or add `--max-model-len` to the vLLM command         |
| Can't access https://localhost   | SSL cert files missing or wrong names    | Ensure `certs/fullchain.pem` and `certs/privkey.pem` exist               |
| Model download stuck / fails     | Invalid or missing HF token              | Verify `HUGGING_FACE_HUB_TOKEN` in `.env`                                |
| "Connection refused" on port 443 | Containers still starting                | Wait and check `docker compose ps` — all should show running             |
| Open WebUI shows "No models"     | vLLM not fully loaded yet                | Wait for vLLM to finish loading (check `docker compose logs -f vllm`)    |
| WebSocket errors in browser      | Missing map block in nginx.conf          | Ensure the `map $http_upgrade` block exists in `nginx.conf`              |

## 📄 License
MIT License | If you have any issue feel free to contact at `shergillkuldeep@outlook.com`