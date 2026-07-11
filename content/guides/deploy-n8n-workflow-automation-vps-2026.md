---
title: "How to Deploy n8n on a VPS: Self-Hosted AI Workflow Automation in 2026"
description: "Complete guide to deploying n8n (open-source workflow automation) on a budget VPS with Docker. Covers provider selection, server sizing, SSL setup, AI node configuration, and cost comparison vs n8n Cloud."
date: 2026-07-11
tags: ["n8n", "workflow automation", "ai tools", "vps", "docker", "self-hosted", "deployment", "budget vps"]
---

One-sentence verdict: n8n is the most flexible self-hosted workflow automation platform available, and deploying it on a $4–12/month VPS gives you unlimited executions instead of paying $20–60+/month for the cloud plan.

## Who This Guide Is For

- Solo developers and indie hackers building automations (Zapier-style) without per-task pricing
- Small teams that need AI-powered workflows connecting LLMs, databases, APIs, and internal tools
- Anyone who hit n8n Cloud's execution limits or wants full data ownership
- DevOps engineers looking to automate infrastructure tasks with a visual builder
- Content creators automating publishing pipelines across platforms

## What Is n8n?

n8n (pronounced "n-eight-n") is an open-source workflow automation platform with 400+ integrations. It competes with Zapier, Make.com, and Power Automate — but with full source code access and no per-execution fees when self-hosted.

Key features:

- **Visual workflow builder** with drag-and-drop nodes
- **AI Agent nodes** — connect OpenAI, Anthropic, Ollama, or any LLM via API
- **400+ integrations** — Slack, Gmail, GitHub, databases, HTTP, SSH, and more
- **Code nodes** — write JavaScript or Python when visual nodes aren't enough
- **Webhook triggers** — receive events from any service in real time
- **Credential encryption** — all secrets encrypted at rest
- **Self-hosted = unlimited executions** — no per-task billing

The n8n Cloud starter plan costs $20/month for 2,500 executions. The Pro plan is $50/month. Self-hosting removes those limits entirely.

## Quick Cost Comparison: Self-Hosted vs n8n Cloud

| Plan | Monthly Cost | Executions | Active Workflows | Data Ownership |
|------|---:|---|---|---|
| n8n Cloud Starter | $20/mo | 2,500 | 5 | n8n servers (EU/US) |
| n8n Cloud Pro | $50/mo | 10,000 | 50 | n8n servers (EU/US) |
| Self-hosted (Hetzner CX22) | €3.29/mo | **Unlimited** | **Unlimited** | Your server |
| Self-hosted (RackNerd 2GB) | ~$4/mo | **Unlimited** | **Unlimited** | Your server |
| Self-hosted (DigitalOcean 2GB) | $12/mo | **Unlimited** | **Unlimited** | Your server |

**Bottom line:** Self-hosting pays for itself within the first month if you run more than a handful of workflows.

## VPS Provider Recommendations for n8n

| Provider | Plan | RAM | vCPU | Storage | Price | Best For |
|----------|------|-----|------|---------|------:|---------|
| Hetzner CX22 | Cloud | 4 GB | 2 | 40 GB | €3.29/mo | Best value overall |
| RackNerd | KVM | 2.5 GB | 2 | 40 GB | ~$4/mo | Cheapest entry (US) |
| Contabo | Cloud VPS S | 8 GB | 4 | 200 GB | €5.99/mo | Most RAM per dollar |
| DigitalOcean | Basic | 2 GB | 1 | 50 GB | $12/mo | Best docs + global DCs |
| Vultr | Cloud Compute | 2 GB | 1 | 55 GB | $12/mo | Hourly billing, 30+ locations |

### Minimum Requirements

- **Light use** (< 20 workflows, no AI nodes): 1 vCPU, 1 GB RAM, 20 GB disk
- **Standard use** (20–100 workflows, webhooks): 2 vCPU, 2 GB RAM, 40 GB disk
- **Heavy / AI workflows** (100+ workflows, LLM calls, image processing): 2–4 vCPU, 4–8 GB RAM, 60+ GB disk

n8n itself uses ~200 MB RAM idle. Each active workflow execution adds 50–150 MB depending on data volume.

## Step-by-Step Deployment with Docker Compose

### Step 1: Provision Your VPS

Choose a provider from the table above. For this guide we'll use Ubuntu 22.04 or 24.04 LTS.

After provisioning:

```bash
# SSH into your server
ssh root@your-server-ip

# Update system
apt update && apt upgrade -y

# Install Docker and Docker Compose
curl -fsSL https://get.docker.com | sh
```

### Step 2: Create Project Directory

```bash
mkdir -p /opt/n8n && cd /opt/n8n
```

### Step 3: Create Docker Compose File

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=n8n.yourdomain.com
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://n8n.yourdomain.com/
      - GENERIC_TIMEZONE=UTC
      # Database (SQLite by default, PostgreSQL recommended for production)
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=your-strong-password-here
      # Encryption key for credentials
      - N8N_ENCRYPTION_KEY=your-random-encryption-key
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=your-strong-password-here
      - POSTGRES_DB=n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  n8n_data:
  postgres_data:
EOF
```

### Step 4: Generate Secure Keys

```bash
# Generate encryption key
echo "N8N_ENCRYPTION_KEY: $(openssl rand -hex 32)"

# Generate database password
echo "DB_PASSWORD: $(openssl rand -base64 24)"
```

Replace the placeholder values in `docker-compose.yml` with these generated keys.

### Step 5: Set Up Reverse Proxy with Caddy (Auto-SSL)

```bash
cat > Caddyfile << 'EOF'
n8n.yourdomain.com {
    reverse_proxy n8n:5678
}
EOF
```

Add Caddy to your `docker-compose.yml`:

```yaml
  caddy:
    image: caddy:2-alpine
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
```

Add volumes:
```yaml
  caddy_data:
  caddy_config:
```

### Step 6: Launch

```bash
docker compose up -d
```

n8n will be available at `https://n8n.yourdomain.com` within 60 seconds. The first visit prompts you to create an owner account.

### Step 7: Verify Installation

```bash
# Check all containers are running
docker compose ps

# Check n8n logs
docker compose logs n8n --tail 50
```

## Configuring AI Nodes in n8n

n8n's AI capabilities make it especially powerful on a VPS:

### OpenAI / Anthropic Integration

1. Go to **Settings → Credentials → Add Credential**
2. Select "OpenAI API" or "Anthropic API"
3. Enter your API key
4. Use the **AI Agent** node or **Chat Model** node in workflows

### Local LLM via Ollama (Same Server)

If your VPS has 8+ GB RAM, you can run Ollama alongside n8n:

```yaml
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    volumes:
      - ollama_data:/root/.ollama
    # Expose only internally
    expose:
      - "11434"
```

Then in n8n, add an Ollama credential pointing to `http://ollama:11434`.

### Example AI Workflow Ideas

- **Content pipeline:** RSS trigger → AI summarize → post to Slack/Discord
- **Email triage:** Gmail trigger → classify with LLM → auto-label + draft response
- **Code review bot:** GitHub webhook → diff analysis with Claude → post comment
- **SEO automation:** Scheduled → scrape rankings → AI generate improvement suggestions
- **Customer support:** Webhook → RAG lookup → draft response → human review queue

## Backup Strategy

```bash
# Automated daily backup script
cat > /opt/n8n/backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/opt/n8n/backups"
mkdir -p $BACKUP_DIR
DATE=$(date +%Y%m%d)

# Backup PostgreSQL
docker exec n8n-postgres pg_dump -U n8n n8n | gzip > $BACKUP_DIR/n8n-db-$DATE.sql.gz

# Keep only last 7 days
find $BACKUP_DIR -name "*.gz" -mtime +7 -delete
EOF

chmod +x /opt/n8n/backup.sh

# Add to crontab - run daily at 3 AM
(crontab -l 2>/dev/null; echo "0 3 * * * /opt/n8n/backup.sh") | crontab -
```

## Security Hardening

1. **Firewall:** Allow only ports 80, 443, and your SSH port
```bash
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 22/tcp
ufw enable
```

2. **Disable public n8n port:** Remove the `ports: - "5678:5678"` mapping since Caddy handles traffic

3. **Enable basic auth** for the editor (optional, adds a layer before n8n's own login):
```
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=secure-password
```

4. **Keep n8n updated:**
```bash
cd /opt/n8n
docker compose pull
docker compose up -d
```

## Performance Tuning

| Setting | Light Use | Standard | Heavy/AI |
|---------|-----------|----------|----------|
| `EXECUTIONS_PROCESS` | `main` | `main` | `own` (separate process per execution) |
| `N8N_CONCURRENCY_PRODUCTION_LIMIT` | 5 | 20 | 50 |
| PostgreSQL `shared_buffers` | 128MB | 256MB | 512MB |
| Swap | 1 GB | 2 GB | 4 GB |

Add swap on low-RAM servers:
```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

## Risks and Limitations

| Risk | Mitigation |
|------|-----------|
| Server goes down = all automations stop | Set up uptime monitoring (UptimeRobot, free tier) |
| No automatic scaling | Monitor RAM; upgrade VPS plan when utilization > 80% |
| Self-managed updates | Subscribe to n8n releases on GitHub; update monthly |
| Credential security is your responsibility | Use strong encryption key; restrict SSH access |
| SQLite data loss on crash | Use PostgreSQL (as shown above) instead of default SQLite |
| IP reputation for email sending | Use external SMTP (SendGrid, Resend) rather than VPS mail |

## Monitoring

Check n8n health:
```bash
curl -s http://localhost:5678/healthz
```

Set up a simple monitoring cron:
```bash
# Alert if n8n is down
*/5 * * * * curl -sf http://localhost:5678/healthz || echo "n8n is DOWN" | mail -s "n8n Alert" you@email.com
```

## Final Cost Breakdown

| Component | Monthly Cost |
|-----------|---:|
| VPS (Hetzner CX22) | €3.29 |
| Domain (.com) | ~€1.00 (amortized) |
| SSL certificate | Free (Caddy auto-TLS) |
| Backups (included w/ Hetzner) | €0.00 |
| **Total** | **~€4.30/mo** |

Compare to n8n Cloud Pro at $50/month — you save over $500/year while getting unlimited executions and full data ownership.

## Conclusion

Self-hosting n8n on a budget VPS is one of the best ROI moves for anyone building automations in 2026. The combination of unlimited executions, AI agent capabilities, and full data control makes it compelling versus any managed plan.

Start with a $4–6/month VPS, deploy with Docker Compose in under 15 minutes, and scale up only when your workflows demand it. For most solo developers and small teams, a 2–4 GB VPS handles hundreds of workflows without breaking a sweat.
