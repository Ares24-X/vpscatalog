---
title: "Deploy Open WebUI on a VPS: Self-Hosted ChatGPT Alternative (2026 Guide)"
description: "Complete guide to deploying Open WebUI on a budget VPS. Get a private ChatGPT-like interface connected to local LLMs or external APIs. Covers Docker setup, reverse proxy, authentication, model configuration, and cost breakdown."
date: 2026-07-19
tags: ["ai", "open-webui", "self-hosted", "vps", "docker", "chatgpt-alternative", "llm"]
---

One-sentence verdict: Open WebUI gives you a polished ChatGPT-style interface on your own server for $5–$20/month, connecting to either local models via Ollama or external APIs like OpenAI and Anthropic — with full data ownership.

## Who This Guide Is For

- Privacy-focused teams who want a shared AI chat interface without sending data to third-party servers.
- Developers already running Ollama on a VPS who want a proper web UI instead of curl commands.
- Small businesses that need a ChatGPT-like tool for staff but cannot justify $20/user/month for commercial plans.
- Hobbyists and tinkerers who want to experiment with multiple LLM backends from one dashboard.
- IT consultants deploying AI tools for clients who require self-hosted solutions for compliance reasons.

If you just need personal ChatGPT access for basic tasks and don't care about data sovereignty, this is overkill — just pay for a ChatGPT subscription.

## What Open WebUI Actually Is

Open WebUI (formerly Ollama WebUI) is a self-hosted web application that provides a ChatGPT-like interface. It supports multiple backends:

- **Ollama** — connect to locally running models (Llama 3, Mistral, Phi, etc.)
- **OpenAI-compatible APIs** — use GPT-4o, Claude, or any API that follows the OpenAI format
- **Multiple models simultaneously** — switch between local and cloud models in the same interface

Key features that matter in practice:

| Feature | What it means for you |
|---|---|
| Multi-user auth | Share one server with your team, each gets their own chat history |
| RAG (document upload) | Upload PDFs/docs and chat with them using local models |
| Model management | Pull and delete Ollama models from the UI |
| Prompt templates | Save and share system prompts across the team |
| Web search integration | Models can search the web for current information |
| API key management | Connect to OpenAI/Anthropic without sharing keys with end users |

## Hardware Requirements

Open WebUI itself is lightweight. The VPS sizing depends entirely on whether you run local models or just proxy to external APIs.

### Scenario A: API Proxy Only (cheapest)

You connect Open WebUI to OpenAI, Anthropic, or other cloud APIs. No local inference.

| Resource | Minimum | Recommended |
|---:|---:|---:|
| RAM | 1 GB | 2 GB |
| CPU | 1 vCPU | 2 vCPU |
| Disk | 10 GB | 20 GB |
| Monthly cost | $4–$6 | $6–$10 |

**Best providers for this:** Racknerd ($11/year deals), Contabo ($5/mo), Hetzner CX22 (€4/mo).

### Scenario B: Local Models via Ollama

You run Ollama alongside Open WebUI for fully private inference.

| Model target | RAM | CPU | Disk | Monthly cost |
|---|---:|---:|---:|---:|
| Small models (3B) | 4 GB | 2 vCPU | 30 GB | $6–$12 |
| Medium models (7–8B) | 8 GB | 4 vCPU | 50 GB | $12–$24 |
| Large models (13B+) | 16 GB | 6 vCPU | 80 GB | $24–$48 |

**Best providers for this:** Hetzner CPX31 (€15/mo for 8GB), Contabo VPS M (€10/mo for 16GB), DigitalOcean Premium 8GB ($56/mo but better CPU).

> **Warning:** Contabo offers the most RAM per dollar but has older CPUs with slower single-thread performance. For CPU-only LLM inference, single-thread speed matters. Hetzner AMD EPYC instances are noticeably faster at the same core count.

## Step-by-Step Deployment

### Prerequisites

- A VPS running Ubuntu 22.04 or 24.04 (Debian 12 also works)
- A domain name pointed to your server's IP (for HTTPS)
- SSH access with sudo privileges

### Step 1: Install Docker

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker using the official script
curl -fsSL https://get.docker.com | sh

# Add your user to docker group
sudo usermod -aG docker $USER

# Log out and back in, then verify
docker --version
```

### Step 2: Deploy Open WebUI with Docker Compose

Create a project directory:

```bash
mkdir -p ~/open-webui && cd ~/open-webui
```

Create `docker-compose.yml`:

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3000:8080"
    volumes:
      - open-webui-data:/app/backend/data
    environment:
      - OLLAMA_BASE_URL=http://host.docker.internal:11434
      - WEBUI_SECRET_KEY=change-this-to-a-random-string
      - ENABLE_SIGNUP=true
    extra_hosts:
      - "host.docker.internal:host-gateway"

volumes:
  open-webui-data:
```

Start the container:

```bash
docker compose up -d
```

Open WebUI is now running on port 3000. The first user to register becomes the admin.

### Step 3: (Optional) Install Ollama for Local Models

Skip this if you only plan to use external APIs.

```bash
# Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# Pull a starter model
ollama pull llama3.2:3b

# Verify it's running
curl http://localhost:11434/api/tags
```

### Step 4: Set Up Reverse Proxy with Caddy

Caddy handles HTTPS automatically via Let's Encrypt.

```bash
# Install Caddy
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install caddy
```

Edit `/etc/caddy/Caddyfile`:

```
chat.yourdomain.com {
    reverse_proxy localhost:3000
}
```

Restart Caddy:

```bash
sudo systemctl restart caddy
```

Your Open WebUI instance is now live at `https://chat.yourdomain.com` with automatic HTTPS.

### Step 5: Connect External APIs

Once logged in as admin:

1. Go to **Admin Panel → Settings → Connections**
2. Add OpenAI API:
   - URL: `https://api.openai.com/v1`
   - Key: your OpenAI API key
3. Add Anthropic (via OpenAI-compatible proxy like LiteLLM, or use the built-in Anthropic connection if available in your version)

Now users can choose between local Ollama models and cloud models in the same interface.

### Step 6: Secure the Installation

```bash
# Disable public signups after your team registers
# Edit docker-compose.yml, change:
#   ENABLE_SIGNUP=true → ENABLE_SIGNUP=false
# Then: docker compose up -d

# Set up UFW firewall
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# Enable automatic security updates
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

## Cost Comparison: Open WebUI vs. Commercial Alternatives

| Solution | Monthly cost (5 users) | Data privacy | Local model support |
|---|---:|:---:|:---:|
| ChatGPT Team | $125 ($25/user) | Data used for training (opt-out available) | No |
| Claude for Work | $150 ($30/user) | Better privacy policy | No |
| Open WebUI + API keys | $6–$15 (server) + API usage | Full ownership | Yes |
| Open WebUI + Ollama only | $12–$24 (server) | Complete — no external calls | Yes |

For a 5-person team doing moderate AI usage, expect $30–$60/month total with Open WebUI + API keys — about half the cost of commercial plans with better privacy.

## Common Problems and Fixes

### Open WebUI can't connect to Ollama

Most common cause: Docker networking. The container can't reach `localhost` on the host.

```bash
# Verify Ollama is listening on all interfaces
sudo systemctl edit ollama
# Add under [Service]:
# Environment="OLLAMA_HOST=0.0.0.0"
sudo systemctl restart ollama

# Test from inside the container
docker exec open-webui curl http://host.docker.internal:11434/api/tags
```

### High memory usage with multiple models

Ollama keeps the last used model loaded in RAM. With limited memory:

```bash
# Set Ollama to unload models after 5 minutes of inactivity
sudo systemctl edit ollama
# Add: Environment="OLLAMA_KEEP_ALIVE=5m"
sudo systemctl restart ollama
```

### Slow responses with local models

CPU-only inference is inherently slow. Mitigations:

- Use smaller quantized models (Q4_K_M over Q8)
- Pick models with fewer parameters (3B over 7B)
- Increase CPU cores on your VPS
- Accept that 7B models will do 4–7 tokens/second on most VPS CPUs

### Database corruption after hard reboot

Open WebUI uses SQLite by default. If your VPS provider force-reboots machines:

```bash
# Back up regularly
docker exec open-webui cp /app/backend/data/webui.db /app/backend/data/webui.db.bak

# Or switch to PostgreSQL for production reliability
# Add to docker-compose.yml environment:
#   DATABASE_URL=postgresql://user:pass@db:5432/openwebui
```

## Maintenance Checklist

| Task | Frequency | Command |
|---|---|---|
| Update Open WebUI | Weekly | `docker compose pull && docker compose up -d` |
| Update Ollama | Monthly | `curl -fsSL https://ollama.ai/install.sh \| sh` |
| Back up data | Daily (cron) | `docker exec open-webui cp /app/backend/data/webui.db /backups/` |
| Check disk usage | Weekly | `df -h && docker system df` |
| Review user accounts | Monthly | Admin Panel → Users |

## Who Should NOT Self-Host This

- **Non-technical users** — there's no managed version with a support team. If Docker is new to you, expect a learning curve.
- **Teams that need guaranteed uptime** — a single VPS has no redundancy. If the server goes down, everyone loses access.
- **Anyone processing regulated data (HIPAA, SOC2)** — self-hosting alone doesn't make you compliant. You still need proper security controls, audit logs, and possibly a compliance framework.
- **Users who only need occasional AI access** — if you use ChatGPT twice a week, the $20/month subscription is simpler and cheaper than maintaining a server.

## Bottom Line

Open WebUI on a VPS is the best option for small teams and privacy-conscious developers who want ChatGPT-level UX with full data control. The sweet spot is a $10–$15/month Hetzner or Contabo VPS running Open WebUI as an API proxy, with Ollama for sensitive queries that can't leave your server. You get multi-user access, conversation history, document chat, and model flexibility — all for less than a single ChatGPT Team subscription.
