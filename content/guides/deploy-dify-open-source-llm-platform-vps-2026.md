---
title: "How to Deploy Dify on a VPS: Open-Source LLM App Platform in 2026"
description: "Step-by-step guide to deploying Dify (open-source LLMOps platform) on a VPS with Docker. Covers provider choice, server sizing, SSL, backups, and cost analysis."
date: 2026-07-09
tags: ["dify", "open-source", "llm", "ai tools", "vps", "docker", "self-hosted", "deployment"]
---

One-sentence verdict: Dify is the fastest way to build production-ready LLM applications on your own VPS, and with the right server choice you can run it reliably for $5–20/month instead of paying for the managed cloud plan.

## Who This Guide Is For

- Developers who want to self-host an LLM application platform instead of paying Dify Cloud ($59+/month)
- Indie hackers building AI-powered internal tools, chatbots, or knowledge bases
- Small teams that need API key management, prompt engineering, and RAG pipelines under one roof
- Anyone who has tried OpenAI's playground and wants more workflow control without vendor lock-in

## What Is Dify?

Dify is an open-source LLM application development platform. Think of it as a self-hosted alternative to the OpenAI Assistants API plus a visual workflow builder. It supports:

- **Chatbots and conversational apps** with prompt orchestration
- **Knowledge bases (RAG)** that ingest PDFs, web pages, and documents
- **Visual workflow builder** for multi-step AI pipelines
- **API endpoint generation** so you can embed AI into your own apps
- **Multi-model support**: OpenAI, Anthropic, DeepSeek, local models via Ollama, and more

The cloud version starts at $59/month. Self-hosting on a VPS brings that cost down to the price of the server.

## Quick Comparison: VPS Providers for Dify

| Provider | Starting Price | RAM | Best For | Main Trade-off | CTA |
|---|---:|---|------|-------------|
| Hetzner CX22 | €3.29/mo | 4 GB | Best value, EU datacenters | Limited global regions | [Start with Hetzner →](https://www.hetzner.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| Hetzner CX32 | €6.39/mo | 8 GB | Production Dify with multiple knowledge bases | EU/US only | [Start with Hetzner →](https://www.hetzner.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| DigitalOcean | $24/mo | 4 GB | Best docs, global regions | 4× more expensive per GB of RAM | [Start with DigitalOcean →](https://www.digitalocean.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| RackNerd | ~$4/mo | 2 GB | Cheapest entry point | 2 GB is tight for Dify; use only with external DB | [Start with RackNerd →](https://www.racknerd.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| Vultr | $12/mo | 2 GB | Global region choice and hourly billing | Needs the 4 GB plan ($24/mo) for comfortable Dify | [Start with Vultr →](https://www.vultr.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |

## Recommended Server Sizing for Dify

Dify runs as a set of Docker containers: API server, worker, web frontend, PostgreSQL, Redis, and Weaviate (vector database). Together they are heavier than a typical web app.

| Workload | Minimum VPS | Monthly Cost | Notes |
|---|---|---|---|
| Testing and personal use | 2 vCPU / 4 GB RAM | €3.29–$12 | Fine for 1–2 users, 1 knowledge base |
| Small team production | 4 vCPU / 8 GB RAM | €6.39–$24 | Multiple knowledge bases, 5–10 users |
| Heavy RAG workload | 4 vCPU / 16 GB RAM | €15–$48 | Large document ingestion, many concurrent queries |

**Important:** The 2 GB plans (RackNerd $4, Vultr $12) will struggle with the full Dify stack. If you use an external managed database and vector store, 2 GB can work for the app layer only — but that adds complexity and another bill. For beginners, 4 GB is the practical minimum.

## Step-by-Step Deployment

### Step 1: Choose and Provision Your VPS

Pick a provider from the table above. For this guide, I use a Hetzner CX22 (2 vCPU, 4 GB RAM, 40 GB SSD at €3.29/month).

Create the server with:
- **OS:** Ubuntu 22.04 or 24.04 LTS
- **SSH key:** Add yours during creation
- **Firewall:** Open ports 22 (SSH), 80 (HTTP), 443 (HTTPS)

### Step 2: Install Docker and Docker Compose

```bash
# Connect to your VPS
ssh root@your-server-ip

# Install Docker
curl -fsSL https://get.docker.com | sh

# Add your user to the docker group (optional, if not using root)
usermod -aG docker $USER

# Verify installation
docker --version
docker compose version
```

### Step 3: Clone Dify and Start the Stack

```bash
# Clone the Dify repository
git clone https://github.com/langgenius/dify.git
cd dify/docker

# Copy the example environment file
cp .env.example .env

# Edit the .env file — set a strong SECRET_KEY and choose your ports
nano .env

# Start all services
docker compose up -d
```

Wait 1–2 minutes for all containers to start. Check with `docker compose ps` — you should see containers for api, worker, web, db, redis, and weaviate all running.

### Step 4: Set Up a Reverse Proxy with SSL

Dify's web interface runs on port 3000 by default. You need a reverse proxy to serve it over HTTPS.

```bash
# Install Nginx
apt update && apt install -y nginx

# Install Certbot for Let's Encrypt SSL
apt install -y certbot python3-certbot-nginx

# Create Nginx config
cat > /etc/nginx/sites-available/dify << 'EOF'
server {
    server_name your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
    }
}
EOF

# Enable the site
ln -s /etc/nginx/sites-available/dify /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx

# Get SSL certificate
certbot --nginx -d your-domain.com
```

### Step 5: Initial Dify Setup

1. Open `https://your-domain.com` in your browser
2. Create your admin account (email + password)
3. Add at least one model provider: go to Settings → Model Provider → add your OpenAI, Anthropic, or DeepSeek API key
4. Create your first app — choose between Chatbot, Text Generator, Agent, or Workflow

### Step 6: Configure Backups

Dify stores data in PostgreSQL and files on disk. Set up automated backups:

```bash
# Daily database backup script
cat > /opt/backup-dify.sh << 'EOF'
#!/bin/bash
BACKUP_DIR=/opt/dify-backups
mkdir -p $BACKUP_DIR
DATE=$(date +%Y%m%d)
cd /root/dify/docker
docker compose exec -T db pg_dump -U postgres dify > $BACKUP_DIR/dify-db-$DATE.sql
# Keep last 7 days of backups
find $BACKUP_DIR -name "*.sql" -mtime +7 -delete
EOF

chmod +x /opt/backup-dify.sh

# Add to crontab (daily at 3 AM)
(crontab -l 2>/dev/null; echo "0 3 * * * /opt/backup-dify.sh") | crontab -
```

## Cost Breakdown: Self-Hosted vs Dify Cloud

| Item | Dify Cloud (Professional) | Self-Hosted on Hetzner CX32 | Self-Hosted on DigitalOcean |
|---|---|---|---|
| Monthly platform cost | $59 | €6.39 | $24 |
| Annual platform cost | $708 | ~€77 | $288 |
| LLM API costs | Separate | Separate | Separate |
| Setup time | 5 minutes | 30–60 minutes | 30–60 minutes |
| Maintenance | None | OS updates, Docker updates | OS updates, Docker updates |
| Data control | Dify's infrastructure | Full control | Full control |
| Custom domain | Yes | Yes | Yes |
| Team members | Unlimited | Unlimited | Unlimited |

**The bottom line:** Self-hosting saves you $53–$65/month after the one-time setup investment. Over a year, that is $630–$780 saved — enough to pay for several VPS projects or a significant LLM API budget.

## Risks and Limitations

### 1. Server Maintenance Is Your Responsibility

Unlike the cloud version, you handle OS updates, Docker updates, security patches, and monitoring. Budget 1–2 hours per month for maintenance. If you miss a critical CVE patch, your server and API keys are at risk.

### 2. Resource Contention

Dify's PostgreSQL, Redis, and Weaviate containers compete for CPU and RAM. A large document ingestion job can slow down the chatbot interface for other users. Monitor with `docker stats` and consider a separate database server if you scale past 10 users.

### 3. Single Point of Failure

Without a load balancer or redundant database, your VPS is a single point of failure. If the server goes down, your Dify apps go down. Mitigate with:
- Provider-level snapshots or automatic backups
- Uptime monitoring (UptimeRobot, Better Uptime Stack)
- A documented restore procedure you have actually tested

### 4. Vector Database Growth

Weaviate stores embeddings — those grow with every document you ingest. Monitor disk usage with `df -h`. A 40 GB SSD fills faster than you expect when uploading hundreds of documents. Start with at least 40 GB and upgrade if needed.

### 5. API Key Exposure

Dify stores your LLM provider API keys. If the server is compromised, those keys can be abused — and you pay the bill. Always:
- Use SSH key authentication, never passwords
- Set spending limits on your OpenAI/Anthropic accounts
- Rotate API keys periodically
- Keep Dify behind HTTPS with valid SSL

## When Self-Hosting Dify Makes Sense

| Scenario | Recommendation |
|---|---|
| You are prototyping and want zero ops | Use Dify Cloud ($59/mo) |
| You have 1–3 internal tools and can handle Docker | Self-host on 4 GB VPS |
| Your team is 5+ people with heavy RAG usage | Self-host on 8 GB VPS |
| You need HIPAA/GDPR data residency guarantees | Self-host (only option) |
| You want to customize Dify's source code | Self-host (only option) |
| You are an agency building client AI tools | Self-host on 8 GB+ VPS, one instance per client |

## Alternative Tools Worth Considering

If Dify feels too heavy or you have a more focused use case, consider these alternatives:

| Tool | Best For | Resource Needs | Self-Hosted Cost |
|---|---|---|---|
| Flowise | Visual LLM workflow builder (drag-and-drop) | 1 vCPU / 2 GB RAM | $4–$6/month |
| Open WebUI | ChatGPT-like interface for local/remote models | 1 vCPU / 2 GB RAM | $4–$6/month |
| n8n + AI nodes | Automation with AI steps (email, Slack, webhooks + LLM) | 1 vCPU / 1 GB RAM | $3–$5/month |
| LangFlow | LangChain visual editor | 2 vCPU / 4 GB RAM | $5–$12/month |

Each of these runs comfortably on a budget VPS and can complement or replace Dify depending on your exact needs.

## Verdict

Dify is mature enough for production use on a VPS in mid-2026. The Docker deployment is well-documented, the community is active, and the feature set rivals managed platforms that cost 5–10× more. The trade-off is server administration — but if you are reading a VPS guide, you probably already accept that.

**Primary recommendation:** Hetzner CX32 (8 GB RAM, €6.39/month) gives you comfortable headroom for Dify plus room for a second project. It is the sweet spot between cost and capability.

**Budget alternative:** Hetzner CX22 (4 GB RAM, €3.29/month) works for personal use and light team workloads.

[Start with Hetzner → (commission €50/sale)](https://www.hetzner.com/?ref=AFFILIATE_TAG_PLACEHOLDER)

Not ready to manage a server? Try [Dify Cloud (from $59/month)](https://dify.ai/?ref=AFFILIATE_TAG_PLACEHOLDER) for the fully managed experience.
