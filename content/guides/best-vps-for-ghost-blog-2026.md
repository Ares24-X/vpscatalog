---
title: "Best VPS for Hosting a Ghost Blog in 2026"
description: "Ghost is a fast, lean publishing platform that runs well on a modest VPS. This guide covers who should use it, what specs you need, which providers are worth it, and how to avoid common setup mistakes."
date: 2026-07-03
tags: ["ghost", "blog hosting", "vps", "cms", "self-hosted", "nodejs"]
---

One-sentence verdict: a $6–$10/month Hetzner or DigitalOcean VPS with 1 vCPU and 1–2 GB RAM will run Ghost comfortably for a personal blog or small publication under 50,000 monthly visitors.

## Who This Guide Is For

- Bloggers and newsletter writers moving away from WordPress or Substack
- Developers who want full control over their CMS without managed hosting fees
- Small teams launching a publication on a tight budget
- Marketers testing Ghost's built-in membership and email features before committing to Ghost(Pro)

This guide is **not** for people who want zero server maintenance. Ghost requires Node.js, a reverse proxy, SSL, and occasional updates. If you want a fully managed solution, Ghost(Pro) exists and starts at $9/month. This guide is for people who want to run it themselves and keep monthly costs low.

## Price Range

| Setup | Monthly cost estimate |
|---|---:|
| Hetzner CX22 (2 vCPU / 4 GB RAM) | ~$5–6/month |
| DigitalOcean Basic Droplet (1 vCPU / 2 GB RAM) | ~$12/month |
| Vultr Cloud Compute (1 vCPU / 2 GB RAM) | ~$12/month |
| Linode Nanode (1 vCPU / 1 GB RAM) | ~$5/month |
| RackNerd promo KVM (1 vCPU / 1 GB RAM) | ~$12–15/year promo |

Ghost itself is free and open source. You only pay for the VPS and your domain.

## Recommended Configuration

Ghost runs on Node.js and uses either MySQL or SQLite. For a production install, MySQL is recommended. Here is what the server needs:

| Component | Minimum | Comfortable |
|---|---|---|
| CPU | 1 vCPU | 2 vCPU |
| RAM | 1 GB | 2 GB |
| Storage | 20 GB SSD | 40 GB SSD |
| OS | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |
| Node.js | 18.x LTS | 20.x LTS |
| Database | SQLite (dev only) | MySQL 8 |
| Reverse proxy | Nginx | Nginx |

**Memory note:** Ghost needs about 350–500 MB at idle. MySQL adds another 200–400 MB. On a 1 GB RAM server, you will hit the limit if you also run cron jobs, monitoring agents, or fail2ban. Budget 2 GB RAM for a comfortable production setup.

## Quick Provider Picks

| Provider | Best for | Plan | Trade-off | Link |
|---|---|---|---|---|
| Hetzner | Best value globally | CX22 (2 vCPU / 4 GB) | EU/US datacenters only | [Start with Hetzner →](https://www.hetzner.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| DigitalOcean | Beginner-friendly docs | Basic 2 GB Droplet | Costs more than Hetzner | [Start with DigitalOcean →](https://www.digitalocean.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| Vultr | US/Asia edge locations | Cloud Compute 2 GB | Variable performance | [Start with Vultr →](https://www.vultr.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| Linode / Akamai | Stable long-term hosting | Nanode 1 GB or Linode 2 GB | Fewer promo deals | [Start with Linode →](https://www.linode.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| RackNerd | Absolute lowest cost | 1 GB KVM promo | US only, support is slow | [Start with RackNerd →](https://www.racknerd.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |

## Setup Steps (Condensed)

Ghost has an official CLI (`ghost-cli`) that handles most of the install. The outline below assumes Ubuntu 22.04 and Nginx.

### 1. Provision the VPS

Create a new Ubuntu 22.04 instance. Add a non-root sudo user. Enable UFW and allow ports 22, 80, and 443.

```bash
adduser ghostuser
usermod -aG sudo ghostuser
ufw allow OpenSSH
ufw allow 'Nginx Full'
ufw enable
```

### 2. Install dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 20.x
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install Nginx and MySQL
sudo apt install -y nginx mysql-server

# Install Ghost CLI
sudo npm install -g ghost-cli
```

### 3. Create MySQL user for Ghost

```sql
CREATE DATABASE ghostdb;
CREATE USER 'ghostuser'@'localhost' IDENTIFIED BY 'StrongPassHere';
GRANT ALL PRIVILEGES ON ghostdb.* TO 'ghostuser'@'localhost';
FLUSH PRIVILEGES;
```

### 4. Install Ghost

```bash
# Create install directory
sudo mkdir -p /var/www/ghost
sudo chown ghostuser:ghostuser /var/www/ghost
cd /var/www/ghost

# Run the installer (it will ask for domain, database, SSL)
ghost install
```

Ghost CLI will configure Nginx, request an SSL certificate via Let's Encrypt, and create a systemd service automatically.

### 5. Verify the install

```bash
ghost status
ghost ls
```

Visit `https://yourdomain.com/ghost` to complete the admin setup.

## Ghost vs WordPress on VPS: Quick Comparison

| Factor | Ghost | WordPress |
|---|---|---|
| Built-in newsletter/membership | Yes, native | Plugin required |
| Resource usage at idle | Lower (~400 MB RAM) | Higher (PHP + MySQL) |
| Admin UI speed | Fast | Can be slow with plugins |
| Plugin/theme ecosystem | Smaller | Very large |
| Learning curve (self-hosted) | Moderate (Node.js stack) | Low (shared-hosting friendly) |
| SEO tooling built-in | Basic | Extensive (Yoast, RankMath) |
| Multisite support | No | Yes |

**Verdict:** Ghost wins on speed and built-in monetization features. WordPress wins on ecosystem and flexibility. If your use case is writing + email list, Ghost is the leaner choice.

## Ghost vs Cloudflare Pages (Static)

Cloudflare Pages is free and fast, but it only serves static content. Ghost is dynamic, meaning you get:

- Member login and gating
- Built-in email newsletter (via Mailgun or Mailersend)
- Comments (via integrations)
- Dynamic search
- Members-only content

If you only need a blog without membership, a static site on Cloudflare Pages with a headless Ghost backend (Ghost as API, Cloudflare Pages for rendering) is a valid hybrid. But the setup complexity is much higher. For most people, Ghost on a $6/month VPS is the simpler path.

## Risk Reminders

**Backup before every Ghost update.** Major Ghost version upgrades (e.g., v4 → v5 → v5.x) can break themes or integrations. Always snapshot the VPS and back up `/var/www/ghost/content` before running `ghost update`.

**SSL renewal.** Ghost CLI sets up Let's Encrypt auto-renewal via Certbot. Verify it works: `sudo certbot renew --dry-run`. A lapsed SSL cert will take your blog offline.

**MySQL tuning.** On a 1 GB server, MySQL's default config consumes too much memory. Add `innodb_buffer_pool_size = 64M` to `/etc/mysql/mysql.conf.d/mysqld.cnf` to reduce RAM usage.

**Email deliverability.** Ghost's built-in email uses an SMTP provider (Mailgun, Postmark, SendGrid). Do not try to run Postfix on the same VPS for newsletters — deliverability will be poor. Budget $0–$10/month for an email API depending on list size.

**Node.js version pinning.** Ghost is strict about supported Node.js versions. Check the [Ghost compatibility matrix](https://ghost.org/docs/faq/node-versions/) before every Node.js upgrade.

## Who Should Not Self-Host Ghost

- You expect more than 100,000 monthly visitors without a CDN in front
- You do not want to handle Linux server maintenance and updates
- You need WooCommerce or other WordPress e-commerce features
- You want a large marketplace of themes and plugins

For high-traffic use, put Cloudflare in front of your Ghost VPS. Ghost handles traffic well; the bottleneck is usually MySQL, not Ghost itself.

## Final Recommendation

For a personal blog or small publication, **Hetzner CX22 at ~$6/month** is the best combination of price and performance. You get 2 vCPU, 4 GB RAM, and 40 GB SSD — well above what Ghost needs — with room to add more services later.

If you are in North America and want simpler onboarding documentation, **DigitalOcean's 2 GB Basic Droplet** is the easier starting point despite costing about twice as much.

Either way, Ghost on a VPS is a legitimate alternative to Ghost(Pro) once you are comfortable with a Linux command line.
