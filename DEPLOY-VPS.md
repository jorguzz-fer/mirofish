# Deploying MiroFish on a VPS (OpenAI-compatible LLM + nginx/HTTPS)

This guide runs MiroFish in Docker behind an nginx reverse proxy with a free
Let's Encrypt certificate. It uses a cheap **OpenAI-compatible** LLM provider
(DeepSeek, Groq, OpenRouter, Qwen, …) — no API key from OpenAI/Anthropic needed.

**Assumptions:** a fresh Ubuntu 22.04/24.04 VPS, a domain you control, and `sudo` access.

---

## 1. Point your domain at the VPS

Create a DNS **A record**: `your-domain.com → <VPS public IP>`. Wait for it to
propagate (`dig +short your-domain.com` should return the VPS IP).

---

## 2. Install Docker + nginx + certbot

```bash
# Docker Engine + compose plugin
curl -fsSL https://get.docker.com | sudo sh

# nginx + certbot
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx
```

---

## 3. Get the code

```bash
sudo mkdir -p /opt && cd /opt
sudo git clone https://github.com/jorguzz-fer/mirofish.git
cd mirofish
sudo git checkout claude/fervent-gauss-wcto6y   # or main, once merged
```

---

## 4. Configure the LLM provider

```bash
sudo cp .env.vps.example .env
sudo nano .env
```

Uncomment ONE provider block and paste your API key. Cheapest options:

| Provider   | Base URL                                   | Example model              | Notes                |
|------------|--------------------------------------------|----------------------------|----------------------|
| DeepSeek   | `https://api.deepseek.com/v1`              | `deepseek-chat`            | Cheap, strong        |
| Groq       | `https://api.groq.com/openai/v1`           | `llama-3.3-70b-versatile`  | Free tier, very fast |
| OpenRouter | `https://openrouter.ai/api/v1`             | `deepseek/deepseek-chat`   | One key, many models |
| Qwen       | `https://dashscope-intl.aliyuncs.com/compatible-mode/v1` | `qwen-plus`  | Alibaba              |

> Simulations consume a LOT of tokens. Keep `OASIS_DEFAULT_MAX_ROUNDS` low (5)
> until you've measured cost on your provider.

---

## 5. Build and start the app

```bash
sudo docker compose -f docker-compose.vps.yml up -d --build
```

The container binds to `127.0.0.1:5001` (not public). Verify locally:

```bash
curl -s http://127.0.0.1:5001/api/graph/project/list   # -> {"count":0,"data":[],"success":true}
sudo docker compose -f docker-compose.vps.yml logs -f   # watch startup
```

---

## 6. nginx reverse proxy + HTTPS

```bash
# Install the server block
sudo cp deploy/nginx/mirofish.conf /etc/nginx/sites-available/mirofish.conf
sudo sed -i 's/your-domain.com/REAL_DOMAIN_HERE/' /etc/nginx/sites-available/mirofish.conf
sudo ln -s /etc/nginx/sites-available/mirofish.conf /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default

sudo nginx -t && sudo systemctl reload nginx

# Issue + install the Let's Encrypt cert (rewrites the block to add HTTPS)
sudo certbot --nginx -d REAL_DOMAIN_HERE
```

Certbot adds the `443` server, the HTTP→HTTPS redirect, and a renewal timer.
Open **https://REAL_DOMAIN_HERE** — MiroFish should load.

---

## 7. Firewall (optional but recommended)

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'   # 80 + 443
sudo ufw enable
```

Port 5001 stays loopback-only, so it's never exposed directly.

---

## Operations

| Task            | Command                                                                |
|-----------------|------------------------------------------------------------------------|
| Update          | `cd /opt/mirofish && sudo git pull && sudo docker compose -f docker-compose.vps.yml up -d --build` |
| Logs            | `sudo docker compose -f docker-compose.vps.yml logs -f`                |
| Restart         | `sudo docker compose -f docker-compose.vps.yml restart`                |
| Stop            | `sudo docker compose -f docker-compose.vps.yml down`                   |
| Cert renewal    | automatic (certbot timer); test with `sudo certbot renew --dry-run`    |

**Persistent data** lives in `backend/uploads` (sessions, simulations) and
`backend/data` (KuzuDB graph) — both mounted as volumes, so they survive
rebuilds. Back these up.

---

## Sizing

OASIS pulls in the full ML stack (PyTorch, scikit-learn). Give the VPS at
least **2 vCPU / 4 GB RAM** and ~10 GB free disk for the image + data.
Simulations are CPU- and token-bound; more rounds/agents = more of both.

---

## Troubleshooting

### "no available server" / Traefik can't reach the app (esp. on Coolify)

**Symptom:** the app container is healthy (logs show "Backend started
successfully"; `python -c "import urllib.request as u; print(u.urlopen('http://127.0.0.1:5001/').status)"`
returns `200` inside the container) and the proxy labels point at port `5001`,
but the public URL returns **"no available server"**.

**Check the DNS record FIRST.** The most common cause is the simplest one:
the domain's **A record points to the wrong IP** (an old server, a typo, or a
proxy in front). The request never reaches your VPS's Traefik, so there is no
backend → "no available server", and the ACME challenge below also fails.
Confirm `dig +short your-domain.com` returns the VPS's **real public IPv4**
before debugging anything else. (If you run **more than one VPS**, double-check
the record points to the box that actually runs Coolify/this app — pointing at
the wrong server is an easy and very common mistake.)

**Second cause: the domain is proxied through Cloudflare (orange
cloud).** Check the proxy/ACME logs — if you see ACME HTTP-01 challenge
failures with a `404` from an IP in the `2606:4700::/32` range, that IP is
**Cloudflare**, not your VPS:

```
ERR Unable to obtain ACME certificate ... Invalid response from
http://your-domain/.well-known/acme-challenge/...: 404
... 2606:4700:3034::...   <- Cloudflare IP
```

What happens: Let's Encrypt's validation request hits Cloudflare → 404 (it
never reaches Traefik) → no certificate is issued → the HTTPS router has no
valid TLS → Traefik replies **"no available server"**. The app was never the
problem; it's DNS.

**Fix (in the Cloudflare dashboard):**
1. DNS → find the record for the host (e.g. `crowd`).
2. Switch it from **Proxied (orange 🟠)** to **DNS only (grey ⚪)**.
3. Ensure it's an **A** record pointing to the VPS **IPv4**. If a proxied
   **AAAA** (IPv6) record exists, remove it (or point it at the VPS's real
   IPv6) — otherwise the ACME challenge keeps hitting Cloudflare over v6.
4. Wait ~2 min, then redeploy / restart the proxy. The cert is issued and the
   site loads.

To keep Cloudflare proxying afterwards: issue the cert first with grey cloud,
then re-enable the orange cloud with Cloudflare **SSL/TLS = Full (strict)**.

### Proxy labels still point to the wrong port (3000)

On Coolify, labels are generated when the resource is created (default port
3000). If you change **Ports Exposes** to `5001` afterwards, the read-only
labels do **not** auto-update. Click **Save**, then **Reset Labels to
Defaults**, confirm `loadbalancer.server.port=5001`, and redeploy.

### No external database needed

MiroFish uses **embedded KuzuDB** (files under `backend/data/graphdb`) — there
is no Postgres/MySQL service to deploy. (The sibling project *BettaFish* is the
one that needs PostgreSQL; MiroFish does not.)
