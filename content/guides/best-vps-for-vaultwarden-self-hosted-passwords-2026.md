---
title: "Best VPS for Vaultwarden (Self-Hosted Bitwarden) in 2026"
description: "A practical guide to choosing and setting up a VPS for Vaultwarden — the lightweight, self-hosted Bitwarden-compatible password manager. Covers server requirements, provider picks, deployment steps, and security hardening."
date: 2026-07-07
tags: ["vaultwarden", "self-hosted", "password manager", "bitwarden", "vps security", "lightweight vps", "privacy"]
---

One-sentence verdict: Vaultwarden runs comfortably on the cheapest VPS you can find — 512 MB RAM is enough — but the real decision is picking a provider with reliable uptime, good peering to your location, and a clean IP reputation.

## Who This Guide Is For

- Privacy-conscious users who want full control over their password vault.
- Small teams (2–20 people) tired of paying per-seat fees for Bitwarden Cloud or 1Password.
- Developers and sysadmins who already manage a VPS and want to consolidate services.
- Anyone in a region where cloud password managers are blocked or unreliable.

## Who Should Stick with Bitwarden Cloud

- Non-technical users who cannot commit to regular server maintenance.
- Organizations subject to compliance audits that require vendor-managed infrastructure.
- Anyone without a backup strategy they trust — losing your password vault is catastrophic.

## Minimum Server Requirements

Vaultwarden is written in Rust and uses a fraction of the resources that official Bitwarden Server requires.

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 1 vCPU | 1 vCPU |
| RAM | 256 MB | 512 MB – 1 GB |
| Storage | 1 GB | 5–10 GB (with attachment storage) |
| OS | Any Linux with Docker | Ubuntu 22.04/24.04 or Debian 12 |
| Network | Static IPv4, ports 443/80 | Low-latency to your primary location |

For context: Vaultwarden idles at around 20–40 MB RAM with a few hundred entries. Even with 50 users and 10,000+ entries, it rarely exceeds 150 MB.

## Best VPS Providers for Vaultwarden

### Quick Comparison

| Provider | Plan | Price | RAM | Storage | Location Coverage | Why It Works |
|----------|------|-------|-----|---------|-------------------|--------------|
| RackNerd | 768 MB KVM | $1.50/mo | 768 MB | 15 GB SSD | US (multiple DCs) | Cheapest reliable option for personal use |
| Ionos | VPS Linux XS | $2/mo | 1 GB | 10 GB NVMe | US, EU | Clean IPs, good EU peering, easy setup |
| Hetzner | CX22 | €3.79/mo | 2 GB | 20 GB NVMe | Germany, Finland, US | Best price-to-performance in Europe |
| BuyVM | Slice 512 | $2/mo | 512 MB | 10 GB SSD | US, Luxembourg | DDoS protection included, privacy-friendly |
| Vultr | Cloud Compute | $6/mo | 1 GB | 25 GB NVMe | 32 locations worldwide | Best location diversity for low latency |

### Detailed Picks

**Best for personal use (1–3 users): RackNerd 768 MB**

RackNerd frequently runs promotional deals that get you 768 MB RAM and 15 GB SSD for around $1.50/month billed annually. Performance is adequate for Vaultwarden with plenty of headroom. The downside: support is slow, and you are responsible for all server management.

**Best for EU-based users: Hetzner CX22**

Hetzner offers excellent network performance within Europe and the CX22 gives you 2 GB RAM — enough to run Vaultwarden alongside a reverse proxy and monitoring stack. Their Nuremberg and Helsinki data centers have strong connectivity to most of Europe. Note: Hetzner requires identity verification for new accounts.

**Best for teams (5–20 users): Ionos VPS Linux S**

At $4/month for 2 GB RAM, Ionos provides a stable platform with automated backups available as an add-on. Their control panel makes initial setup straightforward, and their IPs tend to have clean reputations — important if you are also running email notifications from the same server.

**Best for privacy: BuyVM**

BuyVM accepts cryptocurrency, does not require extensive identity verification, and includes DDoS protection. Their Luxembourg location is good for EU privacy compliance. The 512 MB plan is tight but workable for Vaultwarden alone.

## Deployment Steps

### Prerequisites

- A domain name (e.g., `vault.yourdomain.com`)
- DNS pointed to your VPS IP address
- SSH access to your server

### Step 1: Initial Server Setup

```bash
# Update system
apt update && apt upgrade -y

# Create a non-root user
adduser vaultadmin
usermod -aG sudo vaultadmin

# Configure firewall
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

### Step 2: Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker vaultadmin
```

### Step 3: Deploy Vaultwarden with Docker Compose

Create `/opt/vaultwarden/docker-compose.yml`:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: "https://vault.yourdomain.com"
      SIGNUPS_ALLOWED: "false"
      ADMIN_TOKEN: "" # Generate with: openssl rand -base64 48
      SMTP_HOST: "smtp.yourdomain.com"
      SMTP_FROM: "vault@yourdomain.com"
      SMTP_PORT: "587"
      SMTP_SECURITY: "starttls"
      SMTP_USERNAME: "vault@yourdomain.com"
      SMTP_PASSWORD: "your-smtp-password"
    volumes:
      - ./data:/data
    ports:
      - "127.0.0.1:8080:80"
```

### Step 4: Set Up Reverse Proxy with Caddy

```bash
apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update && apt install caddy
```

Create `/etc/caddy/Caddyfile`:

```
vault.yourdomain.com {
    reverse_proxy localhost:8080
}
```

```bash
systemctl restart caddy
```

### Step 5: Start Vaultwarden

```bash
cd /opt/vaultwarden
docker compose up -d
```

### Step 6: Create Your Account

1. Temporarily set `SIGNUPS_ALLOWED: "true"` in docker-compose.yml.
2. Restart: `docker compose up -d`
3. Visit `https://vault.yourdomain.com` and create your account.
4. Set `SIGNUPS_ALLOWED: "false"` again and restart.

## Security Hardening Checklist

These steps are non-negotiable for a password manager:

- **Disable signups** after creating accounts — prevents unauthorized registrations.
- **Enable fail2ban** for SSH and optionally for Vaultwarden login attempts.
- **Automated backups** — at minimum, back up `/opt/vaultwarden/data/` daily to an offsite location.
- **Unattended upgrades** — enable automatic security patches for the OS.
- **Admin panel protection** — use a long random `ADMIN_TOKEN` and consider disabling the admin panel entirely after initial setup by leaving `ADMIN_TOKEN` empty.
- **SSH key-only auth** — disable password authentication in `/etc/ssh/sshd_config`.
- **Regular Vaultwarden updates** — check for new releases monthly or use Watchtower for automated container updates.

```bash
# Example: backup script (/opt/vaultwarden/backup.sh)
#!/bin/bash
TIMESTAMP=$(date +%Y%m%d_%H%M)
tar -czf /tmp/vaultwarden-backup-$TIMESTAMP.tar.gz /opt/vaultwarden/data/
# Upload to your preferred offsite storage
rclone copy /tmp/vaultwarden-backup-$TIMESTAMP.tar.gz remote:backups/vaultwarden/
rm /tmp/vaultwarden-backup-$TIMESTAMP.tar.gz
```

## Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|------------|
| Server compromise exposes all passwords | Critical | Keep OS patched, minimize attack surface, use fail2ban |
| Data loss from disk failure | Critical | Automated offsite backups (test restores quarterly) |
| VPS provider goes offline | High | Export vault regularly, keep local backup copy |
| Forgot admin token or SSH key | Medium | Store recovery credentials in a separate, offline location |
| Docker image supply chain attack | Medium | Pin image versions, verify checksums, update deliberately |

## Cost Breakdown: Self-Hosted vs Cloud

| | Vaultwarden on VPS | Bitwarden Cloud (Teams) | 1Password Business |
|---|---|---|---|
| 10 users | $2–6/mo (VPS cost) | $40/mo | $80/mo |
| 20 users | $2–6/mo (VPS cost) | $80/mo | $160/mo |
| Features | Full Bitwarden client compatibility | Full official feature set | Different ecosystem |
| Maintenance | You handle updates, backups, uptime | Vendor managed | Vendor managed |
| Data location | Your choice | US/EU cloud | US/Canada |

The cost savings are significant for teams, but factor in your time for maintenance — roughly 1–2 hours per month for updates and monitoring.

## When to Upgrade Your VPS

Vaultwarden is so lightweight that you will likely never need to upgrade for the password manager alone. Consider upgrading if:

- You add other services (monitoring, email, personal wiki) to the same server.
- Attachment storage grows beyond your disk allocation.
- You need better network performance for users in multiple continents.

## Bottom Line

For a personal Vaultwarden instance, grab whatever KVM VPS you can find for $1.50–3/month with at least 512 MB RAM. For a team setup, spend $4–6/month on a provider with automated backup options and good uptime history. The deployment takes 15–20 minutes, and the ongoing maintenance is minimal — but take backups seriously, because this is the one self-hosted service where data loss is unacceptable.
