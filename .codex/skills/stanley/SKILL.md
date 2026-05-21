---
name: stanley
description: Configure Domain and Nginx Reverse Proxy for VPS. Use the skill only when user asks to configure NGNIX on VPS.
---

# Configure Domain + Nginx Reverse Proxy for VPS

## 1. Prerequisites: Point Domain to VPS
*This step requires a human intervention. Prompt the user if they have already completed this step.*

In your domain registrar (Cloudflare, Namecheap, GoDaddy, etc.), create an **A Record**:

| Type | Name | Value |
|------|------|------|
| A | @ | `<VPS_IP>` |

Optional (for subdomain):

| Type | Name | Value |
|------|------|------|
| A | `www` | `<VPS_IP>` |

Wait for DNS propagation if needed.

## SSH into VPS

```bash
ssh <username>@<VPS_IP>
```

---

# 2. Create Nginx Reverse Proxy Config

Create a new Nginx site config:

```bash
sudo nano /etc/nginx/sites-available/<sitename>.conf

Paste the following configuration:

# ======================================
# HTTP → HTTPS Redirect
# ======================================
server {
    listen 80;
    server_name <domain-name>;

    return 301 https://$host$request_uri;
}

# ======================================
# HTTPS Reverse Proxy
# ======================================
server {
    listen 443 ssl http2;
    server_name <domain-name>;

    # ----------------------------------
    # SSL Certificates
    # ----------------------------------
    ssl_certificate     /etc/nginx/ssl/cloudflare-origin.pem;
    ssl_certificate_key /etc/nginx/ssl/cloudflare-origin.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    # ----------------------------------
    # Security Headers
    # ----------------------------------
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # ----------------------------------
    # Timeout Hardening
    # ----------------------------------
    client_body_timeout 10s;
    client_header_timeout 10s;
    keepalive_timeout 30s;
    send_timeout 10s;

    # ----------------------------------
    # Block Access to Sensitive Files
    # ----------------------------------
    location ~* /(\.env|\.git|\.svn|\.hg|composer|package|yarn|node_modules) {
        deny all;
        return 404;
    }

    # ----------------------------------
    # Block Potentially Executable Files
    # ----------------------------------
    location ~* \.(sh|exe|bin|dll|so)$ {
        deny all;
        return 404;
    }

    # ----------------------------------
    # Reverse Proxy to App
    # ----------------------------------
    location / {
        proxy_pass http://localhost:<APP_PORT>;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

# 3. Enable the Nginx Site:
```bash
sudo ln -s /etc/nginx/sites-available/<sitename>.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```