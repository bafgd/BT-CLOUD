# Deploying btcloud.work

Architecture: one nginx instance on your Linux box handles all three hostnames.
The landing page is served directly by nginx as static files. Talk and Drive
are reverse-proxied to Stoat and OpenCloud, which can run on the same box
(as Docker containers) or elsewhere on your LAN.

```
                     ┌───────────────────────────┐
 Internet ───────────▶  nginx  (ports 80 / 443)  │
                     │                           │
                     │  btcloud.work ──▶ static files (this landing page)
                     │  talk.btcloud.work ──▶ 127.0.0.1:PORT (Stoat/Caddy)
                     │  drive.btcloud.work ──▶ 127.0.0.1:9200 (OpenCloud)
                     └───────────────────────────┘
```

Assumes Ubuntu/Debian (`sites-available` / `sites-enabled` layout). If you're
on a distro without that layout, drop the `.conf` files straight into
`/etc/nginx/conf.d/` instead and skip the symlink steps.

---

## 0. Before you start

- Root or sudo access on the Linux box that will run nginx
- Access to your domain's DNS settings (wherever `btcloud.work` is registered)
- Access to your home router's port forwarding settings
- Docker + Docker Compose v2 installed, if Stoat/OpenCloud aren't running yet:
  ```bash
  sudo apt-get update
  sudo apt-get install -y ca-certificates curl git
  curl -fsSL https://get.docker.com | sudo sh
  ```

**A note on your home IP:** opening 80/443 on your router publishes your
home's public IP address to the world. That's inherent to self-hosting from
home — a VPN, or fronting the HTTP sites with a CDN/proxy like Cloudflare,
can hide it for the web traffic, but Stoat's voice/video calling (below)
needs direct UDP ports that can't be hidden that way. If that's a
dealbreaker, you can leave voice calling off.

**If your ISP gives you a dynamic IP** (most home connections do), your A
records will go stale whenever it changes. Set up dynamic DNS — either your
router's built-in DDNS client (if your DNS provider supports it) or a small
client like `ddclient`, running on the nginx box, that updates the record
automatically.

---

## 1. DNS

At your DNS provider, create A records (and AAAA if you have IPv6) pointing
at your home's public IP:

| Host | Type | Value |
|---|---|---|
| `btcloud.work` | A | your public IP |
| `www.btcloud.work` | A | your public IP |
| `talk.btcloud.work` | A | your public IP |
| `drive.btcloud.work` | A | your public IP |

Give it a few minutes to propagate before moving on. Check with:

```bash
dig +short talk.btcloud.work
```

## 2. Router port forwarding

Forward these to the internal LAN IP of your nginx box:

| Port | Protocol | Purpose |
|---|---|---|
| 80 | TCP | HTTP + Let's Encrypt validation |
| 443 | TCP | HTTPS |

Stoat's voice/video calling additionally needs these forwarded straight to
whichever machine actually runs the Stoat containers (nginx can't proxy
this — it's raw WebRTC media, not HTTP):

| Port | Protocol | Purpose |
|---|---|---|
| 7881 | TCP | LiveKit signaling |
| 50000–50100 | UDP | LiveKit call media |

Skip these two rows if you don't care about voice/video in Talk.

## 3. Install nginx and certbot

```bash
sudo apt-get update
sudo apt-get install -y nginx certbot python3-certbot-nginx
sudo ufw allow 'Nginx Full'   # opens 80 + 443 if you use ufw
```

---

## 4. The landing page

```bash
sudo mkdir -p /var/www/btcloud.work
sudo cp index.html /var/www/btcloud.work/
sudo chown -R www-data:www-data /var/www/btcloud.work
```

Install the site config:

```bash
sudo cp btcloud.work.conf /etc/nginx/sites-available/btcloud.work
sudo ln -s /etc/nginx/sites-available/btcloud.work /etc/nginx/sites-enabled/
```

---

## 5. Talk (Stoat)

If you haven't deployed Stoat yet:

```bash
git clone https://github.com/stoatchat/self-hosted stoat
cd stoat
chmod +x ./generate_config.sh
./generate_config.sh talk.btcloud.work
```

When the script asks whether to place Stoat behind another reverse proxy,
**say yes**. It'll bind its internal Caddy to a local port and tell you what
that port is (their docs currently reference `8880`, but use whatever the
script actually prints for you). Then:

```bash
docker compose up -d
```

Back on the nginx box, edit `talk.btcloud.work.conf` and swap `1234` for the
real port from the step above (skip this if Stoat runs on this same nginx
box and the port matches). Then install it:

```bash
sudo cp talk.btcloud.work.conf /etc/nginx/sites-available/talk.btcloud.work
sudo ln -s /etc/nginx/sites-available/talk.btcloud.work /etc/nginx/sites-enabled/
```

## 6. Drive (OpenCloud)

If you haven't deployed OpenCloud yet:

```bash
git clone https://github.com/opencloud-eu/opencloud-compose.git
cd opencloud-compose
cp .env.example .env
nano .env
```

Set at least:

```env
OC_DOMAIN=drive.btcloud.work
INITIAL_ADMIN_PASSWORD=some-strong-password-here
COMPOSE_FILE=docker-compose.yml:external-proxy/opencloud.yml
```

(Use `opencloud-exposed.yml` instead of `opencloud.yml` if OpenCloud runs on
a *different* machine than nginx — that variant publishes port 9200 on the
LAN instead of just localhost. This skips Collabora/in-browser office
editing to keep things simple; add it later from the [OpenCloud docs](https://docs.opencloud.eu/docs/admin/getting-started/container/docker-compose/external-proxy/) if you want it.)

```bash
docker compose up -d
```

Install the nginx config:

```bash
sudo cp drive.btcloud.work.conf /etc/nginx/sites-available/drive.btcloud.work
sudo ln -s /etc/nginx/sites-available/drive.btcloud.work /etc/nginx/sites-enabled/
```

---

## 7. Check nginx, then get certificates

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Visit `http://btcloud.work` etc. in a browser to confirm the plain-HTTP
versions work before adding TLS. Then get one certificate covering all three:

```bash
sudo certbot --nginx -d btcloud.work -d www.btcloud.work -d talk.btcloud.work -d drive.btcloud.work
```

Certbot edits all three site files in place: adds the certificate lines,
sets up the HTTP→HTTPS redirect, and reloads nginx for you. Test the
auto-renewal:

```bash
sudo certbot renew --dry-run
```

## 8. A few extra lines worth adding after certbot

Certbot only adds the certificate + redirect — it won't know about
service-specific tuning. Open `/etc/nginx/sites-available/drive.btcloud.work`
and add these inside the new `listen 443 ssl;` server block (anywhere above
the closing `}`):

```nginx
client_max_body_size 10M;      # raise this if you'll upload bigger files
proxy_buffering off;
proxy_request_buffering off;
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
keepalive_requests 100000;
keepalive_timeout 5m;
```

Without these, large uploads or long-lived connections in OpenCloud can time
out or get cut off mid-sync. Reload once more:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

The `talk.btcloud.work` config doesn't need anything extra — the `/ws` and
`/livekit` blocks you installed in step 5 survive certbot's edit untouched.

---

## 9. Test it

- `https://btcloud.work` — landing page, both cards should flip to "online"
  within a few seconds
- `https://talk.btcloud.work` — Stoat sign-up/login screen
- `https://drive.btcloud.work` — OpenCloud login screen

If a card sticks on "can't reach," that service isn't answering on 443 yet —
double check its container is running and the corresponding nginx site is
enabled and reloaded.

## 10. Hardening (worth doing before you forget)

```bash
sudo apt-get install -y fail2ban
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

- Use SSH keys and disable password auth (`PasswordAuthentication no` in
  `/etc/ssh/sshd_config`), especially once port 80/443 are open to the world
- Keep Docker images and the host OS patched — `docker compose pull` and
  `apt-get upgrade` periodically
- Back up `secrets.env`/`Revolt.toml` (Stoat) and your OpenCloud data volumes
  — losing those means losing access to everything stored in each service

---

### Sources for the app-specific steps

- Stoat reverse proxy setup: [github.com/stoatchat/self-hosted](https://github.com/stoatchat/self-hosted#placing-behind-another-reverse-proxy-or-another-port)
- OpenCloud reverse proxy setup: [docs.opencloud.eu](https://docs.opencloud.eu/docs/admin/getting-started/container/docker-compose/external-proxy/)
