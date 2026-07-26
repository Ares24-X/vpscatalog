---
title: "Best VPS for Running Telegram & Discord Bots in 2026"
description: "A practical guide to choosing a VPS for hosting Telegram bots, Discord bots, and other always-on chat automation. Covers specs, pricing, deployment steps, and common pitfalls."
date: 2026-07-26
tags: ["vps for bots", "telegram bot hosting", "discord bot hosting", "cheap vps", "always-on server", "bot deployment"]
---

One-sentence verdict: A $2–5/month VPS with 1 vCPU and 1 GB RAM handles most single-purpose bots comfortably — RackNerd and Hostinger offer the best value, while Hetzner gives the cleanest developer experience for multi-bot setups.

## Who This Guide Is For

- Developers running one or more Telegram, Discord, Slack, or WhatsApp bots that need 24/7 uptime.
- Community managers who outgrew free-tier hosting (Heroku eco dynos, Railway hobbyist limits, Replit deployments that sleep).
- Bot builders who need persistent storage, cron scheduling, or webhook endpoints.
- Anyone tired of bots going offline when their laptop closes.

## Who Should Look Elsewhere

- Bots that process millions of messages per day or handle real-time voice (you need 4+ GB RAM and dedicated CPU).
- Enterprise compliance scenarios requiring SOC 2 certification from your host.
- People who want zero server management — use Railway, Fly.io, or a managed container platform instead.

## What Specs Actually Matter for Bots

Most chat bots are I/O-bound, not CPU-bound. They spend 99% of their time waiting for API responses. Here's what you actually need:

| Requirement | Why It Matters | Minimum Spec |
|---|---|---|
| RAM | Bot framework + runtime overhead | 512 MB (1 bot), 1 GB (2–4 bots) |
| CPU | Message parsing, occasional media processing | 1 shared vCPU is fine |
| Storage | Logs, SQLite databases, media cache | 10–20 GB SSD |
| Bandwidth | API calls are tiny; media bots use more | 500 GB–1 TB/month |
| Uptime | Bots should respond within seconds | 99.5%+ (most budget VPS deliver this) |
| IPv4 | Required for most bot APIs | 1 dedicated IPv4 |

**The spec trap:** Don't overspend. A bot responding to 1,000 messages/day uses less than 5% of a 1 vCPU/1 GB VPS. Scale up only when you hit actual memory limits.

## Recommended VPS Providers

### 1. RackNerd — Best Raw Value

| Plan | Price | vCPU | RAM | Storage | Bandwidth |
|---|---|---|---|---|---|
| KVM 768 MB | $1.49/month (billed annually) | 1 | 768 MB | 15 GB SSD | 1 TB |
| KVM 2 GB | $2.49/month (billed annually) | 2 | 2 GB | 30 GB SSD | 2.5 TB |

**Pros:** Unbeatable price for always-on workloads. Annual billing locks in rates. Multiple US datacenter locations.

**Cons:** No managed services. Support is ticket-only. Panel is basic SolusVM. No automatic backups on cheapest tiers.

**Best for:** Single-purpose bots on a tight budget. Developers comfortable with Linux CLI.

### 2. Hostinger VPS — Best Onboarding

| Plan | Price | vCPU | RAM | Storage | Bandwidth |
|---|---|---|---|---|---|
| KVM 1 | $4.99/month | 1 | 4 GB | 50 GB NVMe | 4 TB |
| KVM 2 | $6.99/month | 2 | 8 GB | 100 GB NVMe | 8 TB |

**Pros:** Generous specs for the price. Clean control panel. AI-assisted setup wizard. Good documentation. Multiple global locations.

**Cons:** Introductory pricing requires 48-month commitment for best rates. Renewal prices are higher.

**Best for:** First-time VPS users who want hand-holding. Running multiple bots plus a small web dashboard.

### 3. Hetzner Cloud — Best Developer Experience

| Plan | Price | vCPU | RAM | Storage | Bandwidth |
|---|---|---|---|---|---|
| CX22 | €3.99/month | 2 shared | 4 GB | 40 GB | 20 TB |
| CAX11 (ARM) | €3.99/month | 2 Ampere | 4 GB | 40 GB | 20 TB |

**Pros:** Excellent API and CLI (`hcloud`). Hourly billing — spin up, test, destroy. Snapshots are cheap. EU and US datacenters. ARM option runs Node.js/Python bots with lower power draw.

**Cons:** No US East location yet. Requires ID verification that can take 1–2 days. Strict anti-abuse policy.

**Best for:** Developers who want infrastructure-as-code, CI/CD pipelines, or multi-bot orchestration with Docker Compose.

### 4. DigitalOcean — Best Ecosystem

| Plan | Price | vCPU | RAM | Storage | Bandwidth |
|---|---|---|---|---|---|
| Basic Droplet | $6/month | 1 shared | 1 GB | 25 GB SSD | 1 TB |
| Basic Droplet | $12/month | 1 | 2 GB | 50 GB SSD | 2 TB |

**Pros:** Massive tutorial library. One-click app marketplace. Monitoring built in. Team features. Excellent uptime track record.

**Cons:** More expensive per-spec than alternatives. 1 GB plan feels tight for modern bot frameworks.

**Best for:** Teams, production bots with uptime requirements, and developers who value documentation.

## Price Comparison Summary

| Provider | Cheapest Suitable Plan | Monthly Cost | RAM | Best Feature |
|---|---|---|---|---|
| RackNerd | KVM 768 MB | $1.49 | 768 MB | Price |
| Hetzner | CX22 / CAX11 | ~$4.30 | 4 GB | API + hourly billing |
| Hostinger | KVM 1 | $4.99 | 4 GB | Onboarding UX |
| DigitalOcean | Basic 1 GB | $6.00 | 1 GB | Ecosystem + tutorials |

## Step-by-Step: Deploy a Bot on a Fresh VPS

This applies to any provider. We'll use a Python Telegram bot as the example.

### 1. Initial Server Setup (5 minutes)

```bash
# SSH in as root, then create a non-root user
adduser botuser
usermod -aG sudo botuser

# Basic hardening
apt update && apt upgrade -y
apt install -y ufw fail2ban
ufw allow OpenSSH
ufw enable

# Switch to your user
su - botuser
```

### 2. Install Runtime (2 minutes)

```bash
# For Python bots
sudo apt install -y python3 python3-pip python3-venv

# For Node.js bots
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

### 3. Deploy Your Bot (3 minutes)

```bash
# Clone your bot repo
git clone https://github.com/yourname/your-bot.git
cd your-bot

# Python example
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Create .env with your bot token
echo "BOT_TOKEN=your_token_here" > .env
```

### 4. Keep It Running with systemd (2 minutes)

```bash
sudo tee /etc/systemd/system/mybot.service << 'EOF'
[Unit]
Description=My Telegram Bot
After=network.target

[Service]
Type=simple
User=botuser
WorkingDirectory=/home/botuser/your-bot
Environment=PATH=/home/botuser/your-bot/venv/bin:/usr/bin
ExecStart=/home/botuser/your-bot/venv/bin/python main.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable mybot
sudo systemctl start mybot
sudo systemctl status mybot
```

### 5. Monitor and Maintain

```bash
# Check logs
journalctl -u mybot -f

# Auto-restart on crash is handled by systemd
# Set up unattended security updates
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

## Common Pitfalls and How to Avoid Them

### Bot Goes Offline After SSH Disconnect

**Problem:** Running the bot with `python main.py` directly — it dies when you close the terminal.

**Solution:** Always use systemd (shown above) or at minimum `screen`/`tmux`. systemd is the correct production approach because it auto-restarts on crash.

### Memory Leaks Kill the Bot After Days

**Problem:** Python bots using `python-telegram-bot` or `discord.py` can leak memory if you store message history in RAM or don't close HTTP sessions properly.

**Solution:** Monitor with `systemctl status mybot` (shows memory usage). Set `MemoryMax=512M` in the systemd unit to prevent runaway processes from killing your VPS. Restart weekly with a cron job if the leak is unfixable.

### Rate Limits and IP Bans

**Problem:** Telegram and Discord rate-limit aggressively. Cheap VPS IPs are sometimes pre-flagged from previous abuse.

**Solution:** Implement exponential backoff in your bot code. If you get persistent 429 errors on a fresh VPS, open a ticket with your provider to get a clean IP. Avoid RackNerd's cheapest IPs for Discord if you experience immediate CloudFlare challenges.

### No Backups, Bot Token Exposed

**Problem:** VPS disk dies, you lose your bot's database. Or worse, you commit your token to a public repo.

**Solution:**
- Back up SQLite databases to an object storage bucket (Backblaze B2 is free for 10 GB).
- Use `.env` files and never commit them. Add `.env` to `.gitignore`.
- Rotate your bot token immediately if exposed.

### Webhook vs Polling

**Problem:** Polling (getUpdates) is simple but adds latency and uses more resources than webhooks.

**Solution:**
- **Use polling** for development and bots with fewer than 100 users. Simpler to debug, no SSL needed.
- **Use webhooks** for production bots with fast response requirements. Requires a domain and SSL cert (free with Let's Encrypt + Caddy/nginx).

## When to Scale Beyond a Single VPS

You've outgrown a basic VPS when:

- RAM usage consistently exceeds 80% of your allocation.
- Bot response times exceed 2 seconds under normal load.
- You're running 5+ bots and deployments interfere with each other.
- You need zero-downtime deploys (blue-green or rolling).

**Next steps:** Move to Docker Compose on a 4 GB+ VPS, or split bots across multiple cheap VPS instances. For serious scale, consider Kubernetes on Hetzner Cloud or DigitalOcean's managed K8s.

## Final Recommendation

For most bot developers, the sweet spot is:

- **Just starting out or single bot:** RackNerd 768 MB ($1.49/month) — it works, it's cheap, you'll learn Linux.
- **Multiple bots or want comfort:** Hetzner CX22 ($4.30/month) — best specs-per-dollar with proper developer tooling.
- **First VPS ever, need guidance:** Hostinger KVM 1 ($4.99/month) — the control panel won't scare you away.

All three will keep your bot running 24/7 without breaking the bank. Start small, monitor actual usage for a week, and resize only if the numbers demand it.
