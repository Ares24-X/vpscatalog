---
title: "Best VPS for Hosting E-Commerce Stores in 2026"
description: "A practical guide to choosing the right VPS for WooCommerce, Medusa, Saleor, and other self-hosted e-commerce platforms — covering specs, security, performance tuning, and provider recommendations."
date: 2026-07-18
---

One-sentence verdict: Pick Hetzner for the best price-to-performance ratio on self-hosted shops, DigitalOcean for guided first deployments with managed databases, and Vultr for global edge presence when your buyers are spread across continents.

## Why E-Commerce Needs More Than a Basic VPS

Running an online store is not the same as hosting a blog. Every extra second of page load costs conversions. Payment processing demands TLS everywhere and stable uptime. Inventory syncs, order webhooks, image-heavy product pages, and checkout spikes during sales events all push resource requirements beyond what a $3 shared plan can handle reliably.

The real question is not "can I run a store on a VPS?" — it is whether the VPS can survive a flash sale, keep checkout fast under load, and stay secure enough to handle payment data without waking you at 3 AM.

## Who This Guide Is For

- Indie sellers moving off Shopify to reduce monthly fees and own their data
- Developers building headless commerce frontends (Next.js + Medusa, Saleor, etc.)
- Small businesses running WooCommerce who outgrew shared hosting
- Agencies managing multiple client stores on a single infrastructure stack

## Quick Comparison

| Provider | Best fit | Starting config | Main trade-off | CTA |
|---|---|---|---|---|
| Hetzner | Best value for self-hosted shops | 2 vCPU / 4 GB / 40 GB NVMe from ~€6/mo | Less hand-holding, fewer one-click templates | [Start with Hetzner →](https://www.hetzner.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| DigitalOcean | Easiest managed database + droplet combo | 2 vCPU / 2 GB from $18/mo (with managed DB) | Higher cost per resource | [Start with DigitalOcean →](https://www.digitalocean.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| Vultr | Global presence for international shops | 2 vCPU / 4 GB from $24/mo, 32 locations | Support quality varies | [Start with Vultr →](https://www.vultr.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| Linode / Akamai | Reliable mid-range for growing stores | 2 vCPU / 4 GB from $24/mo | Fewer regions than Vultr | [Start with Linode →](https://www.linode.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| Contabo | Cheapest raw resources for dev/staging | 4 vCPU / 8 GB from €6/mo | Inconsistent I/O, not ideal for production checkout | [Start with Contabo →](https://contabo.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |

## Recommended Server Sizes by Platform

| Platform | Minimum VPS | Recommended production | Notes |
|---|---|---|---|
| WooCommerce (< 500 products) | 1 vCPU / 2 GB RAM | 2 vCPU / 4 GB RAM | PHP + MySQL are memory-hungry under load |
| WooCommerce (1000+ products) | 2 vCPU / 4 GB RAM | 4 vCPU / 8 GB RAM | Add Redis object cache, separate DB if possible |
| Medusa.js | 1 vCPU / 2 GB RAM | 2 vCPU / 4 GB RAM | Node.js + PostgreSQL, lighter than WooCommerce |
| Saleor | 2 vCPU / 4 GB RAM | 4 vCPU / 8 GB RAM | Python + PostgreSQL + Celery workers need breathing room |
| PrestaShop | 1 vCPU / 2 GB RAM | 2 vCPU / 4 GB RAM | Similar profile to WooCommerce |

## Price Ranges You Should Expect

- **Starter store** (under 200 products, low traffic): $5–12/month for the VPS alone
- **Growing store** (500–2000 products, daily orders): $18–36/month
- **Busy store** (flash sales, high concurrency, multiple payment gateways): $48–96/month
- **Multi-store agency setup**: $72–150/month with proper isolation

These exclude domain, CDN, email, and backup storage costs. Budget an extra $5–15/month for automated off-site backups — losing order data is not recoverable from an apology email.

## Configuration Recommendations

### Storage and I/O

Product images, thumbnails, and media assets accumulate fast. A store with 1,000 products and 5 images each already needs 10–20 GB just for media. NVMe storage matters for database queries during checkout — spinning disks add latency where it hurts conversion rates.

**Recommendation:** Start with 40 GB NVMe minimum. Offload media to object storage (Cloudflare R2, Backblaze B2, or S3-compatible) once you pass 500 products.

### Memory and Caching

E-commerce platforms run many database queries per page load. A single WooCommerce product page can trigger 50+ queries without caching. Redis or Memcached reduces this dramatically.

**Recommendation:** Always add Redis. For WooCommerce, install Redis Object Cache plugin. For Medusa/Saleor, configure the built-in Redis cache layer. Budget 512 MB–1 GB RAM for Redis alone on busy stores.

### CPU

Checkout, cart calculations, tax computations, and inventory checks are CPU-bound. A single vCPU handles a few concurrent shoppers but buckles during a 50-person flash sale.

**Recommendation:** 2 vCPU minimum for any store expecting more than 10 concurrent buyers. Scale to 4 vCPU before running promotions.

## Security Checklist for Store VPS

E-commerce servers handle payment data and personal information. Cut corners here and you lose customer trust permanently.

1. **TLS everywhere** — Use Let's Encrypt with auto-renewal. No exceptions, not even for staging if it faces the internet.
2. **Firewall rules** — Expose only ports 80, 443, and SSH (on a non-default port). Block everything else.
3. **Automatic security updates** — Enable unattended-upgrades (Ubuntu/Debian) or dnf-automatic (Fedora/RHEL).
4. **Database not exposed** — PostgreSQL/MySQL should listen on localhost only. Never open 3306/5432 to the internet.
5. **SSH key-only auth** — Disable password login. Use fail2ban as a second layer.
6. **PCI compliance note** — If you process cards directly (not via Stripe/PayPal hosted checkout), you need PCI DSS compliance. Most small stores should use hosted payment pages to avoid this burden entirely.
7. **Backups** — Daily automated backups with at least 7-day retention. Test restores monthly.

## Step-by-Step: Deploy a WooCommerce Store on a VPS

This covers the most common path. Adjust for your platform of choice.

### 1. Provision the VPS

Choose 2 vCPU / 4 GB RAM with Ubuntu 24.04 LTS. Enable automatic backups at the provider level if available.

### 2. Initial server hardening

```bash
# Update system
apt update && apt upgrade -y

# Create a non-root user
adduser shopadmin
usermod -aG sudo shopadmin

# Disable root SSH login
sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
systemctl restart sshd

# Enable UFW firewall
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

### 3. Install the LEMP stack

```bash
apt install -y nginx mariadb-server php8.3-fpm php8.3-mysql \
  php8.3-xml php8.3-mbstring php8.3-curl php8.3-zip \
  php8.3-gd php8.3-intl php8.3-redis redis-server

# Secure MariaDB
mysql_secure_installation
```

### 4. Configure PHP for e-commerce

```bash
# Edit /etc/php/8.3/fpm/php.ini
memory_limit = 512M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 300
max_input_vars = 5000
```

### 5. Install WordPress + WooCommerce

```bash
cd /var/www
wget https://wordpress.org/latest.tar.gz
tar xzf latest.tar.gz
chown -R www-data:www-data wordpress
```

Configure Nginx server block, set up the database, run the WordPress installer, then install WooCommerce from the plugin directory.

### 6. Add Redis caching

```bash
# Redis is already installed, enable the WordPress plugin
wp plugin install redis-cache --activate
wp redis enable
```

### 7. Set up SSL with Let's Encrypt

```bash
apt install certbot python3-certbot-nginx
certbot --nginx -d yourstore.com -d www.yourstore.com
```

### 8. Configure automated backups

```bash
# Daily database dump + file backup to object storage
# Use restic, borgbackup, or a simple script with rclone
apt install rclone
# Configure rclone with your backup destination
```

## Risk Warnings

- **Unmanaged means you own the uptime.** If your VPS goes down during a sale, there is no support team fixing it for you at 2 AM. Consider a monitoring service (UptimeRobot, Hetrixtools) and alerting.
- **Payment gateway outages are not your fault but are your problem.** Have a fallback payment method configured.
- **Plugin/extension bloat kills performance.** Every additional WooCommerce plugin adds queries. Audit quarterly.
- **Scaling is manual.** Unlike Shopify, you need to resize or load-balance yourself. Plan capacity before marketing campaigns, not during.
- **Legal compliance varies by region.** GDPR, PCI DSS, cookie consent, and tax collection requirements depend on where you and your customers are located. A VPS does not solve compliance — it just gives you control over how you implement it.

## When NOT to Use a VPS for E-Commerce

A VPS is the wrong choice if:

- You have zero server administration experience and no budget to learn or hire help
- Your store needs 99.99% uptime guarantees with SLA-backed compensation
- You process more than 100,000 orders/month (look at managed Kubernetes or dedicated servers)
- You want to launch in under 24 hours with no technical setup (use Shopify, then migrate later)

## Comparison: Self-Hosted VPS vs Managed Platforms

| Factor | VPS (self-hosted) | Shopify / BigCommerce |
|---|---|---|
| Monthly cost (small store) | $12–36 | $39–299 |
| Setup time | Hours to days | Minutes |
| Customization | Unlimited | Limited by platform |
| Transaction fees | Only payment gateway fees | Platform fee + gateway fee |
| Scaling responsibility | You | Platform |
| Data ownership | Full | Platform-dependent |
| Security responsibility | You | Platform |
| Plugin ecosystem | Open source, unlimited | App store, often paid |

## Final Recommendation

For most self-hosted e-commerce stores in 2026:

1. **Start with Hetzner CX32** (2 vCPU / 4 GB / 40 GB NVMe) at ~€6/month if you are comfortable with Linux administration
2. **Choose DigitalOcean** if you want managed databases and one-click app marketplace convenience
3. **Pick Vultr** if your customers are geographically distributed and you need edge locations in Asia, South America, or Africa
4. **Avoid Contabo for production** — the price is tempting but I/O inconsistency during peak hours can freeze checkout pages

Whichever you choose: enable backups from day one, add Redis caching, put a CDN (Cloudflare free tier works) in front for static assets, and monitor uptime before your first customer notices a problem.
