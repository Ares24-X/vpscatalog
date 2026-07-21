---
title: "How to Deploy Flowise on a VPS: Build AI Chatbots Without Code in 2026"
description: "Step-by-step guide to self-hosting Flowise (open-source no-code LLM app builder) on a budget VPS with Docker. Covers server sizing, provider comparison, SSL setup, persistent storage, and cost savings vs managed alternatives."
date: 2026-07-21
tags: ["flowise", "ai chatbot", "langchain", "no-code", "vps", "docker", "self-hosted", "deployment", "budget vps", "llm"]
---

One-sentence verdict: Flowise lets you build production-ready AI chatbots and RAG pipelines with a drag-and-drop interface, and self-hosting it on a $5–10/month VPS gives you unlimited builds with full data privacy — no vendor lock-in.

## Who This Guide Is For

- Non-technical founders who want custom AI chatbots without writing code
- Developers prototyping RAG (Retrieval-Augmented Generation) apps quickly
- Agencies building AI assistants for multiple clients on a single server
- Anyone who wants a visual LangChain/LlamaIndex builder with full control over data
- Teams needing private LLM workflows that never leave their infrastructure

## What Is Flowise?

Flowise is an open-source, low-code platform for building LLM-powered applications. Think of it as a visual canvas where you drag, connect, and configure AI components — LLMs, vector stores, document loaders, memory modules, and tools — without writing boilerplate code.

Key features:

- **Visual drag-and-drop builder** — connect LangChain and LlamaIndex components visually
- **50+ integrations** — OpenAI, Anthropic, Ollama, Pinecone, Qdrant, PostgreSQL, and more
- **Chat embed widget** — drop a chatbot on any website with one script tag
- **API access** — every chatflow becomes a REST API endpoint automatically
- **Document loaders** — ingest PDFs, CSVs, websites, Notion pages for RAG
- **Multi-user support** — role-based access for teams and agencies
- **Marketplace** — community-shared templates for common use cases
- **Credentials encryption** — API keys stored encrypted at rest

Flowise competes with Langflow, Dify, and commercial platforms like Voiceflow or Botpress. The difference: it's fully open-source (Apache 2.0), lightweight, and runs comfortably on minimal hardware.

## Quick Cost Comparison: Self-Hosted vs Alternatives

| Option | Monthly Cost | Chatflows | API Calls | Data Location |
|--------|---:|---|---|---|
| Flowise Cloud (Starter) | $35/mo | 5 | 5,000/mo | Flowise servers |
| Flowise Cloud (Pro) | $65/mo | 20 | 25,000/mo | Flowise servers |
| Dify Cloud (Professional) | $59/mo | 50 apps | Limited | Dify servers |
| **Self-hosted (Hetzner CX22)** | **€3.29/mo** | **Unlimited** | **Unlimited** | **Your server** |
| **Self-hosted (RackNerd 2GB)** | **$3.49/mo** | **Unlimited** | **Unlimited** | **Your server** |

Self-hosting pays for itself in month one if you're building more than a single chatbot.

## Minimum Server Requirements

Flowise is a Node.js app — it's lightweight compared to Python-heavy alternatives.

| Workload | vCPU | RAM | Storage | Estimated Cost |
|----------|---:|---:|---:|---:|
| Development / 1–2 chatflows | 1 | 1 GB | 20 GB | $3–5/mo |
| Production / 5–15 chatflows | 2 | 2 GB | 40 GB | $5–10/mo |
| Agency / 20+ chatflows + vector DB | 2–4 | 4 GB | 80 GB SSD | $10–20/mo |

> **Important:** These specs cover Flowise itself. If you run a local LLM via Ollama alongside it, you'll need significantly more RAM (16 GB+) and ideally a GPU VPS. For most users, connecting to external LLM APIs (OpenAI, Anthropic, Groq) is more cost-effective.

## Recommended VPS Providers

| Provider | Plan | Specs | Price | Best For |
|----------|------|-------|---:|---|
| **Hetzner** | CX22 | 2 vCPU / 4 GB / 40 GB | €3.29/mo | Best value in EU (Germany/Finland) |
| **RackNerd** | VPS 2GB | 2 vCPU / 2 GB / 40 GB | $3.49/mo | Budget US hosting, annual deals |
| **Contabo** | Cloud VPS S | 4 vCPU / 8 GB / 50 GB | €6.99/mo | Raw specs per dollar |
| **DigitalOcean** | Basic Droplet | 1 vCPU / 2 GB / 50 GB | $12/mo | Beginner-friendly UI, good docs |
| **Vultr** | Cloud Compute | 1 vCPU / 2 GB / 50 GB | $12/mo | Global locations, hourly billing |

**Our pick for most users:** Hetzner CX22 — unbeatable price-to-performance for a Node.js workload like Flowise.

## Risk Considerations

Before deploying, understand the tradeoffs:

- **No automatic updates** — you manage upgrades yourself (Docker makes this easy)
- **You handle backups** — set up automated snapshots or volume backups
- **Security is on you** — firewall, SSL, and access control are your responsibility
- **LLM API costs still apply** — self-hosting Flowise is free, but OpenAI/Anthropic tokens are not
- **Single point of failure** — unless you configure redundancy, one server = one failure domain
- **No managed support** — community Discord is active, but there's no SLA

## Step-by-Step Deployment with Docker

### Prerequisites

- A VPS with Ubuntu 22.04 or 24.04 (Debian also works)
- A domain name pointed to your server's IP (for SSL)
- SSH access to the server

### Step 1: Initial Server Setup

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker and Docker Compose
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Log out and back in for group changes
exit
```

### Step 2: Create Project Directory

```bash
mkdir -p ~/flowise && cd ~/flowise
```

### Step 3: Create Docker Compose File

```bash
cat > docker-compose.yml << 'EOF'
version: "3.8"

services:
  flowise:
    image: flowiseai/flowise:latest
    container_name: flowise
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - FLOWISE_USERNAME=admin
      - FLOWISE_PASSWORD=CHANGE_THIS_PASSWORD
      - APIKEY_STORAGE_TYPE=json
      - DATABASE_TYPE=sqlite
      - DATABASE_PATH=/root/.flowise
      - SECRETKEY_PATH=/root/.flowise
      - LOG_LEVEL=info
    volumes:
      - flowise_data:/root/.flowise

volumes:
  flowise_data:
EOF
```

### Step 4: Start Flowise

```bash
docker compose up -d
```

Flowise is now running on port 3000. Verify:

```bash
curl -s http://localhost:3000 | head -5
```

### Step 5: Set Up Nginx Reverse Proxy with SSL

```bash
# Install Nginx and Certbot
sudo apt install -y nginx certbot python3-certbot-nginx

# Create Nginx config
sudo cat > /etc/nginx/sites-available/flowise << 'EOF'
server {
    listen 80;
    server_name flowise.yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }
}
EOF

# Enable site and get SSL
sudo ln -sf /etc/nginx/sites-available/flowise /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d flowise.yourdomain.com --non-interactive --agree-tos -m your@email.com
```

### Step 6: Configure Firewall

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

### Step 7: Verify Deployment

Visit `https://flowise.yourdomain.com` — you should see the Flowise login screen. Enter the credentials you set in the Docker Compose file.

## Upgrading Flowise

```bash
cd ~/flowise
docker compose pull
docker compose up -d
```

That's it. Docker handles the image update while preserving your data in the named volume.

## Production Hardening Checklist

| Task | Command / Action | Priority |
|------|-----------------|----------|
| Change default password | Update `FLOWISE_PASSWORD` in docker-compose.yml | Critical |
| Enable API key auth | Set `FLOWISE_SECRETKEY_OVERWRITE` environment variable | High |
| Automated backups | `docker run --rm -v flowise_data:/data -v ~/backups:/backup alpine tar czf /backup/flowise-$(date +%F).tar.gz /data` | High |
| Log rotation | Add `logging.options` to Docker Compose | Medium |
| Rate limiting | Add `limit_req_zone` in Nginx config | Medium |
| Fail2ban | Protect SSH and Nginx from brute force | Medium |
| Unattended upgrades | `sudo apt install unattended-upgrades` | Low |

## Using PostgreSQL Instead of SQLite (Recommended for Production)

For production workloads with multiple users, switch to PostgreSQL:

```yaml
version: "3.8"

services:
  flowise:
    image: flowiseai/flowise:latest
    container_name: flowise
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - FLOWISE_USERNAME=admin
      - FLOWISE_PASSWORD=CHANGE_THIS_PASSWORD
      - DATABASE_TYPE=postgres
      - DATABASE_HOST=db
      - DATABASE_PORT=5432
      - DATABASE_NAME=flowise
      - DATABASE_USER=flowise
      - DATABASE_PASSWORD=CHANGE_DB_PASSWORD
      - SECRETKEY_PATH=/root/.flowise
    volumes:
      - flowise_data:/root/.flowise
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    container_name: flowise_db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=flowise
      - POSTGRES_USER=flowise
      - POSTGRES_PASSWORD=CHANGE_DB_PASSWORD
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  flowise_data:
  postgres_data:
```

## Embedding the Chatbot on Your Website

Once you've built a chatflow, Flowise generates an embed script:

```html
<script type="module">
  import Chatbot from "https://cdn.jsdelivr.net/npm/flowise-embed/dist/web.js";
  Chatbot.init({
    chatflowid: "your-chatflow-id",
    apiHost: "https://flowise.yourdomain.com",
  });
</script>
```

This adds a chat bubble to any website — no backend code needed on the frontend.

## Performance Tips

- **Use streaming responses** — enable streaming in your chatflow for better perceived latency
- **Cache embeddings** — if using RAG, store computed embeddings to avoid reprocessing documents
- **Use Groq or Together AI** for fast inference — cheaper and faster than OpenAI for many tasks
- **Monitor memory** — run `docker stats flowise` to watch resource usage
- **Set `LOG_LEVEL=error`** in production to reduce I/O

## Frequently Asked Questions

**Can I run Flowise without Docker?**
Yes — `npx flowise start` works, but Docker provides isolation, easy upgrades, and reproducible deployments.

**Does Flowise support multiple users?**
Yes. The enterprise features (RBAC, audit logs) are available in the open-source version since v1.8+.

**Can I connect Flowise to a local LLM?**
Absolutely. Run Ollama on the same server or a separate GPU VPS, then point Flowise's ChatOllama node to `http://localhost:11434`.

**How much bandwidth does Flowise use?**
Minimal. The app itself is lightweight. Bandwidth depends on how many API calls your chatflows make and how many users interact with your embedded bots.

**Is my data safe?**
All credentials are encrypted. Conversation logs stay on your server. No telemetry is sent to Flowise unless you opt in.

## Conclusion

Flowise is the fastest way to go from idea to deployed AI chatbot without writing LangChain boilerplate. Self-hosting on a $5/month VPS gives you:

- Unlimited chatflows and API calls
- Full data sovereignty
- No monthly SaaS fees beyond your LLM API usage
- A production-ready setup in under 30 minutes

For most users, a Hetzner CX22 (€3.29/month) or RackNerd 2GB plan ($3.49/month) is more than enough to run Flowise with multiple active chatflows serving real users.

Start with the Docker setup above, build your first RAG chatbot, and embed it on your site — all in an afternoon.
