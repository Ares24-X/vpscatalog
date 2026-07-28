---
title: "How to Host Multiple Websites on a Budget VPS with Cloudflare Tunnel (2026 Guide)"
description: "Learn how to use Cloudflare Tunnel to securely expose multiple websites from a single cheap VPS without opening ports, managing SSL certificates, or revealing your server IP. Step-by-step setup with Docker, Caddy, and real cost breakdowns."
date: 2026-07-28
tags: ["cloudflare", "tunnel", "vps", "self-hosted", "docker", "multi-site", "security", "budget"]
---

One-sentence verdict: Cloudflare Tunnel lets you host dozens of websites on a single $5–$15/month VPS with zero exposed ports, free SSL, automatic DDoS protection, and no need for a static IP — making budget VPS providers viable for serious web hosting.

## Who This Guide Is For

- Freelancers and agencies running 5–20 client sites who want cheap, secure hosting.
- Developers with side projects scattered across too many $5 droplets.
- Self-hosters who want to expose services without poking holes in their firewall.
- Anyone tired of managing Let's Encrypt renewals and nginx SSL configs.

If you only host one static site, Cloudflare Pages is simpler. This guide shines when you need multiple dynamic sites (WordPress, Ghost, custom apps) on one server.

## What Cloudflare Tunnel Actually Does

Traditional hosting: your VPS has a public IP → you open ports 80/443 → DNS points to that IP → attackers scan it.

Cloudflare Tunnel: your VPS makes an outbound-only connection to Cloudflare's network → traffic flows through Cloudflare → your server IP is never exposed → no open inbound ports needed.

The result:

- **No port forwarding** — your firewall can block all inbound traffic
- **No SSL management** — Cloudflare handles edge certificates
- **No static IP required** — works behind NAT, CGNAT, or dynamic IPs
- **Built-in DDoS protection** — traffic hits Cloudflare first
- **Zero Trust ready** — add authentication to any site with one click

The free Cloudflare plan includes unlimited tunnels. You only pay for your VPS.

## Architecture Overview

```
[Visitors] → [Cloudflare Edge] → [Tunnel] → [Your VPS]
                                               ├── site1.com → Docker container (WordPress)
                                               ├── site2.com → Docker container (Ghost)
                                               ├── app.site3.com → Docker container (Node.js)
                                               └── admin.site4.com → Docker container (internal tool)
```

All routing happens inside `cloudflared` on your VPS. Each hostname maps to a local service (port or Unix socket).

## VPS Requirements

You need less than you think. The `cloudflared` daemon uses roughly 30 MB of RAM and negligible CPU.

### Minimum Specs by Site Count

| Sites hosted | RAM needed | vCPU | Storage | Estimated cost |
|---:|---:|---:|---:|---:|
| 1–3 static/light | 1 GB | 1 | 20 GB | $3–$5/mo |
| 3–8 mixed (WordPress + apps) | 2–4 GB | 2 | 40 GB | $5–$10/mo |
| 8–20 dynamic sites | 4–8 GB | 4 | 80 GB | $10–$20/mo |
| 20+ with databases | 8–16 GB | 4–8 | 160 GB | $15–$30/mo |

The bottleneck is usually RAM (WordPress eats 128–256 MB per site with PHP-FPM), not bandwidth or CPU.

### Best Budget VPS Providers for This Setup

| Provider | Best plan | Monthly cost | Specs | Why it fits |
|---|---|---:|---|---|
| RackNerd | KVM 2.5 GB | $5 | 2.5 GB / 2 vCPU / 50 GB | Cheapest entry for multi-site |
| Contabo | VPS S | $5 | 4 GB / 4 vCPU / 50 GB | Most RAM per dollar |
| Hetzner | CX22 | €4 | 4 GB / 2 vCPU / 40 GB | Best network, EU location |
| DigitalOcean | Basic 2 GB | $12 | 2 GB / 1 vCPU / 50 GB | Easiest setup, good docs |
| Vultr | Cloud Compute | $6 | 1 GB / 1 vCPU / 25 GB | 32 global locations |

For this guide, any Linux VPS with 2+ GB RAM works. Ubuntu 22.04/24.04 LTS recommended.

## Step-by-Step Setup

### Step 1: Prepare Your VPS

SSH into your server and update packages:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget gnupg lsb-release
```

Install Docker (we'll containerize everything):

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# Log out and back in, then verify:
docker --version
```

### Step 2: Install Cloudflared

```bash
# Add Cloudflare's GPG key and repo
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update && sudo apt install -y cloudflared
cloudflared --version
```

### Step 3: Authenticate and Create a Tunnel

```bash
# Login (opens a browser link — copy URL if headless)
cloudflared tunnel login

# Create a named tunnel
cloudflared tunnel create multisite

# Note the tunnel UUID printed (e.g., a1b2c3d4-e5f6-...)
```

This creates credentials at `~/.cloudflared/<UUID>.json`.

### Step 4: Configure DNS Routes

For each domain you want to route through the tunnel:

```bash
cloudflared tunnel route dns multisite site1.com
cloudflared tunnel route dns multisite site2.com
cloudflared tunnel route dns multisite app.site3.com
```

This creates CNAME records pointing each domain to `<UUID>.cfargotunnel.com`.

### Step 5: Create the Tunnel Config

Create `~/.cloudflared/config.yml`:

```yaml
tunnel: a1b2c3d4-e5f6-7890-abcd-ef1234567890
credentials-file: /home/your-user/.cloudflared/a1b2c3d4-e5f6-7890-abcd-ef1234567890.json

ingress:
  - hostname: site1.com
    service: http://localhost:8081
  - hostname: site2.com
    service: http://localhost:8082
  - hostname: app.site3.com
    service: http://localhost:8083
  - hostname: admin.site4.com
    service: http://localhost:8084
    originRequest:
      noTLSVerify: true
  # Catch-all (required)
  - service: http_status:404
```

Each `hostname` maps to a local port where your Docker container listens.

### Step 6: Deploy Your Sites with Docker Compose

Create a `docker-compose.yml`:

```yaml
version: "3.8"

services:
  # WordPress site
  wordpress:
    image: wordpress:latest
    ports:
      - "8081:80"
    environment:
      WORDPRESS_DB_HOST: wp-db
      WORDPRESS_DB_USER: wp
      WORDPRESS_DB_PASSWORD: ${WP_DB_PASS}
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wp-data:/var/www/html
    restart: unless-stopped

  wp-db:
    image: mariadb:11
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASS}
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wp
      MYSQL_PASSWORD: ${WP_DB_PASS}
    volumes:
      - wp-db-data:/var/lib/mysql
    restart: unless-stopped

  # Ghost blog
  ghost:
    image: ghost:5
    ports:
      - "8082:2368"
    environment:
      url: https://site2.com
      database__client: sqlite3
    volumes:
      - ghost-data:/var/lib/ghost/content
    restart: unless-stopped

  # Custom Node.js app
  node-app:
    build: ./my-node-app
    ports:
      - "8083:3000"
    restart: unless-stopped

  # Admin panel (Portainer)
  portainer:
    image: portainer/portainer-ce
    ports:
      - "8084:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer-data:/data
    restart: unless-stopped

volumes:
  wp-data:
  wp-db-data:
  ghost-data:
  portainer-data:
```

Start everything:

```bash
docker compose up -d
```

### Step 7: Run the Tunnel as a Service

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared

# Verify it's running
sudo systemctl status cloudflared
```

### Step 8: Lock Down Your Firewall

Since all traffic flows through the tunnel, block everything inbound except SSH:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
```

No port 80 or 443 needed. Your server is invisible to port scanners.

## Adding Zero Trust Authentication

Want to protect admin panels without coding login pages? In the Cloudflare Zero Trust dashboard:

1. Go to **Access → Applications → Add an Application**
2. Select **Self-hosted**, enter your hostname (e.g., `admin.site4.com`)
3. Add a policy: "Allow" → emails ending in `@yourdomain.com`
4. Save

Now anyone visiting `admin.site4.com` hits a Cloudflare login page first. No VPN needed.

## Performance Considerations

### Latency Impact

Cloudflare Tunnel adds 1–5 ms latency compared to direct connections. For most websites, this is unnoticeable. Cloudflare's edge caching can actually make sites faster for global visitors.

### When NOT to Use This Setup

| Scenario | Why it's wrong | Better option |
|---|---|---|
| Gaming servers (low-latency UDP) | Tunnel adds latency, UDP support limited | Direct IP, DDoS-protected provider |
| Large file hosting (100 GB+) | Cloudflare's free plan has soft bandwidth limits | Object storage (S3, R2) |
| Real-time streaming | WebSocket works but adds overhead | Dedicated streaming infra |
| Sites needing origin IP visibility | Tunnel hides real visitor IPs by default | Use `CF-Connecting-IP` header |

### Handling the CF-Connecting-IP

Your backend sees Cloudflare's IP, not the visitor's. Fix this by reading the `CF-Connecting-IP` header:

```nginx
# In nginx (if using it as local reverse proxy)
real_ip_header CF-Connecting-IP;
set_real_ip_from 0.0.0.0/0;
```

For WordPress, install the "Cloudflare" plugin or add to `wp-config.php`:

```php
if (isset($_SERVER['HTTP_CF_CONNECTING_IP'])) {
    $_SERVER['REMOTE_ADDR'] = $_SERVER['HTTP_CF_CONNECTING_IP'];
}
```

## Cost Breakdown: Traditional vs Tunnel Setup

Hosting 10 WordPress sites:

| Approach | Monthly cost | Effort |
|---|---:|---|
| 10 separate $5 VPS instances | $50 | 10 servers to maintain |
| 1 VPS + cPanel/Plesk | $15 + $15 license = $30 | License fees, panel overhead |
| 1 VPS + Cloudflare Tunnel + Docker | $10–$15 | One server, no panels |
| Managed WordPress (Kinsta/WP Engine) | $200+ | Hands-off but expensive |

The tunnel approach saves $15–$185/month while giving you better security than most alternatives.

## Risks and Gotchas

- **Cloudflare dependency**: If Cloudflare goes down, your sites go down. Rare but it happens (2–3 incidents per year).
- **Terms of Service**: Cloudflare's free plan is not designed for serving large media files. Use R2 or external CDN for video/large assets.
- **Debugging is harder**: You can't `curl` your server directly since ports are closed. Use `cloudflared access` for testing.
- **Resource limits**: One VPS running 20 WordPress sites will hit RAM limits during traffic spikes. Monitor with `htop` and set PHP-FPM `pm.max_children` conservatively.
- **Backup responsibility**: You own backups. Set up automated snapshots or `restic` to object storage.

## Monitoring Your Tunnel

Check tunnel health:

```bash
cloudflared tunnel info multisite
```

View live metrics in the Cloudflare dashboard under **Zero Trust → Tunnels**.

Set up basic uptime monitoring with a free tool like UptimeRobot pointing at each domain's public URL.

## Summary

| Step | What you get |
|---|---|
| Budget VPS ($5–$15/mo) | Raw compute and storage |
| Docker Compose | Isolated, reproducible site deployments |
| Cloudflare Tunnel | Zero-port-exposure, free SSL, DDoS protection |
| UFW firewall locked | Server invisible to scanners |
| Zero Trust (optional) | Auth layer without coding |

Total recurring cost for hosting 5–10 sites: **$5–$15/month** with enterprise-grade security.

The traditional approach of renting multiple VPS instances or paying for hosting panels is dead for anyone comfortable with Docker and a terminal. Cloudflare Tunnel turns a single budget VPS into a secure multi-site hosting platform with less ongoing maintenance than nginx + certbot ever required.
