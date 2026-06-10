---
title: "How to Deploy WordPress on a VPS"
description: "Step-by-step guide to installing WordPress on a VPS. Works on DigitalOcean, Vultr, Linode, or any cloud VPS."
date: 2026-06-10
---

## Deploy WordPress on a VPS in 10 Minutes

This guide works on any VPS provider. I'll show you the fastest way using a one-click app or manual setup.

### Option 1: One-Click App (2 minutes)

Most VPS providers offer a WordPress one-click image:

1. Create a new VPS instance
2. Select "WordPress" from the one-click apps
3. Choose your plan and data center
4. Deploy and grab your login URL from the server console

Works on: DigitalOcean, Vultr, Linode (varies by provider)

### Option 2: Manual LEMP Stack (10 minutes)

SSH into your VPS and run these commands:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Nginx, MySQL, PHP
sudo apt install nginx mysql-server php-fpm php-mysql php-curl php-gd php-xml php-mbstring php-zip -y

# Download WordPress
cd /var/www/html
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzf latest.tar.gz
sudo mv wordpress/* .
sudo rm -rf wordpress latest.tar.gz

# Set permissions
sudo chown -R www-data:www-data /var/www/html

# Create database
sudo mysql -e "CREATE DATABASE wordpress;"
sudo mysql -e "CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'your-password-here';"
sudo mysql -e "GRANT ALL ON wordpress.* TO 'wpuser'@'localhost';"
sudo mysql -e "FLUSH PRIVILEGES;"
```

Then visit your server's IP address and follow the WordPress setup wizard.

### Which Option is Better?

| | One-Click | Manual |
|---|-----------|--------|
| Time | 2 min | 10 min |
| Control | Limited | Full |
| Learning | None | You learn Linux |
| Best for | Quick setup | Long-term projects |

For a production site, go manual or use a managed WP host. For testing or learning, one-click is fine.
