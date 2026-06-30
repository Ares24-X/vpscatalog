---
title: "How to Deploy Ollama and Local LLMs on a VPS in 2026"
description: "Step-by-step guide to running self-hosted AI models like Llama 3, Mistral, and Phi on a budget VPS using Ollama. Covers RAM requirements, CPU-only inference, model selection, and realistic performance expectations."
date: 2026-06-30
tags: ["ai", "ollama", "llm", "self-hosted", "vps", "llama", "mistral"]
---

One-sentence verdict: you can run small language models on a $10–$30/month VPS with 8–16 GB RAM, but keep expectations realistic — CPU-only inference is slow, and anything above 7B parameters needs serious hardware.

## Who This Guide Is For

- Developers who want a private AI endpoint without sending data to OpenAI or Anthropic.
- Hobbyists experimenting with local LLMs who need a persistent server.
- Small teams building internal tools (code review bots, document Q&A) where latency under 5 seconds is acceptable.
- Privacy-conscious users who cannot use cloud AI APIs due to data residency rules.

If you need real-time chat with sub-second token streaming for end users, this setup is not the answer. You need GPU instances or managed inference APIs for that.

## What Ollama Actually Needs

Ollama is a wrapper that makes running quantized LLMs simple. One command downloads and serves a model. The catch is hardware.

### Minimum Requirements by Model Size

| Model | Parameters | RAM needed (Q4 quantized) | Recommended VPS | Tokens/sec (CPU-only) |
|---|---:|---:|---|---:|
| Phi-3 Mini | 3.8B | 4 GB | 4 GB RAM / 2 vCPU | 8–12 t/s |
| Llama 3.2 3B | 3B | 3.5 GB | 4 GB RAM / 2 vCPU | 10–15 t/s |
| Mistral 7B | 7B | 6 GB | 8 GB RAM / 4 vCPU | 4–7 t/s |
| Llama 3.1 8B | 8B | 6.5 GB | 8 GB RAM / 4 vCPU | 3–6 t/s |
| Llama 3.1 70B | 70B | 42 GB | Not feasible on budget VPS | < 1 t/s |

Rule of thumb: the model file in Q4_K_M quantization is roughly 60% of the parameter count in GB. Your VPS needs that plus 2–3 GB for the OS, Ollama runtime, and your application.

## Provider Comparison for AI Workloads

| Provider | Best plan for Ollama | Monthly cost | RAM / vCPU | Why it works | CTA |
|---|---|---:|---|---|---|
| Hetzner | CX32 or CX42 | €7–€15 | 8–16 GB / 4–8 vCPU | Best price per GB RAM in Europe | [Start with Hetzner →](https://www.hetzner.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| Contabo | VPS M or L | $7–$13 | 8–16 GB / 4–6 vCPU | Cheapest raw RAM, slower CPU | [Start with Contabo →](https://contabo.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| DigitalOcean | Regular 8 GB Droplet | $48 | 8 GB / 4 vCPU | Easy setup, premium pricing | [Start with DigitalOcean →](https://www.digitalocean.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| Vultr | High Frequency 8 GB | $48 | 8 GB / 4 vCPU | Fast NVMe, many regions | [Start with Vultr →](https://www.vultr.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |
| RackNerd | Special 8 GB | $30–$40/year | 8 GB / 4 vCPU | Extreme budget (Black Friday deals) | [Check RackNerd deals →](https://www.racknerd.com/?ref=AFFILIATE_TAG_PLACEHOLDER) |

Budget pick: Hetzner CX32 (8 GB RAM, 4 vCPU, ~€7.50/month) handles 7B models comfortably. If you only need 3B models, Contabo's cheapest 8 GB plan or even a 4 GB Hetzner instance works.

## Step-by-Step Deployment

### 1. Provision the Server

Choose Ubuntu 22.04 or 24.04 LTS. Minimum 8 GB RAM for 7B models.

```bash
# After SSH into your new VPS
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget
```

### 2. Install Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

This installs the Ollama binary and sets up a systemd service. Verify:

```bash
ollama --version
systemctl status ollama
```

### 3. Pull a Model

Start with a small model to test:

```bash
# Lightweight and fast — good for testing
ollama pull phi3

# Better quality, needs 6+ GB free RAM
ollama pull llama3.1:8b-instruct-q4_K_M
```

### 4. Test Locally

```bash
ollama run phi3 "Explain what a VPS is in two sentences."
```

If this works without out-of-memory errors, your server can handle the model.

### 5. Expose the API (Optional)

Ollama serves an OpenAI-compatible API on port 11434 by default, but only on localhost.

To expose it safely:

```bash
# Edit the systemd service to listen on all interfaces
sudo systemctl edit ollama

# Add these lines:
# [Service]
# Environment="OLLAMA_HOST=0.0.0.0"

sudo systemctl restart ollama
```

Then protect it with a reverse proxy and authentication:

```bash
sudo apt install -y nginx apache2-utils

# Create a password file
sudo htpasswd -c /etc/nginx/.htpasswd llmuser

# Nginx config
sudo tee /etc/nginx/sites-available/ollama << 'EOF'
server {
    listen 443 ssl;
    server_name your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    location / {
        auth_basic "LLM API";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://127.0.0.1:11434;
        proxy_set_header Host $host;
        proxy_read_timeout 300s;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/ollama /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 6. Connect from Your App

Use the OpenAI-compatible endpoint:

```python
import openai

client = openai.OpenAI(
    base_url="https://your-domain.com/v1",
    api_key="not-needed-but-required-field",
)

response = client.chat.completions.create(
    model="llama3.1:8b-instruct-q4_K_M",
    messages=[{"role": "user", "content": "Hello"}],
)
print(response.choices[0].message.content)
```

## Performance Tuning Tips

1. **Use quantized models.** Q4_K_M gives the best speed/quality balance. Q5 is slightly better quality but 20% slower.
2. **Set `num_ctx` lower.** Default context window is 2048 tokens. Reducing to 1024 speeds up inference for short tasks.
3. **Limit concurrent requests.** Ollama queues requests by default. Two simultaneous users on a 7B model will cause timeouts.
4. **Add swap space.** If your VPS has exactly enough RAM, a 4 GB swap file prevents OOM kills during model loading.
5. **Monitor with htop.** Watch RSS memory during inference. If it touches 90% of total RAM, downgrade to a smaller model.

```bash
# Add 4 GB swap
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Risks and Honest Limitations

| Risk | Reality | Mitigation |
|---|---|---|
| Slow inference | 7B model on 4 vCPU = 3–7 tokens/second. A 500-word response takes 30–60 seconds. | Use 3B models for speed-sensitive tasks |
| OOM crashes | Model loading spikes RAM briefly above steady-state usage | Add swap, monitor, set limits |
| No GPU acceleration | Budget VPS has no GPU. Inference is pure CPU. | Accept the speed trade-off or rent GPU instances |
| Model quality | Small quantized models hallucinate more than GPT-4 or Claude | Use for low-stakes tasks: drafts, summaries, classification |
| Security exposure | An open Ollama port = anyone can use your server | Always use auth + HTTPS, never expose port 11434 directly |
| Bandwidth spikes | Downloading a 7B model is 4–5 GB | Pull models once, keep them on disk |

## When NOT to Use This Setup

- You need responses under 1 second → use a cloud API (OpenAI, Anthropic, Groq).
- You need 70B+ models → rent a GPU instance (RunPod, Vast.ai, Lambda).
- You have more than 5 concurrent users → you need multiple servers or a managed inference service.
- You are building a product with SLA requirements → this is hobby/internal-tool territory.

## Cost Comparison: Self-Hosted vs API

| Approach | Monthly cost | Speed | Privacy | Best for |
|---|---:|---|---|---|
| Ollama on Hetzner 8 GB | ~$8 | 3–7 t/s | Full control | Internal tools, experiments |
| Ollama on Contabo 16 GB | ~$13 | 5–10 t/s | Full control | Running multiple models |
| OpenAI GPT-4o Mini API | ~$5–$50 (usage) | 50+ t/s | Data sent to OpenAI | Production apps with low volume |
| Groq (Llama 3.1 70B) | Free tier / ~$1–$10 | 200+ t/s | Data sent to Groq | Speed-critical, non-private tasks |

Self-hosting makes sense when: (1) you need data to stay on your server, (2) you have low request volume, and (3) you can tolerate 5–30 second response times.

## Recommended Stack

For most people running Ollama on a budget VPS:

- **Server:** Hetzner CX32 (8 GB / 4 vCPU) — ~€7.50/month
- **OS:** Ubuntu 24.04 LTS
- **Model:** Llama 3.2 3B or Phi-3 Mini for speed; Llama 3.1 8B for quality
- **Reverse proxy:** Nginx + Let's Encrypt + basic auth
- **Monitoring:** htop + simple cron script checking memory usage
- **Backup:** Snapshot the server weekly (models can be re-downloaded)

## Final Checklist

- [ ] VPS has at least 2 GB RAM headroom above model size
- [ ] Swap file configured (4 GB minimum)
- [ ] Ollama installed and model pulled successfully
- [ ] API not exposed to public internet without authentication
- [ ] Tested response time is acceptable for your use case
- [ ] Set up automatic security updates (`unattended-upgrades`)
- [ ] Firewall configured (only ports 22, 80, 443 open)

---

*Last updated: June 2026. Prices and model performance change frequently. Verify current plans on provider websites before purchasing.*
