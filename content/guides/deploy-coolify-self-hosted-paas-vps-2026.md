---
title: "How to Deploy Coolify on a VPS – Self-Hosted PaaS Alternative to Vercel & Heroku (2026)"
description: "Complete guide to installing Coolify on a VPS for self-hosted app deployments. Covers server requirements, installation steps, SSL setup, resource planning, and cost comparison vs Vercel, Railway, and Render."
date: 2026-07-15
tags: ["coolify", "paas", "self-hosted", "vps", "vercel-alternative", "heroku-alternative", "deployment"]
---

One-sentence verdict: Coolify turns a $5–$20/month VPS into your own Vercel/Heroku with Git-push deploys, automatic SSL, and Docker-based isolation — but you own the infrastructure and the bill stays flat no matter how many projects you ship.

## Who This Guide Is For

- Indie developers tired of Vercel/Railway bills scaling past $50/month with modest traffic.
- Small teams running 3–15 apps who want a single control plane without per-seat pricing.
- Freelancers deploying client sites who need isolated environments without creating separate cloud accounts.
- Self-hosting enthusiasts who want a GUI-based deployment platform without writing Docker Compose for every project.

If you need multi-region edge deployments with sub-50ms global latency, stick with Vercel or Cloudflare. Coolify runs on a single server (or a small cluster) and is best for centralized, predictable workloads.

## What Coolify Actually Does

Coolify is an open-source, self-hostable platform that handles:

- **Git-push deployments** — connect GitHub/GitLab/Gitea repos and deploy on push.
- **Automatic SSL** — Let's Encrypt certificates provisioned and renewed automatically.
- **Database management** — one-click PostgreSQL, MySQL, Redis, MongoDB instances.
- **Docker-based isolation** — every app runs in its own container.
- **Built-in monitoring** — CPU, RAM, disk usage per service.
- **Team collaboration** — multiple users with role-based access.

Think of it as Dokku with a modern UI, or a self-hosted Railway.

## Server Requirements

### Minimum Specs by Use Case

| Use Case | RAM | vCPU | Disk | Monthly Cost | Example Provider |
|---|---:|---:|---:|---:|---|
| Coolify only (control plane) | 2 GB | 2 | 30 GB | $5–$7 | RackNerd, Contabo |
| 1–3 small apps + databases | 4 GB | 2 | 50 GB | $8–$12 | Hetzner, DigitalOcean |
| 5–10 apps + CI builds | 8 GB | 4 | 80 GB | $15–$25 | Hetzner, Vultr |
| 10–20 apps + heavy builds | 16 GB | 6 | 160 GB | $30–$50 | Hetzner, Contabo |

**Critical note on RAM:** Coolify itself uses 500–800 MB at idle. Each deployed app adds 100–500 MB depending on the runtime. Node.js apps with build steps can spike to 1–2 GB during deployment. If you run out of RAM during builds, containers get OOM-killed silently.

### Recommended OS

- Ubuntu 22.04 or 24.04 LTS (officially supported)
- Debian 12 (community-tested, works fine)

Avoid Alpine or CentOS — Coolify's installer expects apt and systemd.

## Cost Comparison: Coolify vs Managed PaaS

| Platform | 5 Apps + 2 DBs | 10 Apps + 5 DBs | Bandwidth (1 TB) | SSL | Custom Domains |
|---|---:|---:|---:|---|---|
| **Coolify on Hetzner** | $7/mo (4 GB VPS) | $16/mo (8 GB VPS) | Included | Free (Let's Encrypt) | Unlimited |
| Vercel Pro | $20/mo + $20/member | $20/mo + overages | $40/100 GB overage | Included | Included |
| Railway | ~$25–$60/mo (usage) | ~$80–$150/mo | $0.10/GB | Included | Included |
| Render | $7/service × 7 = $49 | $7/service × 15 = $105 | Included (100 GB) | Included | Included |
| DigitalOcean App Platform | $5/app × 7 = $35 | $5/app × 15 = $75 | Included | Included | Included |

The breakeven point: if you run more than 2–3 apps, self-hosting Coolify is almost always cheaper. The tradeoff is maintenance time.

## Step-by-Step Installation

### Step 1: Provision Your VPS

Pick any provider with a clean Ubuntu 22.04+ image. Minimum 2 GB RAM, 2 vCPU, 30 GB disk.

Recommended providers for Coolify:

- **Hetzner** (best price/performance in EU/US) — CX22 at €4.49/mo for 4 GB RAM
- **Contabo** (maximum RAM per dollar) — 8 GB for ~$6.99/mo
- **DigitalOcean** (solid US/SG/EU presence) — $12/mo for 2 GB with premium CPU
- **Vultr** (widest location selection) — $12/mo for 2 GB

### Step 2: Initial Server Setup

SSH into your fresh server:

```bash
# Update system
apt update && apt upgrade -y

# Set hostname (optional, helps identify in Coolify UI)
hostnamectl set-hostname coolify-prod

# Enable firewall basics
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 8000/tcp   # Coolify UI (can be removed after setup)
ufw enable
```

### Step 3: Install Coolify

One command does it all:

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

This script installs Docker, Docker Compose, and Coolify. Takes 2–5 minutes depending on server speed.

After installation, access the UI at `http://YOUR_SERVER_IP:8000`.

### Step 4: Initial Configuration

1. **Create admin account** — first visitor becomes admin.
2. **Add your server** — Coolify auto-detects the localhost. Click "Validate" to confirm Docker access.
3. **Configure domain** — Point a wildcard DNS (`*.deploy.yourdomain.com`) to your server IP. Set this in Settings → Wildcard Domain.
4. **Enable auto-SSL** — Coolify handles Let's Encrypt automatically once the domain resolves.

### Step 5: Deploy Your First App

1. Go to **Projects → New → Application**.
2. Select **GitHub** (connect via OAuth or deploy key).
3. Pick your repository and branch.
4. Coolify auto-detects the framework (Next.js, Nuxt, Laravel, static, Dockerfile).
5. Set your environment variables.
6. Click **Deploy**.

First deploy takes longer (pulling base images). Subsequent deploys use cached layers.

### Step 6: Add a Database

1. Go to **Projects → New → Database**.
2. Pick PostgreSQL, MySQL, Redis, MongoDB, or others.
3. Coolify provisions it in a container with persistent volumes.
4. Copy the internal connection string into your app's environment variables.

Databases are accessible only within the internal Docker network by default — no public exposure.

## Production Hardening

### Automated Backups

Coolify supports S3-compatible backup destinations:

1. Go to **Settings → Backup**.
2. Add an S3 bucket (AWS S3, Backblaze B2, MinIO, Cloudflare R2).
3. Schedule daily backups of all databases.

Cost: Backblaze B2 charges $0.005/GB/month. A 10 GB database backup costs $0.60/year.

### Resource Limits

Set memory and CPU limits per container to prevent one runaway app from killing everything:

- Node.js apps: 512 MB limit, 256 MB reservation
- Databases: 1 GB limit for PostgreSQL with moderate load
- Build processes: 2 GB limit (Next.js builds are RAM-hungry)

### Monitoring and Alerts

Coolify shows basic metrics in the UI. For production:

- Enable the built-in **Sentinel** monitoring agent.
- Set up notifications via Discord, Slack, Telegram, or email.
- Alert on disk usage above 80% — Docker images accumulate fast.

### Docker Cleanup Cron

Add a weekly cleanup to prevent disk bloat:

```bash
# /etc/cron.weekly/docker-cleanup
#!/bin/bash
docker system prune -af --volumes --filter "until=168h"
```

This removes unused images, containers, and volumes older than 7 days.

## Common Pitfalls and Risks

| Issue | Symptom | Fix |
|---|---|---|
| OOM during builds | Deploy fails silently, container restarted | Increase RAM or set build memory limit |
| Disk full | All deploys fail, databases crash | Enable Docker prune cron, add disk space |
| SSL not issuing | Browser shows insecure warning | Check DNS propagation, ensure ports 80/443 open |
| Slow builds | 10+ minute deploy times | Add swap (2–4 GB), use build caching |
| Single point of failure | Server goes down = all apps down | Set up a second server as worker node |

### When NOT to Use Coolify

- You need 99.99% uptime SLA — single-server Coolify has no automatic failover.
- Your apps need edge/CDN globally — use Cloudflare Pages or Vercel for static/JAMstack.
- You have 50+ microservices — Kubernetes is more appropriate at that scale.
- You do not want to maintain infrastructure at all — pay for a managed PaaS.

## Coolify vs Alternatives: Self-Hosted PaaS Comparison

| Feature | Coolify | Dokku | CapRover | Portainer |
|---|---|---|---|---|
| GUI Dashboard | ✅ Modern UI | ❌ CLI only | ✅ Basic UI | ✅ Docker-focused |
| Git Push Deploy | ✅ | ✅ | ✅ | ❌ |
| Auto SSL | ✅ Let's Encrypt | ✅ | ✅ | Manual |
| Database Management | ✅ One-click | ✅ Plugins | ✅ One-click | Manual |
| Multi-Server | ✅ Worker nodes | ❌ Single server | ✅ Cluster | ✅ Swarm/K8s |
| Build Packs | ✅ Nixpacks + Docker | ✅ Heroku buildpacks | ✅ Docker only | ❌ |
| Team Management | ✅ Roles + projects | ❌ | Basic | ✅ |
| Resource Monitoring | ✅ Built-in | ❌ | Basic | ✅ |
| Learning Curve | Low | Medium | Low | Medium |
| Active Development | Very active (2026) | Maintenance mode | Slow | Active |

Coolify wins for most indie developers because it combines a clean UI with genuine deployment power. Dokku is lighter but lacks a dashboard. CapRover works but development has slowed. Portainer is great for container management but is not a deployment platform.

## Recommended VPS Providers for Coolify

### Best Overall: Hetzner Cloud

- CX22: 4 GB RAM, 2 vCPU, 40 GB — €4.49/mo
- CX32: 8 GB RAM, 4 vCPU, 80 GB — €8.49/mo
- Fast NVMe storage, included 20 TB bandwidth
- EU and US East locations

### Best Budget: Contabo

- Cloud VPS S: 8 GB RAM, 4 vCPU, 200 GB — $6.99/mo
- Massive storage included
- Slower disk I/O than Hetzner but fine for most deployments
- EU, US, Asia locations

### Best US Coverage: Vultr / DigitalOcean

- Multiple US regions with low-latency networking
- Clean API integration
- $12–$24/mo for 2–4 GB RAM instances

### Best for Asia: Vultr (Tokyo/Singapore) or DigitalOcean (Singapore)

- Sub-50ms latency for Asian users
- Same pricing as US regions

## Final Verdict

Coolify is the best self-hosted PaaS option in 2026 for indie developers and small teams. A $7–$16/month Hetzner VPS replaces $50–$150/month in managed PaaS bills once you run 3+ apps. The setup takes 15 minutes, the UI is genuinely good, and the active development pace means bugs get fixed fast.

The real cost is your time: expect 1–2 hours/month on updates, monitoring, and occasional troubleshooting. If that tradeoff works for you, Coolify on a VPS is one of the best infrastructure decisions you can make.

Start with a 4 GB Hetzner instance, deploy your side projects, and scale up only when resource metrics tell you to.
