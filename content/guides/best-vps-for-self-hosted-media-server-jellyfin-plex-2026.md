---
title: "Best VPS for Self-Hosted Media Server (Jellyfin & Plex) in 2026"
description: "Complete guide to running Jellyfin or Plex on a VPS. Covers hardware requirements, storage options, transcoding needs, provider comparison, and step-by-step deployment with Docker."
date: 2026-07-27
tags: ["vps", "jellyfin", "plex", "media server", "self-hosted", "streaming", "docker"]
---

One-sentence verdict: Hetzner's CX32 (4 vCPU, 8 GB RAM, €7.59/mo) with a Storage Box add-on is the best value for a personal Jellyfin media server, while Contabo's Storage VPS suits large libraries on a tight budget — but skip transcoding-heavy setups unless you are paying for dedicated CPU.

## Who This Guide Is For

- Cord-cutters who want their own Netflix-like streaming service accessible anywhere.
- Privacy-focused users who do not want media libraries scanned by third-party cloud services.
- Expats and travelers who need remote access to a media collection without relying on home internet upload speeds.
- Small families or friend groups (2–5 simultaneous streams) sharing a personal library.
- Self-hosters already running other services on a VPS who want to add media streaming.

## Who Should Not Use a VPS for Media Streaming

- Anyone with 10+ simultaneous users — you need dedicated hardware or a seedbox with Plex support.
- Users whose library exceeds 20 TB — VPS storage costs become absurd at that scale; use a dedicated server.
- People who require 4K HDR transcoding — no affordable VPS CPU can handle this in real-time.
- Users comfortable with a home server and decent upload speed (50+ Mbps) — a local setup is cheaper long-term.

## Jellyfin vs Plex: Which to Deploy on a VPS

| Feature | Jellyfin | Plex |
|---|---|---|
| Cost | Completely free | Free tier + Plex Pass ($5/mo or $120 lifetime) |
| Transcoding | Software-only on VPS (no GPU) | Hardware transcoding requires Plex Pass |
| Remote access | Built-in, no account required | Requires Plex account, routes through Plex servers |
| Privacy | Fully self-contained | Plex phones home, collects metadata |
| Client apps | Web, Android, iOS (third-party), Roku, Fire TV | Polished native apps on all platforms |
| Resource usage | Lighter on idle | Higher baseline RAM usage |
| Best for VPS | Yes — no external dependencies | Works, but Plex account requirement adds friction |

**Recommendation for VPS deployment:** Jellyfin. It is lighter, fully open-source, requires no external account, and does not route discovery through third-party servers. Plex works fine too, but you are paying for infrastructure while also depending on Plex's servers for authentication.

## Hardware Requirements

### Minimum Specs by Use Case

| Scenario | vCPU | RAM | Storage | Bandwidth | Monthly Cost Range |
|---|---:|---:|---:|---:|---:|
| 1 user, direct play only | 2 | 2 GB | 50 GB+ | 2 TB | $5–8/mo |
| 2–3 users, occasional transcode | 4 | 4 GB | 100 GB+ | 4 TB | $8–15/mo |
| 4–5 users, regular transcoding | 6 | 8 GB | 200 GB+ | 8 TB | $15–30/mo |
| Large library, metadata-heavy | 4 | 8 GB | 500 GB+ | Unmetered | $20–40/mo |

### Key Resource Considerations

**CPU:** Each 1080p transcode stream needs roughly 2,000 PassMark score. A typical 4-vCPU VPS scores 4,000–6,000, handling 1–2 simultaneous transcodes. Direct play (client supports the file format natively) uses almost zero CPU.

**RAM:** Jellyfin idles at ~200 MB. Each active stream adds 100–300 MB. Metadata scanning for large libraries (5,000+ items) can spike to 2–3 GB temporarily.

**Storage:** This is the real constraint on a VPS. Options:
- Local SSD/NVMe — fast but expensive per GB ($0.10–0.20/GB/mo on most providers).
- Block storage add-ons — cheaper ($0.02–0.05/GB/mo) but slower.
- Object storage (S3-compatible) via rclone mount — cheapest for bulk media but adds latency.
- Hetzner Storage Box — excellent middle ground at €3.81/mo for 1 TB (BX11).

**Bandwidth:** A single 1080p stream uses 5–15 Mbps. A 2-hour movie at 8 Mbps consumes about 7 GB. Budget 500 GB–1 TB per active user per month.

## VPS Provider Comparison for Media Servers

| Provider | Plan | vCPU | RAM | Storage | Bandwidth | Price | Best For |
|---|---|---:|---:|---:|---:|---:|---|
| Hetzner CX32 | Cloud | 4 | 8 GB | 80 GB NVMe | 20 TB | €7.59/mo | Best overall value |
| Hetzner CX32 + BX21 | Cloud + Storage Box | 4 | 8 GB | 80 GB + 5 TB | 20 TB | €15.78/mo | Large libraries |
| Contabo Storage VPS M | VPS | 6 | 16 GB | 800 GB SSD | 32 TB | €10.49/mo | Maximum storage per dollar |
| Netcup RS 2000 G11 | Root Server | 6 | 16 GB | 256 GB SSD | 80 TB | €10.68/mo | Heavy bandwidth users |
| HostHatch Storage NVMe | VPS | 4 | 8 GB | 250 GB NVMe | 5 TB | $10/mo | NVMe speed + storage |
| Vultr High Frequency | Cloud | 4 | 8 GB | 128 GB NVMe | 5 TB | $48/mo | Low-latency streaming |

### Why Hetzner Wins for Most Users

- Storage Boxes are mountable via CIFS/SMB or SSH/SFTP — Jellyfin reads directly from them.
- European data centers with excellent peering to North America and Asia.
- 20 TB included bandwidth is generous for personal media streaming.
- Consistent CPU performance (dedicated cores on CX series).

### When Contabo Makes Sense

- You have a large library (500 GB+) and want everything on local disk.
- You accept inconsistent CPU performance during peak hours (shared cores).
- Budget is the primary constraint.

## Step-by-Step Deployment: Jellyfin on Docker

### Prerequisites

- A VPS running Ubuntu 22.04+ or Debian 12+.
- A domain name (optional but recommended for HTTPS).
- SSH access with sudo privileges.

### Step 1: Initial Server Setup

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Install Docker Compose
sudo apt install docker-compose-plugin -y

# Create media directories
sudo mkdir -p /opt/media/{movies,shows,music}
sudo mkdir -p /opt/jellyfin/{config,cache}
sudo chown -R 1000:1000 /opt/media /opt/jellyfin
```

### Step 2: Docker Compose Configuration

Create `/opt/jellyfin/docker-compose.yml`:

```yaml
version: "3.8"
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    ports:
      - "8096:8096"
    volumes:
      - /opt/jellyfin/config:/config
      - /opt/jellyfin/cache:/cache
      - /opt/media:/media:ro
    environment:
      - JELLYFIN_PublishedServerUrl=https://media.yourdomain.com
    deploy:
      resources:
        limits:
          memory: 4G
```

### Step 3: Start and Configure

```bash
cd /opt/jellyfin
docker compose up -d

# Check logs
docker compose logs -f jellyfin
```

Access the setup wizard at `http://YOUR_SERVER_IP:8096` and configure:
1. Create an admin account.
2. Add media libraries pointing to `/media/movies`, `/media/shows`, etc.
3. Set preferred language and metadata providers.
4. Configure remote access (enable automatic port mapping if behind NAT).

### Step 4: Reverse Proxy with Caddy (HTTPS)

```bash
# Install Caddy
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install caddy -y
```

Create `/etc/caddy/Caddyfile`:

```
media.yourdomain.com {
    reverse_proxy localhost:8096
    encode gzip
}
```

```bash
sudo systemctl restart caddy
```

### Step 5: Mount External Storage (Hetzner Storage Box Example)

```bash
# Install CIFS utilities
sudo apt install cifs-utils -y

# Create mount point
sudo mkdir -p /mnt/storagebox

# Add to /etc/fstab for persistent mount
echo "//uXXXXXX.your-storagebox.de/backup /mnt/storagebox cifs credentials=/etc/smbcredentials,uid=1000,gid=1000,iocharset=utf8 0 0" | sudo tee -a /etc/fstab

# Create credentials file
echo -e "username=uXXXXXX\npassword=YOUR_PASSWORD" | sudo tee /etc/smbcredentials
sudo chmod 600 /etc/smbcredentials

# Mount
sudo mount -a
```

Then update your Docker Compose to include `/mnt/storagebox:/media/external:ro`.

## Performance Optimization Tips

### 1. Avoid Transcoding When Possible

Configure client apps to direct play. Most modern devices handle H.264/AAC natively. Transcoding is the biggest CPU killer.

Set Jellyfin's transcoding policy to "allow" but educate users to choose "Original" quality in client settings.

### 2. Use Hardware-Accelerated Tone Mapping

Even without GPU, you can enable VAAPI or QSV if the VPS host exposes it (rare on cloud VPS, common on dedicated servers):

```yaml
# Add to docker-compose.yml under jellyfin service
devices:
  - /dev/dri:/dev/dri
```

### 3. Optimize Metadata Scanning

- Schedule library scans during off-peak hours.
- Disable real-time monitoring if your library does not change frequently.
- Use NFO files for metadata instead of online lookups to reduce API calls.

### 4. Cache Aggressively

Allocate 2–4 GB for Jellyfin's transcoding cache. Fast NVMe for the cache directory reduces buffering during partial transcodes.

## Storage Cost Comparison

| Solution | 1 TB Cost/mo | 5 TB Cost/mo | Speed | Best For |
|---|---:|---:|---|---|
| Hetzner Storage Box | €3.81 | €15.12 | ~100 MB/s via SMB | Large libraries, reliable |
| Contabo local SSD | Included (800 GB plan) | ~€15 (upgrade) | NVMe speed | All-in-one simplicity |
| Backblaze B2 + rclone | $6 | $30 | Variable (cache-dependent) | Archival, rarely accessed |
| Wasabi + rclone | $7 | $35 | Good with local cache | No egress fees |
| Local VPS NVMe (Hetzner) | ~€4.60 (extra volume) | ~€23 | Native NVMe | Metadata, cache, hot files |

## Risk Factors and Honest Warnings

**CPU contention on shared VPS:** During peak hours, your transcoding performance can drop 30–50% on providers like Contabo or OVH Starter. If you need consistent transcode performance, pay for dedicated-core plans.

**Bandwidth overages:** A family of 4 streaming nightly can easily hit 3–5 TB/month. Check your provider's overage policy. Hetzner charges €1.19/TB overage; Vultr charges $0.01/GB (that is $10/TB).

**Data durability:** VPS storage is not backup. Keep a local copy of irreplaceable media. Cloud VPS disks can fail, and most providers do not guarantee recovery.

**Legal considerations:** Self-hosting media you own (ripped DVDs, purchased downloads, recorded content) is legal in most jurisdictions. Distributing copyrighted content is not. A media server accessible only to your household is generally fine; sharing with dozens of strangers is legally risky.

**Single point of failure:** Unlike Plex's relay fallback, a Jellyfin VPS that goes down means zero streaming until it is back. Consider basic monitoring (UptimeRobot, free tier) and automated Docker restart policies.

## Monthly Budget Examples

### Budget Setup (~$12/mo)
- Hetzner CX22 (2 vCPU, 4 GB RAM): €4.59
- Hetzner Storage Box BX11 (1 TB): €3.81
- Domain: ~$1/mo (annual billing)
- Supports: 1–2 direct play streams, 500 GB library

### Recommended Setup (~$20/mo)
- Hetzner CX32 (4 vCPU, 8 GB RAM): €7.59
- Hetzner Storage Box BX21 (5 TB): €8.19
- Domain + Cloudflare: ~$1/mo
- Supports: 3–4 streams (1 transcode + 2–3 direct play), 3 TB library

### Power User Setup (~$35/mo)
- Netcup RS 2000 G11 (6 vCPU, 16 GB RAM): €10.68
- Hetzner Storage Box BX31 (10 TB): €15.12
- Monitoring + domain: ~$2/mo
- Supports: 5+ streams, multiple transcodes, 8 TB library

## Final Verdict

For most self-hosters, the Hetzner CX32 + Storage Box combination hits the sweet spot of price, performance, and storage flexibility. Start with Jellyfin in Docker, use Caddy for HTTPS, and only scale up when you actually hit performance limits.

Do not over-provision. A 2-vCPU machine handling direct play for 1–2 users is perfectly adequate. Add CPU and RAM only when transcoding demand proves it necessary.

The entire setup — server, storage, domain, HTTPS — can run for under $15/month and replace multiple streaming subscriptions while giving you complete control over your media library.
