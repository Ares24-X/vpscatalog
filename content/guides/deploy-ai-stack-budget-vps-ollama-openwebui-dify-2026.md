---
title: "Deploy a Complete AI Stack on a Budget VPS: Ollama + Open WebUI + Dify (2026 Guide)"
description: "Run Ollama, Open WebUI, Dify, and n8n together on a single $10–$25/month VPS. Complete Docker Compose setup, GPU-less LLM performance tips, resource allocation, and cost breakdown for self-hosting your entire AI development environment."
date: 2026-07-30
tags: ["ai", "ollama", "open-webui", "dify", "n8n", "vps", "self-hosted", "docker", "llm", "budget"]
---

One-sentence verdict: You can run Ollama (local LLMs), Open WebUI (ChatGPT-style interface), Dify (AI app builder), and n8n (AI-powered workflows) on a single $10–$25/month VPS — with careful resource planning and the right hardware choices.

## Who This Guide Is For

- Developers who want a private AI sandbox without paying per-token API fees for every experiment.
- Freelancers and small teams who need AI-powered internal tools (chat, RAG, workflows) without enterprise SaaS pricing.
- Self-hosters who already run Docker on a VPS and want to add AI capabilities.
- Indie makers building AI prototypes who cannot justify $200+/month in cloud AI platform fees.
- Students and learners exploring LLMs, vector databases, and AI agent frameworks on a real server.

If you only need occasional ChatGPT access, just use the API or web app. This guide is for people who want a persistent, private, multi-tool AI environment they control.

## What This Stack Includes

| Component | What it does | Why you need it |
|---|---|---|
| **Ollama** | Runs local LLMs (Llama 3.1, Mistral, Phi-3, Qwen 2.5) | Free inference — no API bills |
| **Open WebUI** | ChatGPT-like chat interface | Your team's private AI chat portal |
| **Dify** | Visual AI app builder (RAG pipelines, agents, chatbots) | Build AI apps without coding |
| **n8n** | Workflow automation with AI nodes | Connect AI to Slack, email, APIs |
| **Qdrant** | Vector database (for Dify RAG) | Semantic search for your documents |

All running inside Docker containers on a single VPS, managed with Docker Compose.

## VPS Requirements

### Minimum vs. Recommended Specs

| Tier | vCPU | RAM | Storage | Monthly Cost | Can run |
|---|---|---|---|---|---|
| **Minimum (testing only)** | 4 vCPU | 8 GB | 60 GB SSD | $10–$15/mo | Ollama (3B models) + Open WebUI + Dify (light use) |
| **Recommended** | 6 vCPU | 16 GB | 100 GB NVMe | $20–$30/mo | Ollama (8B models) + all tools comfortably |
| **Production** | 8 vCPU | 32 GB | 200 GB NVMe | $40–$60/mo | Ollama (14B models) + team usage |

**Critical insight:** Local LLMs (Ollama) eat RAM. The table above accounts for ~4 GB for the OS and Docker, 6–8 GB for Ollama models, and the rest for Dify, Qdrant, n8n, and your apps.

### GPU vs. CPU-Only

Most budget VPS providers give you CPU-only instances. That's fine. Here's what to expect:

- **8B models (Llama 3.1 8B, Mistral 7B):** 5–15 tokens/second on 4+ vCPU — usable for chat
- **3B models (Phi-3 Mini, Llama 3.2 3B):** 15–30 tokens/second — feels snappy
- **14B+ models:** Borderline unusable without GPU — expect 2–5 tokens/second

If you need GPU, look at RunPod, Vast.ai, or Lambda Labs for dedicated GPU instances. They cost more but make larger models practical.

### Recommended VPS Providers for This Stack

| Provider | Best plan for this | RAM | vCPU | Storage | Price/mo | Notes |
|---|---|---|---|---|---|---|
| **Hetzner CX32** | CPX31 | 16 GB | 8 | 160 GB NVMe | ~$18 | Best value in Europe; excellent CPU |
| **Netcup RS 2000** | RS 2000 G11 | 16 GB | 6 | 320 GB SSD | ~$9 (on sale) | Ridiculous value if you catch a deal |
| **Contabo VPS L** | Cloud VPS L | 30 GB | 8 | 800 GB SSD | ~$15 | Lots of RAM/storage but shared CPU |
| **RackNerd** | 4 GB KVM (holiday) | 4 GB | 3 | 80 GB SSD | ~$5–$7 | Good for testing 3B models only |
| **DigitalOcean** | CPU-Optimized 8 GB | 8 GB | 4 | 100 GB NVMe | $84/mo | Overpriced for this use case |
| **Vultr** | High Frequency 8 GB | 8 GB | 4 | 160 GB NVMe | $48/mo | Better than DO but still pricey |
| **Hetzner CCX33** | Dedicated vCPU | 16 GB | 8 | 160 GB NVMe | ~$27 | Dedicated cores, no noisy neighbors |

**Our pick:** Hetzner CPX31 or CCX33 for Europe/global, Netcup RS series for budget hunters.

## Risk Warnings — Read Before You Proceed

⚠️ **These are real risks, not boilerplate:**

1. **RAM exhaustion is the #1 failure mode.** When Ollama loads a model into memory, there's no guardrail. If total RAM is exceeded, the kernel OOM killer picks a victim — usually your database. Always leave 2+ GB free.

2. **CPU-only inference is slow.** If you've only used ChatGPT, you'll be disappointed by the speed of local LLMs. Set expectations: this is for experimentation and internal tools, not production customer-facing chat.

3. **Storage fills up fast.** A single 8B model is ~5 GB. Dify's PostgreSQL and Qdrant grow with usage. Plan for at least 100 GB if you want to try multiple models.

4. **No SLA on self-hosted AI.** If your VPS goes down, your AI tools go down. For critical workflows, keep a fallback to OpenAI/Anthropic APIs.

5. **Security: these tools expose APIs.** Open WebUI and Dify run web servers. Lock them behind authentication and a reverse proxy (we cover this). Do not expose them directly to the internet without auth.

6. **Dify can be resource-hungry.** Its full stack includes PostgreSQL, Redis, Weaviate/Qdrant, a web server, a worker, and a sandbox. That's 6+ containers. Use `docker compose --profile` to disable pieces you don't need.

## Step 1: Provision Your VPS

1. Sign up with your chosen provider (we recommend Hetzner for this stack)
2. Create a VPS with at least 8 GB RAM, 4 vCPU, 80 GB SSD
3. Choose Ubuntu 24.04 LTS
4. Set up SSH key authentication
5. Note your server IP — you'll need it throughout

```bash
ssh root@your-server-ip
# Create a non-root user with sudo
adduser deploy
usermod -aG sudo deploy
su - deploy
```

## Step 2: Install Docker and Docker Compose

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker  # or log out and back in

# Verify
docker --version
docker compose version
```

## Step 3: Set Up the AI Stack Directory Structure

```bash
mkdir -p ~/ai-stack && cd ~/ai-stack
mkdir -p data/open-webui data/dify data/dify/postgres data/dify/redis data/dify/weaviate data/n8n data/qdrant
```

## Step 4: Create the Docker Compose File

This is the master configuration that ties all four tools together:

```yaml
# ~/ai-stack/docker-compose.yml
version: '3.8'

services:
  # ─── Ollama: Local LLM Inference ───
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    volumes:
      - ./data/ollama:/root/.ollama
    ports:
      - "11434:11434"
    environment:
      - OLLAMA_KEEP_ALIVE=24h
      - OLLAMA_HOST=0.0.0.0
    deploy:
      resources:
        limits:
          memory: 12G
    command: serve

  # ─── Open WebUI: ChatGPT-Style Interface ───
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    depends_on:
      - ollama
    volumes:
      - ./data/open-webui:/app/backend/data
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY:-change-me-to-random}
    deploy:
      resources:
        limits:
          memory: 2G

  # ─── Dify: AI App Builder ───
  # API service
  dify-api:
    image: langgenius/dify-api:latest
    container_name: dify-api
    restart: unless-stopped
    depends_on:
      dify-postgres:
        condition: service_healthy
      dify-redis:
        condition: service_healthy
    volumes:
      - ./data/dify/storage:/app/api/storage
    ports:
      - "5001:5001"
    environment:
      - MODE=api
      - LOG_LEVEL=INFO
      - SECRET_KEY=${DIFY_SECRET_KEY:-change-me-randomly}
      - DB_USERNAME=postgres
      - DB_PASSWORD=${DIFY_DB_PASSWORD:-difyai123456}
      - DB_HOST=dify-postgres
      - DB_PORT=5432
      - DB_DATABASE=dify
      - REDIS_HOST=dify-redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${DIFY_REDIS_PASSWORD:-}
      - STORAGE_TYPE=local
      - STORAGE_LOCAL_PATH=/app/api/storage
      - VECTOR_STORE=qdrant
      - QDRANT_URL=http://qdrant:6333
    deploy:
      resources:
        limits:
          memory: 1G

  # Dify Worker (background tasks)
  dify-worker:
    image: langgenius/dify-api:latest
    container_name: dify-worker
    restart: unless-stopped
    depends_on:
      dify-postgres:
        condition: service_healthy
      dify-redis:
        condition: service_healthy
    volumes:
      - ./data/dify/storage:/app/api/storage
    environment:
      - MODE=worker
      - LOG_LEVEL=INFO
      - SECRET_KEY=${DIFY_SECRET_KEY:-change-me-randomly}
      - DB_USERNAME=postgres
      - DB_PASSWORD=${DIFY_DB_PASSWORD:-difyai123456}
      - DB_HOST=dify-postgres
      - DB_PORT=5432
      - DB_DATABASE=dify
      - REDIS_HOST=dify-redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${DIFY_REDIS_PASSWORD:-}
      - STORAGE_TYPE=local
      - STORAGE_LOCAL_PATH=/app/api/storage
    deploy:
      resources:
        limits:
          memory: 1G

  # Dify Web frontend
  dify-web:
    image: langgenius/dify-web:latest
    container_name: dify-web
    restart: unless-stopped
    depends_on:
      - dify-api
    ports:
      - "3001:3000"
    environment:
      - CONSOLE_API_URL=http://dify-api:5001
      - APP_API_URL=http://dify-api:5001
    deploy:
      resources:
        limits:
          memory: 512M

  # Dify PostgreSQL
  dify-postgres:
    image: postgres:15-alpine
    container_name: dify-postgres
    restart: unless-stopped
    volumes:
      - ./data/dify/postgres:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=${DIFY_DB_PASSWORD:-difyai123456}
      - POSTGRES_DB=dify
      - POSTGRES_USER=postgres
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres", "-d", "dify"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 512M

  # Dify Redis
  dify-redis:
    image: redis:7-alpine
    container_name: dify-redis
    restart: unless-stopped
    volumes:
      - ./data/dify/redis:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 256M

  # ─── Qdrant: Vector Database (shared) ───
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    restart: unless-stopped
    volumes:
      - ./data/qdrant:/qdrant/storage
    ports:
      - "6333:6333"
    deploy:
      resources:
        limits:
          memory: 1G

  # ─── n8n: AI Workflow Automation ───
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    volumes:
      - ./data/n8n:/home/node/.n8n
    ports:
      - "5678:5678"
    environment:
      - N8N_SECURE_COOKIE=false
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY:-change-me-randomly}
      - N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=true
    deploy:
      resources:
        limits:
          memory: 1G

  # ─── Caddy: Reverse Proxy with Auto SSL ───
  caddy:
    image: caddy:2-alpine
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./data/caddy/data:/data
      - ./data/caddy/config:/config
    deploy:
      resources:
        limits:
          memory: 128M
```

## Step 5: Create the Caddyfile for Reverse Proxy

```caddy
# ~/ai-stack/Caddyfile
ai.yourdomain.com {
    reverse_proxy open-webui:8080
}

dify.yourdomain.com {
    reverse_proxy dify-web:3000
}

n8n.yourdomain.com {
    reverse_proxy n8n:5678
}
```

Replace `yourdomain.com` with your actual domain. Point the DNS A/AAAA records for each subdomain to your VPS IP.

## Step 6: Create the Environment File

```bash
# ~/ai-stack/.env
WEBUI_SECRET_KEY=$(openssl rand -hex 32)
DIFY_SECRET_KEY=$(openssl rand -hex 32)
DIFY_DB_PASSWORD=$(openssl rand -hex 16)
N8N_ENCRYPTION_KEY=$(openssl rand -hex 16)
```

## Step 7: Launch the Stack

```bash
cd ~/ai-stack

# Start everything
docker compose up -d

# Check status
docker compose ps

# Watch logs
docker compose logs -f
```

Wait 2–3 minutes for all services to initialize. Check health:

```bash
# Verify Ollama
curl http://localhost:11434/api/tags

# Verify Open WebUI
curl http://localhost:3000  # should return HTML

# Verify Dify
curl http://localhost:3001  # should redirect to setup

# Verify n8n
curl http://localhost:5678/healthz
```

## Step 8: Pull Your First LLM Model

```bash
# Pull Llama 3.1 8B (good balance of quality and speed)
docker exec -it ollama ollama pull llama3.1:8b

# Or pull a smaller, faster model
docker exec -it ollama ollama pull phi3:mini  # 3.8B, very fast

# Test inference
docker exec -it ollama ollama run llama3.1:8b "Explain quantum computing in one paragraph"
```

## Step 9: Configure Each Tool

### Open WebUI

1. Visit `https://ai.yourdomain.com`
2. Create an admin account (first user becomes admin)
3. Go to Settings → Connections → Ollama should auto-detect at `http://ollama:11434`
4. Start chatting — your model will be available in the dropdown

### Dify

1. Visit `https://dify.yourdomain.com`
2. Create an admin account
3. Configure the LLM provider: Settings → Model Provider → add Ollama at `http://ollama:11434`
4. Create your first AI app: a chatbot, a RAG knowledge base, or a workflow

### n8n

1. Visit `https://n8n.yourdomain.com`
2. Create an admin account
3. Install the AI community nodes: Settings → Community Nodes → install `n8n-nodes-ollama`
4. Create a workflow: HTTP trigger → Ollama chat node → respond

## Practical Use Cases

### Use Case 1: Private AI Chat for Your Team

**Set up:** Open WebUI with Llama 3.1 8B. Create user accounts for team members. No data leaves your server. No per-user subscription fees.

### Use Case 2: RAG Knowledge Base from Company Docs

**Set up:** Upload your PDFs and Markdown files to Dify → auto-chunks them → stores vectors in Qdrant → build a chatbot that answers questions from your docs. Connect to Open WebUI or embed on your intranet.

### Use Case 3: AI Slack Bot for Internal Q&A

**Set up:** n8n workflow → Slack webhook trigger → Ollama node for LLM reasoning → Dify for RAG lookup → Slack response. Zero monthly AI API costs.

### Use Case 4: AI-Powered Email Classifier

**Set up:** n8n → Gmail trigger → Ollama for classification → route to different Slack channels or Notion databases based on content.

## Cost Breakdown

| Item | Monthly cost | Notes |
|---|---|---|
| VPS (Hetzner CPX31) | ~$18 | 16 GB RAM, 8 vCPU, 160 GB NVMe |
| Domain | ~$1 | .com domain via Cloudflare |
| Ollama inference | **$0** | Unlimited, CPU-limited |
| Open WebUI | **$0** | Self-hosted, open source |
| Dify | **$0** | Self-hosted community edition |
| n8n | **$0** | Self-hosted community edition |
| **Total** | **~$19/month** | Complete private AI platform |

**Comparison:** A similar setup using paid APIs (OpenAI GPT-4o, Anthropic Claude, Pinecone) for team use:

| Service | Equivalent | Monthly cost |
|---|---|---|
| ChatGPT Team | Open WebUI | $25/user × 3 = $75 |
| Dify Cloud | Dify self-hosted | $59+ (Pro plan) |
| n8n Cloud | n8n self-hosted | $20+ (Starter plan) |
| Pinecone | Qdrant | $70+ |
| **Total paid SaaS** | | **~$224/month** |

**Savings: $205/month** — for the cost of roughly 3 ChatGPT Team seats, you get an entire AI platform.

## Performance Tuning Tips

### 1. Limit Ollama Memory Usage

```bash
# In docker-compose.yml, set a hard memory limit
ollama:
  deploy:
    resources:
      limits:
        memory: 8G  # Adjust based on your total RAM
```

### 2. Use Smaller Models for Faster Responses

```bash
# Speed comparison on 8 vCPU Hetzner:
# phi3:mini (3.8B)  → ~25 tok/s  — snappy
# llama3.2:3b        → ~20 tok/s  — fast
# mistral:7b         → ~12 tok/s  — decent
# llama3.1:8b        → ~8 tok/s   — usable
# qwen2.5:14b        → ~3 tok/s   — slow
```

### 3. Disable Unused Dify Services

If you only use Dify's workflow builder (not the chat/web):

```yaml
# Start only what you need
docker compose up -d ollama open-webui n8n qdrant
# Omit dify-* services entirely
```

### 4. Set Up Swap Space

```bash
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Swap won't replace RAM for LLM inference (it'll be unusably slow), but it prevents the OOM killer from terminating your database processes.

### 5. Schedule Automatic Model Unloading

```bash
# Add a cron job to release idle model memory
crontab -e
# Add: 0 */4 * * * curl -s http://localhost:11434/api/generate -d '{"model":"","keep_alive":0}' > /dev/null
```

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| Ollama can't pull models | Insufficient disk space | `df -h` — you need 2× model size free |
| Open WebUI shows "connection refused" | Ollama not ready | Wait 30s after `docker compose up -d` |
| Dify setup page loops | Database not initialized | `docker compose restart dify-api dify-worker` |
| Models are extremely slow | CPU throttling | Check provider's fair-use policy; upgrade vCPU count |
| OOM killer terminates containers | RAM exhausted | Reduce Ollama memory limit; use smaller models |
| SSL certificate errors | DNS not propagated | Wait 5–10 minutes; Caddy retries automatically |
| n8n can't reach Ollama | Docker network issue | Use service name `ollama` not `localhost` |

## Scaling Beyond One VPS

When you outgrow a single server:

1. **Separate Ollama to its own GPU instance** — keep LLM inference on a beefier server; everything else stays on the cheap VPS
2. **Use cloud LLM APIs as fallback** — configure Open WebUI and Dify to use OpenAI/Anthropic when Ollama is too slow
3. **Add a dedicated database server** — move PostgreSQL and Qdrant to a managed service (Supabase, Qdrant Cloud)
4. **Use Coolify or Dokploy** — if you're provisioning this for multiple clients, use a PaaS wrapper

## Verdict: Should You Build This?

**Yes, if:**

- You want to experiment with AI tools without monthly API bills
- Your team needs a private, on-premises AI workspace
- You're comfortable with Docker and basic Linux administration
- You understand the speed limitations of CPU-only inference

**No, if:**

- You need production-grade speed and reliability — use managed AI APIs
- You want zero maintenance — use ChatGPT Team or Dify Cloud
- Your team has no Docker/Linux experience — the learning curve is real
- You need 70B+ parameter model performance — GPU instances are expensive

**Bottom line:** For ~$19/month, this stack gives you a complete private AI development platform. The trade-off is CPU-only inference speed and the responsibility of self-hosting. For learning, prototyping, and internal tools, it's an incredible value.

---

*Last updated: July 30, 2026. Pricing reflects current market rates and may change.*
