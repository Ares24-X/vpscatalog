---
title: "Deploy Code-Server on a VPS: VS Code in Your Browser (2026 Guide)"
description: "Step-by-step guide to running code-server on a budget VPS. Access a full VS Code IDE from any browser with your own extensions, settings, and terminal. Covers installation, HTTPS setup, resource tuning, and cost comparison vs GitHub Codespaces."
date: 2026-07-23
tags: ["code-server", "vps", "ide", "self-hosted", "developer-tools", "remote-development", "vs-code"]
---

One-sentence verdict: Code-server puts a full VS Code instance in your browser for $5–$12/month on a VPS, giving you a portable development environment that works from any device — including tablets and Chromebooks — without paying $36/month for GitHub Codespaces.

## Who This Guide Is For

- Developers who work from multiple machines and want a consistent environment accessible from any browser.
- Students or hobbyists using Chromebooks, tablets, or low-spec laptops that struggle to run a local IDE.
- Freelancers who need a secure coding environment they can access from client offices without installing anything.
- Teams that want to standardize development environments without managing individual workstations.
- Remote workers in regions with unreliable power — your VPS keeps running even when your laptop dies.

If you already have a powerful laptop and only code from one device, you probably don't need this. A local VS Code install will always feel snappier.

## What Code-Server Actually Is

Code-server is an open-source project by Coder that runs VS Code on a remote server and exposes it through a web browser. It's not a watered-down web editor — it's the real VS Code with:

- Full extension support (most VS Code extensions work, including language servers)
- Integrated terminal with full shell access to the server
- File system access to the remote machine
- Git integration, debugging, and all the features you expect

| Feature | Code-Server (Self-Hosted) | GitHub Codespaces | Gitpod |
|---|---|---|---|
| Monthly cost (2-core) | $5–$12 (VPS cost) | $36/month (60hrs) | $25/month |
| Always-on | Yes | No (auto-stops) | No (auto-stops) |
| Storage | 40–80GB (VPS disk) | 32GB included | 30GB included |
| Custom extensions | All supported | All supported | Most supported |
| Root access | Full | Limited | Limited |
| Data location | Your choice | US/EU (GitHub) | EU (Gitpod) |
| Offline development | No (need internet) | No | No |

The trade-off is clear: you manage the server yourself, but you get unlimited hours, full control, and lower cost.

## Hardware Requirements

Code-server itself is lightweight, but your projects determine resource needs:

| Workload | CPU | RAM | Disk | Estimated Cost |
|---|---|---|---|---|
| Web dev (Node.js, Python scripts) | 1 vCPU | 2 GB | 40 GB | $5–$6/mo |
| Full-stack (Docker, databases, builds) | 2 vCPU | 4 GB | 80 GB | $10–$12/mo |
| Heavy compilation (Rust, C++, large monorepos) | 4 vCPU | 8 GB | 160 GB | $20–$30/mo |
| AI/ML development (model training) | 4+ vCPU | 16 GB+ | 200 GB+ | $40+/mo |

For most web developers, a 2 GB RAM / 1 vCPU instance handles daily work fine. TypeScript projects with language servers benefit from 4 GB.

## VPS Provider Recommendations

Based on price-to-performance for development workloads:

| Provider | Plan | Specs | Monthly Cost | Best For |
|---|---|---|---|---|
| Hetzner CX22 | Cloud | 2 vCPU / 4 GB / 40 GB | €4.35 (~$4.75) | EU-based developers |
| Contabo Cloud VPS S | Cloud | 4 vCPU / 8 GB / 200 GB | $6.99 | Maximum specs per dollar |
| RackNerd 2GB KVM | KVM | 2 vCPU / 2 GB / 40 GB | $4.50 (annual deal) | Budget option |
| DigitalOcean Basic | Droplet | 2 vCPU / 2 GB / 60 GB | $12 | Beginners, good docs |
| Vultr Cloud Compute | Cloud | 1 vCPU / 2 GB / 50 GB | $10 | Global locations |

Hetzner and Contabo win on raw value. DigitalOcean and Vultr charge a premium for better UI, documentation, and support.

## Step-by-Step Deployment

### Prerequisites

- A VPS running Ubuntu 22.04 or 24.04 (Debian also works)
- A domain name pointing to your VPS IP (for HTTPS)
- SSH access to your server

### Step 1: Update and Secure the Server

```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Create a non-root user if you haven't already
sudo adduser coder
sudo usermod -aG sudo coder

# Set up basic firewall
sudo ufw allow OpenSSH
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

### Step 2: Install Code-Server

The official install script handles everything:

```bash
curl -fsSL https://code-server.dev/install.sh | sh
```

This installs the latest stable release and sets up a systemd service. Verify the installation:

```bash
code-server --version
```

### Step 3: Configure Code-Server

Edit the config file:

```bash
mkdir -p ~/.config/code-server
cat > ~/.config/code-server/config.yaml << 'EOF'
bind-addr: 127.0.0.1:8080
auth: password
password: your-strong-password-here
cert: false
EOF
```

Important: We bind to `127.0.0.1` (not `0.0.0.0`) because we'll put Nginx in front for HTTPS. Never expose code-server directly to the internet without TLS.

### Step 4: Start Code-Server as a Service

```bash
sudo systemctl enable --now code-server@$USER
```

Check it's running:

```bash
sudo systemctl status code-server@$USER
```

### Step 5: Set Up Nginx Reverse Proxy with HTTPS

Install Nginx and Certbot:

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

Create an Nginx config:

```bash
sudo cat > /etc/nginx/sites-available/code-server << 'EOF'
server {
    listen 80;
    server_name code.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection upgrade;
        proxy_set_header Accept-Encoding gzip;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF

sudo ln -sf /etc/nginx/sites-available/code-server /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Get an SSL certificate:

```bash
sudo certbot --nginx -d code.yourdomain.com
```

### Step 6: Verify Everything Works

Open `https://code.yourdomain.com` in your browser. You should see the code-server login page. Enter your password and you'll have full VS Code access.

## Performance Tuning

### Increase File Watcher Limit

VS Code needs a high file watcher limit for large projects:

```bash
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Add Swap Space (for 2GB RAM servers)

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Limit Extensions Memory Usage

In your VS Code settings (Settings → search "memory"):

```json
{
  "typescript.tsserver.maxTsServerMemory": 2048,
  "extensions.autoUpdate": false,
  "files.watcherExclude": {
    "**/node_modules/**": true,
    "**/.git/objects/**": true,
    "**/dist/**": true
  }
}
```

## Security Hardening

Running an IDE on the internet requires extra caution:

1. **Use a strong password** — at least 20 characters, randomly generated.
2. **Enable fail2ban** to block brute-force attempts:

```bash
sudo apt install -y fail2ban
sudo cat > /etc/fail2ban/jail.local << 'EOF'
[nginx-http-auth]
enabled = true
maxretry = 5
bantime = 3600
EOF
sudo systemctl restart fail2ban
```

3. **Restrict IP access** (if you have a static IP):

```nginx
# Add to your nginx location block:
allow 203.0.113.0/24;  # Your IP range
deny all;
```

4. **Keep code-server updated**:

```bash
curl -fsSL https://code-server.dev/install.sh | sh
sudo systemctl restart code-server@$USER
```

5. **Consider using SSH tunnel instead** — for maximum security, skip Nginx entirely and access via SSH port forwarding:

```bash
ssh -N -L 8080:127.0.0.1:8080 user@your-server-ip
# Then open http://localhost:8080 in your local browser
```

## Risks and Limitations

| Risk | Mitigation |
|---|---|
| Server goes down = no coding | Set up automated backups, push code to Git frequently |
| Network latency affects typing | Choose a VPS location close to you; use a CDN for static assets |
| Extensions may not all work | ~95% of extensions work; some that need native UI won't |
| Security exposure | Use HTTPS, strong password, fail2ban, and regular updates |
| VPS provider data access | Encrypt sensitive repos with git-crypt if concerned |
| No offline access | Keep a local Git clone as backup for flights/outages |

## Cost Comparison: 12-Month Breakdown

| Solution | Monthly | Annual | What You Get |
|---|---|---|---|
| Code-Server on Hetzner CX22 | $4.75 | $57 | 2 vCPU, 4 GB RAM, 40 GB, always-on |
| Code-Server on Contabo VPS S | $6.99 | $84 | 4 vCPU, 8 GB RAM, 200 GB, always-on |
| GitHub Codespaces (4-core, 60hr/mo) | $36 | $432 | Auto-managed, GitHub integration |
| Gitpod Professional | $25 | $300 | 50hr/mo, team features |
| AWS Cloud9 (t3.small) | ~$18 | ~$216 | 2 vCPU, 2 GB, AWS integration |

Self-hosting saves $200–$375/year compared to managed alternatives, at the cost of managing your own server.

## Extras Worth Setting Up

### Install Common Development Tools

```bash
# Node.js via nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts

# Python
sudo apt install -y python3 python3-pip python3-venv

# Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Git configuration
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### Set Up Automatic Backups

```bash
# Daily backup of workspace to object storage (using rclone)
sudo apt install -y rclone
# Configure rclone with your preferred storage (S3, Backblaze B2, etc.)
# Then add a cron job:
echo "0 3 * * * rclone sync ~/projects remote:code-server-backup" | crontab -
```

## Verdict

Code-server on a budget VPS is the best option if you:

- Need a browser-accessible IDE for under $10/month
- Want always-on availability (no idle timeouts)
- Value full root access and control over your environment
- Work from multiple devices throughout the day

Skip it if you need offline access, are on a team already paying for GitHub Enterprise (Codespaces included), or don't want to manage server updates yourself.

The sweet spot is a Hetzner CX22 or Contabo VPS S — both give you enough power for serious development work at a fraction of what managed cloud IDEs charge.
