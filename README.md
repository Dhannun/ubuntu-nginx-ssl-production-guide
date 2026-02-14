# Production-Grade Ubuntu Server Setup Guide

**Complete guide for setting up Nginx, Let's Encrypt SSL, UFW Firewall, and Docker Compose integration**

---

## Table of Contents

1. [Overview & Architecture](#overview--architecture)
2. [Why This Approach?](#why-this-approach)
3. [Prerequisites](#prerequisites)
4. [Step 1: Initial System Setup](#step-1-initial-system-setup)
5. [Step 2: UFW Firewall Configuration](#step-2-ufw-firewall-configuration)
6. [Step 3: Install Certbot](#step-3-install-certbot)
7. [Step 4: Certificate Methods Comparison](#step-4-certificate-methods-comparison)
8. [Step 5A: HTTP-01 Challenge (Standard Method)](#step-5a-http-01-challenge-standard-method)
9. [Step 5B: DNS-01 Challenge (Cloudflare Method)](#step-5b-dns-01-challenge-cloudflare-method)
10. [Step 6: Nginx Reverse Proxy Configuration](#step-6-nginx-reverse-proxy-configuration)
11. [Step 7: Docker Compose Integration](#step-7-docker-compose-integration)
12. [Step 8: Cloudflare Configuration](#step-8-cloudflare-configuration)
13. [Step 9: Auto-Renewal Setup](#step-9-auto-renewal-setup)
14. [Step 10: Security Hardening](#step-10-security-hardening)
15. [Monitoring & Maintenance](#monitoring--maintenance)
16. [Troubleshooting](#troubleshooting)

---

## Overview & Architecture

### Architecture Diagram

```
Internet
    ↓
Cloudflare (Optional Proxy & DDoS Protection)
    ↓
Your Server IP
    ↓
UFW Firewall (Ports: 22, 80, 443)
    ↓
Nginx (Reverse Proxy - NOT in Docker)
    ↓
Docker Containers (Apps on localhost ports)
```

### Key Components

- **Ubuntu Server** - Base operating system
- **UFW** - Uncomplicated Firewall for network security
- **Nginx** - Reverse proxy and web server (host-level, not containerized)
- **Certbot** - Automated SSL certificate management
- **Let's Encrypt** - Free SSL/TLS certificates
- **Docker Compose** - Container orchestration for applications
- **Cloudflare** - DNS and optional CDN/DDoS protection

---

## Why This Approach?

### 1. Nginx Outside Docker - Strategic Decision

**Advantages:**

✅ **Single Point of SSL Management**
- All domains share one Nginx instance
- SSL certificates managed in one location
- Easier to maintain and renew certificates
- No need to rebuild containers for SSL changes

✅ **Better Performance**
- Direct host network access (no Docker network overhead)
- No container restart needed for adding new domains
- Lower memory footprint

✅ **Simplified Certificate Renewal**
- Certbot renewal hooks work directly with host Nginx
- No volume mounting complexities
- Automatic Nginx reload after certificate renewal

✅ **Flexibility for Multiple Domains**
- Easy to add/remove domains without affecting containers
- Can proxy to different Docker containers or external services
- Centralized logging and monitoring

✅ **Security Benefits**
- Docker containers bound to localhost only
- Only Nginx exposed to internet
- Easier to implement WAF rules and rate limiting
- Single point for security headers

**Why NOT put Nginx in Docker for this use case:**
- Managing SSL certificates across multiple containers is complex
- Each container would need separate certificate volumes
- Harder to implement shared configurations
- More resource intensive with multiple Nginx instances
- Certificate renewal becomes complicated with container restarts

### 2. UFW Firewall - Defense in Depth

**Advantages:**

✅ **Simple Yet Effective**
- Easy to configure and understand
- Great for Ubuntu/Debian systems
- Integrates well with iptables underneath

✅ **Production Security**
- Default deny incoming traffic
- Whitelist only necessary ports
- Rate limiting for SSH (brute force protection)
- IPv6 support built-in

✅ **Docker Compatibility**
- Works alongside Docker's iptables rules
- Protects host-level services
- Prevents direct access to Docker containers

### 3. Let's Encrypt with Certbot - Free & Automated SSL

**Advantages:**

✅ **Zero Cost**
- Free SSL certificates
- Industry-standard encryption
- Trusted by all major browsers

✅ **Automated Renewal**
- Auto-renewal via systemd timer
- Hooks for post-renewal actions
- Email notifications before expiry

✅ **Flexible Challenge Methods**
- HTTP-01 for standard domains
- DNS-01 for wildcards and restricted environments
- TLS-ALPN-01 for port 80 blocked scenarios

### 4. Cloudflare Integration - Enhanced Protection

**Advantages:**

✅ **DDoS Protection**
- Cloudflare's global network absorbs attacks
- Rate limiting at edge
- Bot protection

✅ **CDN Benefits**
- Static content caching
- Reduced bandwidth costs
- Faster global delivery

✅ **DNS Management**
- Fast DNS resolution
- API for automation
- DNS-based certificate validation

✅ **Flexible SSL Options**
- Can proxy traffic (orange cloud) for protection
- Can direct connect (gray cloud) for speed
- Full (strict) mode ensures end-to-end encryption

---

## Prerequisites

- Ubuntu Server (20.04, 22.04, or 24.04 LTS)
- Root or sudo access
- Domain(s) pointed to your server IP
- Cloudflare account (if using DNS-01 or proxy features)
- Docker and Docker Compose installed

---

## Step 1: Initial System Setup

### Update System

```bash
# Update package lists and upgrade existing packages
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y nginx ufw curl wget git
```

**Why these packages?**
- `nginx` - Web server and reverse proxy
- `ufw` - Firewall management
- `curl/wget` - Download utilities
- `git` - Version control (useful for deployment)

### Verify Nginx Installation

```bash
# Check Nginx version
nginx -v

# Check Nginx status
sudo systemctl status nginx

# Enable Nginx to start on boot
sudo systemctl enable nginx
```

---

## Step 2: UFW Firewall Configuration

### Critical First Steps (Prevent Lockout!)

```bash
# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# IMPORTANT: Allow SSH FIRST to prevent lockout
sudo ufw allow OpenSSH
# Or if using custom SSH port: sudo ufw allow 2222/tcp
```

**🚨 WARNING:** Always allow SSH before enabling UFW, or you'll lock yourself out!

### Allow Web Traffic

```bash
# Allow HTTP (needed for Let's Encrypt HTTP-01 challenge)
sudo ufw allow 80/tcp

# Allow HTTPS
sudo ufw allow 443/tcp
```

### Enable IPv6 Support

```bash
# Edit UFW defaults
sudo nano /etc/default/ufw

# Ensure this line is set to:
IPV6=yes
```

**Why IPv6?**
- Future-proofing your server
- Some visitors may only have IPv6
- Let's Encrypt can validate via IPv6

### Enable Firewall

```bash
# Enable UFW
sudo ufw enable

# Check status
sudo ufw status verbose

# Check numbered rules (easier to delete specific rules)
sudo ufw status numbered
```

### Production Security Enhancements

```bash
# Enable logging for security monitoring
sudo ufw logging on

# Rate limit SSH to prevent brute force attacks
sudo ufw limit OpenSSH

# Optional: Restrict SSH to specific IP range
# sudo ufw allow from 192.168.1.0/24 to any port 22

# Optional: Allow ping (ICMP)
sudo ufw allow proto icmp
```

**Expected Output:**
```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     LIMIT       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
```

---

## Step 3: Install Certbot

### Method A: Snap Installation (Recommended)

```bash
# Install Certbot via snap (always latest version)
sudo snap install --classic certbot

# Create symlink for easy access
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

**Why Snap?**
- ✅ Always gets the latest Certbot version
- ✅ Automatic security updates
- ✅ Isolated from system packages
- ✅ Recommended by Let's Encrypt

### Method B: APT Installation (Alternative)

```bash
# Install Certbot via apt
sudo apt install certbot python3-certbot-nginx -y
```

**When to use APT?**
- Snap not available on your system
- Prefer system package manager
- Need specific older version

### Verify Installation

```bash
# Check Certbot version
certbot --version

# Should show: certbot 2.x.x or newer
```

---

## Step 4: Certificate Methods Comparison

### HTTP-01 Challenge

**How it works:**
1. You request a certificate for `example.com`
2. Let's Encrypt gives you a unique token
3. You place it at `http://example.com/.well-known/acme-challenge/TOKEN`
4. Let's Encrypt fetches it via HTTP to verify domain ownership
5. Certificate issued

**Advantages:**
- ✅ Simple setup
- ✅ No API tokens required
- ✅ Works with Cloudflare proxy enabled
- ✅ Automatic with Certbot Nginx plugin

**Disadvantages:**
- ❌ Cannot issue wildcard certificates
- ❌ Requires port 80 open
- ❌ Separate cert needed for each subdomain
- ❌ Domain must be publicly accessible

**Best for:**
- Small number of specific domains
- Standard website hosting
- When you don't need wildcards

### DNS-01 Challenge (Cloudflare)

**How it works:**
1. You request a certificate for `*.example.com`
2. Let's Encrypt gives you a unique token
3. Certbot automatically creates a TXT record in Cloudflare DNS
4. Let's Encrypt queries DNS to verify ownership
5. Certificate issued

**Advantages:**
- ✅ Supports wildcard certificates (`*.example.com`)
- ✅ Works without opening port 80
- ✅ Works behind firewalls/NAT
- ✅ One cert covers all subdomains
- ✅ Can validate private/internal domains

**Disadvantages:**
- ❌ Requires Cloudflare API token
- ❌ Slightly more complex setup
- ❌ DNS propagation delay (60+ seconds)
- ❌ Requires additional Certbot plugin

**Best for:**
- Multiple subdomains under same domain
- Internal/private servers
- When port 80 cannot be exposed
- Large-scale deployments

### Decision Matrix

| Your Situation | Recommended Method | Reason |
|----------------|-------------------|--------|
| 1-5 domains, all different | HTTP-01 | Simple, no API needed |
| Multiple subdomains (app1, app2, app3.example.com) | DNS-01 wildcard | One cert covers all |
| Mixed domains (example1.com, example2.com, app.example1.com) | Hybrid (both) | Wildcard for subdomains, HTTP for others |
| Server behind firewall | DNS-01 | No port 80 needed |
| Maximum simplicity | HTTP-01 | Easiest setup |

---

## Step 5A: HTTP-01 Challenge (Standard Method)

### Configure Nginx for ACME Challenge

```bash
# Create or edit default Nginx config
sudo nano /etc/nginx/sites-available/default
```

**Add this configuration:**

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    # ACME challenge location for all domains
    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        root /var/www/html;
        allow all;
    }

    # Redirect everything else to HTTPS (after certificates obtained)
    location / {
        return 301 https://$host$request_uri;
    }
}
```

**Why this configuration?**
- `^~` - Prefix match that stops further regex matching (priority)
- `root /var/www/html` - Where Certbot places challenge files
- `allow all` - Ensures no access restrictions block Let's Encrypt
- Redirect to HTTPS after SSL setup

```bash
# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### Option 1: Automated with Nginx Plugin (Easiest)

```bash
# For a single domain with www
sudo certbot --nginx -d example.com -d www.example.com

# For multiple separate domains
sudo certbot --nginx -d app1.example.com
sudo certbot --nginx -d app2.example.com
sudo certbot --nginx -d blog.example.com
```

**What this does:**
1. Obtains SSL certificate
2. Automatically modifies Nginx config
3. Sets up HTTPS redirect
4. Configures SSL parameters
5. Reloads Nginx

**Interactive prompts:**
- Email address (for renewal notifications)
- Agree to Terms of Service
- Redirect HTTP to HTTPS? (Choose "2" for yes)

**Advantages:**
- ✅ Fully automated
- ✅ Handles Nginx configuration
- ✅ Best for beginners
- ✅ Minimal manual work

**Disadvantages:**
- ❌ Less control over Nginx config
- ❌ May override custom settings
- ❌ Generic SSL configuration

### Option 2: Manual with Certonly (More Control)

```bash
# Obtain certificate without modifying Nginx
sudo certbot certonly --webroot \
  -w /var/www/html \
  -d example.com \
  -d www.example.com \
  --email your@email.com \
  --agree-tos \
  --no-eff-email

# For multiple domains (creates separate certificates)
sudo certbot certonly --webroot -w /var/www/html -d app1.example.com
sudo certbot certonly --webroot -w /var/www/html -d app2.example.com
```

**What this does:**
1. Obtains certificate only
2. Saves to `/etc/letsencrypt/live/DOMAIN/`
3. Does NOT modify Nginx config
4. You manually configure Nginx (see Step 6)

**Advantages:**
- ✅ Full control over Nginx configuration
- ✅ Can review before applying
- ✅ Easier to troubleshoot
- ✅ Production-grade custom configs

**Certificate locations:**
```
/etc/letsencrypt/live/example.com/
├── fullchain.pem    # Certificate + intermediate (use this in Nginx)
├── privkey.pem      # Private key (keep secure!)
├── cert.pem         # Certificate only
└── chain.pem        # Intermediate certificates
```

---

## Step 5B: DNS-01 Challenge (Cloudflare Method)

### Install Cloudflare Plugin

```bash
# Set trust for plugin
sudo snap set certbot trust-plugin-with-root=ok

# Install Cloudflare DNS plugin
sudo snap install certbot-dns-cloudflare
```

### Create Cloudflare API Token

**Step-by-step in Cloudflare Dashboard:**

1. Log in to Cloudflare
2. Go to **My Profile** → **API Tokens**
3. Click **Create Token**
4. Use **Edit zone DNS** template
5. **Permissions:**
   - Zone - DNS - Edit
6. **Zone Resources:**
   - Include - Specific zone - `example.com`
7. Click **Continue to summary**
8. Click **Create Token**
9. **Copy the token** (shown only once!)

**Security best practice:**
- Use API tokens (NOT Global API Key)
- Limit to specific zones only
- Store securely

### Configure Cloudflare Credentials

```bash
# Create secure directory
sudo mkdir -p /etc/letsencrypt/cloudflare
sudo chmod 0700 /etc/letsencrypt/cloudflare

# Create credentials file
sudo nano /etc/letsencrypt/cloudflare/credentials.ini
```

**Add your token:**

```ini
# Cloudflare API token (recommended - more secure)
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN_HERE

# Legacy method (NOT recommended - less secure)
# dns_cloudflare_email = your@email.com
# dns_cloudflare_api_key = YOUR_GLOBAL_API_KEY
```

**Secure the file:**

```bash
# Restrict permissions (only root can read)
sudo chmod 0400 /etc/letsencrypt/cloudflare/credentials.ini
```

**Why these permissions?**
- `0700` for directory - Only root can access
- `0400` for file - Only root can read, nobody can write
- Prevents unauthorized access to your API token

### Obtain Wildcard Certificate

```bash
# Single wildcard certificate for all subdomains
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare/credentials.ini \
  --dns-cloudflare-propagation-seconds 60 \
  -d example.com \
  -d '*.example.com' \
  --email your@email.com \
  --agree-tos \
  --no-eff-email
```

**Parameters explained:**
- `certonly` - Get certificate only, don't modify configs
- `--dns-cloudflare` - Use Cloudflare DNS challenge
- `--dns-cloudflare-credentials` - Path to API token file
- `--dns-cloudflare-propagation-seconds 60` - Wait time for DNS propagation
- `-d '*.example.com'` - Wildcard for all subdomains (note the quotes!)
- `--no-eff-email` - Don't share email with EFF

**What happens:**
1. Certbot requests certificate from Let's Encrypt
2. Receives DNS challenge token
3. Uses Cloudflare API to create TXT record
4. Waits 60 seconds for DNS propagation
5. Let's Encrypt validates via DNS query
6. Certificate issued and saved

**One wildcard covers:**
- `app1.example.com` ✅
- `app2.example.com` ✅
- `blog.example.com` ✅
- `api.example.com` ✅
- `example.com` ✅ (if included with `-d example.com`)

### Obtain Multiple Domain Certificates

```bash
# For completely different domains
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare/credentials.ini \
  -d example1.com -d '*.example1.com' \
  --email your@email.com

sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare/credentials.ini \
  -d example2.com -d '*.example2.com' \
  --email your@email.com
```

---

## Step 6: Nginx Reverse Proxy Configuration

### Understanding the Setup

**Traffic flow:**
```
User → Cloudflare (optional) → Your Server:443 → Nginx → localhost:3000 → Docker Container
```

### Create Nginx Configuration for Each Domain

```bash
# Create configuration file
sudo nano /etc/nginx/sites-available/app1.example.com
```

### Production-Grade Nginx Configuration

```nginx
# ============================================
# HTTP Server - Redirect to HTTPS
# ============================================
server {
    listen 80;
    listen [::]:80;
    server_name app1.example.com;

    # Allow ACME challenge (for certificate renewal)
    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        root /var/www/html;
    }

    # Redirect all other HTTP traffic to HTTPS
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# ============================================
# HTTPS Server - Main Configuration
# ============================================
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name app1.example.com;

    # ====================================
    # SSL Certificate Configuration
    # ====================================
    
    # For individual domain certificate (HTTP-01)
    ssl_certificate /etc/letsencrypt/live/app1.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app1.example.com/privkey.pem;
    
    # For wildcard certificate (DNS-01) - use this instead
    # ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # ====================================
    # SSL Protocol Configuration
    # ====================================
    
    # Only use TLS 1.2 and 1.3 (disable older insecure versions)
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # Let client choose cipher (better for modern clients)
    ssl_prefer_server_ciphers off;
    
    # Strong cipher suites (AEAD ciphers preferred)
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    # ====================================
    # OCSP Stapling (improves SSL performance)
    # ====================================
    
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/app1.example.com/chain.pem;
    
    # Use Cloudflare DNS for OCSP verification
    resolver 1.1.1.1 1.0.0.1 valid=300s;
    resolver_timeout 5s;

    # ====================================
    # SSL Session Configuration
    # ====================================
    
    # Shared session cache across workers (10MB = ~40,000 sessions)
    ssl_session_cache shared:SSL:10m;
    
    # Session timeout (1 day)
    ssl_session_timeout 1d;
    
    # Disable session tickets (better privacy)
    ssl_session_tickets off;

    # ====================================
    # Security Headers
    # ====================================
    
    # Force HTTPS for 1 year, include subdomains, allow preload list
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    
    # Prevent clickjacking - allow same origin framing only
    add_header X-Frame-Options "SAMEORIGIN" always;
    
    # Prevent MIME type sniffing
    add_header X-Content-Type-Options "nosniff" always;
    
    # Enable XSS filter in browsers
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Control referrer information
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    
    # Content Security Policy (customize based on your needs)
    # add_header Content-Security-Policy "default-src 'self' https:; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';" always;

    # ====================================
    # Logging
    # ====================================
    
    access_log /var/log/nginx/app1.example.com-access.log;
    error_log /var/log/nginx/app1.example.com-error.log;

    # ====================================
    # Reverse Proxy Configuration
    # ====================================
    
    location / {
        # Proxy to Docker container on localhost
        proxy_pass http://127.0.0.1:3000;
        
        # Use HTTP/1.1 for proxy connection
        proxy_http_version 1.1;
        
        # ================================
        # WebSocket Support
        # ================================
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        
        # ================================
        # Standard Proxy Headers
        # ================================
        
        # Pass original host header
        proxy_set_header Host $host;
        
        # Real client IP (important for rate limiting, logging)
        proxy_set_header X-Real-IP $remote_addr;
        
        # IP chain (includes proxies)
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Original protocol (http/https)
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Original port
        proxy_set_header X-Forwarded-Port $server_port;
        
        # Original host
        proxy_set_header X-Forwarded-Host $host;
        
        # ================================
        # Proxy Behavior
        # ================================
        
        # Bypass cache for WebSocket connections
        proxy_cache_bypass $http_upgrade;
        
        # Don't redirect proxy_pass automatically
        proxy_redirect off;
        
        # ================================
        # Timeout Settings
        # ================================
        
        # Time to establish connection to backend
        proxy_connect_timeout 60s;
        
        # Time to transmit request to backend
        proxy_send_timeout 60s;
        
        # Time to receive response from backend
        proxy_read_timeout 60s;
        
        # ================================
        # Buffer Settings
        # ================================
        
        # Enable buffering (better performance for most cases)
        proxy_buffering on;
        
        # Buffer size for reading response header
        proxy_buffer_size 4k;
        
        # Number and size of buffers for response body
        proxy_buffers 8 4k;
        
        # Max size of busy buffers
        proxy_busy_buffers_size 8k;
    }

    # ====================================
    # Optional: Static File Caching
    # ====================================
    
    # Cache static assets
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        proxy_pass http://127.0.0.1:3000;
        proxy_cache_valid 200 1d;
        add_header Cache-Control "public, max-age=86400";
    }

    # ====================================
    # File Upload Size Limit
    # ====================================
    
    # Allow larger file uploads (adjust as needed)
    client_max_body_size 50M;
    
    # Buffer size for request body
    client_body_buffer_size 128k;
}
```

### Enable the Site

```bash
# Create symbolic link to enable site
sudo ln -s /etc/nginx/sites-available/app1.example.com /etc/nginx/sites-enabled/

# Test Nginx configuration for syntax errors
sudo nginx -t

# If test passes, reload Nginx
sudo systemctl reload nginx
```

### Verify Configuration

```bash
# Check if site is enabled
ls -la /etc/nginx/sites-enabled/

# Test HTTPS connection
curl -I https://app1.example.com

# Check SSL certificate
echo | openssl s_client -servername app1.example.com -connect app1.example.com:443 2>/dev/null | openssl x509 -noout -dates -subject
```

### Configuration Explanation

**Why these settings?**

| Setting | Purpose | Benefit |
|---------|---------|---------|
| `http2` | Enable HTTP/2 protocol | Faster page loads, multiplexing |
| `ssl_protocols TLSv1.2 TLSv1.3` | Disable old protocols | Security (prevent downgrade attacks) |
| `ssl_prefer_server_ciphers off` | Client chooses cipher | Better performance on modern devices |
| `OCSP Stapling` | Cache certificate status | Faster SSL handshake, privacy |
| `ssl_session_cache` | Reuse SSL sessions | Faster reconnections |
| `HSTS header` | Force HTTPS | Prevent SSL stripping attacks |
| `X-Frame-Options` | Prevent embedding | Clickjacking protection |
| `proxy_set_header X-Real-IP` | Pass real client IP | Accurate logging in app |
| `proxy_buffering on` | Buffer responses | Better performance |
| `client_max_body_size` | Upload limit | Allow file uploads |

---

## Step 7: Docker Compose Integration

### Security-First Docker Configuration

**Key principle:** Bind Docker containers to `127.0.0.1` (localhost) only, making them accessible ONLY via Nginx reverse proxy.

### Example Docker Compose Configuration

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ==============================
  # Application 1
  # ==============================
  app1:
    image: your-app1:latest
    container_name: app1
    restart: unless-stopped
    
    # SECURITY: Bind to localhost only!
    ports:
      - "127.0.0.1:3000:3000"
      # NOT: "3000:3000" (would expose to 0.0.0.0 - public!)
    
    environment:
      - NODE_ENV=production
      - PORT=3000
    
    volumes:
      - ./app1-data:/app/data
    
    networks:
      - app1-network
    
    # Health check
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # ==============================
  # Application 2 (Different Port)
  # ==============================
  app2:
    image: your-app2:latest
    container_name: app2
    restart: unless-stopped
    
    ports:
      - "127.0.0.1:3001:3000"  # External 3001 -> Internal 3000
    
    environment:
      - NODE_ENV=production
    
    networks:
      - app2-network

  # ==============================
  # Database (PostgreSQL Example)
  # ==============================
  postgres:
    image: postgres:15-alpine
    container_name: postgres-db
    restart: unless-stopped
    
    # NO EXTERNAL PORT - only accessible by other containers
    # ports:
    #   - "127.0.0.1:5432:5432"  # Only uncomment if needed for local development
    
    environment:
      - POSTGRES_USER=dbuser
      - POSTGRES_PASSWORD=strongpassword
      - POSTGRES_DB=production_db
    
    volumes:
      - postgres-data:/var/lib/postgresql/data
    
    networks:
      - app1-network  # Only app1 can access this DB

# ==============================
# Networks
# ==============================
networks:
  app1-network:
    driver: bridge
  app2-network:
    driver: bridge

# ==============================
# Volumes
# ==============================
volumes:
  postgres-data:
    driver: local
```

### Port Binding Explanation

**Insecure (DON'T DO THIS):**
```yaml
ports:
  - "3000:3000"  # Binds to 0.0.0.0:3000 - PUBLICLY ACCESSIBLE!
```

**Secure (CORRECT):**
```yaml
ports:
  - "127.0.0.1:3000:3000"  # Binds to localhost only - ONLY Nginx can access
```

**Why this matters:**
- Docker by default binds to `0.0.0.0` (all interfaces)
- Anyone can access `http://your-server-ip:3000` directly
- Bypasses Nginx security, SSL, rate limiting
- Binding to `127.0.0.1` restricts access to localhost only
- Only Nginx can proxy to the container

### Nginx Configuration for Multiple Apps

```nginx
# app2.example.com configuration
server {
    listen 443 ssl http2;
    server_name app2.example.com;
    
    # SSL certificate (can use same wildcard cert)
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    # ... (SSL settings same as above) ...
    
    # Proxy to app2 on port 3001
    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Managing Docker Containers

```bash
# Start all containers
docker-compose up -d

# View running containers
docker-compose ps

# View logs
docker-compose logs -f app1

# Restart specific service
docker-compose restart app1

# Stop all containers
docker-compose down

# Stop and remove volumes (CAUTION: deletes data!)
# docker-compose down -v
```

### Verify Localhost Binding

```bash
# Check which ports are listening
sudo ss -tulpn | grep docker

# Should show 127.0.0.1:3000, NOT 0.0.0.0:3000

# Test that port is NOT publicly accessible
curl http://YOUR_SERVER_IP:3000
# Should fail: Connection refused

# Test that it IS accessible via Nginx
curl https://app1.example.com
# Should work!
```

---

## Step 8: Cloudflare Configuration

### DNS Settings

**For each domain/subdomain:**

1. Go to Cloudflare Dashboard → DNS
2. Add A record:
   - **Type:** A
   - **Name:** `app1` (or `@` for root domain)
   - **IPv4 address:** Your server IP
   - **Proxy status:** Choose based on needs (see below)
   - **TTL:** Auto

### Proxy Status Decision

**🟠 Proxied (Orange Cloud) - Recommended**

```
User → Cloudflare CDN → Your Server
```

**Advantages:**
- ✅ DDoS protection
- ✅ Hide your server IP
- ✅ Free SSL at Cloudflare edge
- ✅ WAF (Web Application Firewall)
- ✅ CDN caching for static assets
- ✅ Bot protection
- ✅ Analytics

**Disadvantages:**
- ❌ Adds small latency (usually <50ms)
- ❌ Cloudflare sees all traffic
- ❌ Extra hop in connection

**⚪ DNS Only (Gray Cloud)**

```
User → Directly to Your Server
```

**Advantages:**
- ✅ Lower latency (direct connection)
- ✅ No traffic inspection by Cloudflare
- ✅ Simpler troubleshooting

**Disadvantages:**
- ❌ Server IP exposed
- ❌ No DDoS protection
- ❌ No CDN caching
- ❌ No WAF

**Recommendation:** Use **Orange Cloud (Proxied)** for production sites unless you have specific reasons not to.

### SSL/TLS Settings

**Go to:** Cloudflare Dashboard → SSL/TLS

**SSL/TLS encryption mode:**

| Mode | How it works | When to use |
|------|-------------|-------------|
| **Off** | No encryption | ❌ Never (insecure!) |
| **Flexible** | Cloudflare ↔ User: HTTPS<br>Cloudflare ↔ Server: HTTP | ❌ Not recommended (insecure backend) |
| **Full** | Both encrypted, but server cert can be self-signed | ⚠️ Only if no Let's Encrypt |
| **Full (strict)** | Both encrypted, server cert must be valid | ✅ **RECOMMENDED** |

**Set to:** **Full (strict)**

**Why?**
- End-to-end encryption
- Validates your Let's Encrypt certificate
- Prevents man-in-the-middle attacks
- Production best practice

### Additional SSL/TLS Settings

**Edge Certificates:**
- ✅ **Always Use HTTPS** - ON
- ✅ **Automatic HTTPS Rewrites** - ON
- ✅ **Minimum TLS Version** - TLS 1.2
- ✅ **Opportunistic Encryption** - ON
- ✅ **TLS 1.3** - ON
- ⚪ **HTTP Strict Transport Security (HSTS)** - Enable if you're sure (forces HTTPS permanently)

### Firewall Rules (Optional but Recommended)

**Create rules to:**

1. **Block known bad bots:**
   - Field: Known Bots
   - Operator: equals
   - Value: On
   - Action: Block

2. **Rate limit login endpoints:**
   - Field: URI Path
   - Operator: equals
   - Value: `/login` or `/api/auth`
   - Rate: 5 requests per 10 seconds
   - Action: Challenge or Block

3. **Allow only specific countries (if applicable):**
   - Field: Country
   - Operator: not equals
   - Value: US, CA, UK (your allowed countries)
   - Action: Challenge or Block

### Caching Settings

**For better performance:**

1. **Browser Cache TTL:** Respect Existing Headers
2. **Caching Level:** Standard
3. **Cache Everything Page Rule** (for static sites):
   - URL: `example.com/*`
   - Setting: Cache Level = Cache Everything
   - Edge Cache TTL: 2 hours

### Verify Configuration

```bash
# Check if Cloudflare proxy is active
dig app1.example.com

# Should show Cloudflare IPs (104.x.x.x or similar), not your server IP

# Test SSL from Cloudflare
curl -I https://app1.example.com

# Check SSL certificate issuer
echo | openssl s_client -servername app1.example.com -connect app1.example.com:443 2>/dev/null | grep "issuer"
```

### Cloudflare + Let's Encrypt Best Practice

**Recommended setup:**
- Cloudflare SSL mode: **Full (strict)**
- Your server: Let's Encrypt SSL certificate
- Cloudflare proxy: **Enabled (orange cloud)**

**This gives you:**
- DDoS protection from Cloudflare
- Valid SSL on your server
- End-to-end encryption
- Free SSL certificates from Let's Encrypt

---

## Step 9: Auto-Renewal Setup

### Verify Automatic Renewal

Certbot automatically sets up renewal via systemd timer:

```bash
# Check renewal timer status
sudo systemctl status certbot.timer

# Should show: Active: active (waiting)

# List all systemd timers
systemctl list-timers | grep certbot
```

**Expected output:**
```
NEXT                         LEFT          LAST                         PASSED       UNIT           ACTIVATES
Fri 2026-02-14 03:00:00 UTC  12h left      Thu 2026-02-13 03:00:00 UTC  9h ago       certbot.timer  certbot.service
```

### Test Renewal (Dry Run)

```bash
# Test renewal without actually renewing
sudo certbot renew --dry-run

# Should show: Congratulations, all simulated renewals succeeded
```

**This tests:**
- Certificate expiration check
- Domain validation (HTTP-01 or DNS-01)
- Renewal hooks execution
- Nginx reload

### Manual Renewal

```bash
# Force renewal now (if <30 days until expiry)
sudo certbot renew

# Force renewal even if not near expiry
sudo certbot renew --force-renewal

# Renew specific certificate only
sudo certbot renew --cert-name example.com
```

### Configure Post-Renewal Hooks

**Reload Nginx after certificate renewal:**

```bash
# Create deploy hook script
sudo nano /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

**Add:**
```bash
#!/bin/bash
# Reload Nginx after successful certificate renewal
systemctl reload nginx
```

```bash
# Make executable
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

**Hook types:**
- `pre` - Runs before renewal attempt
- `post` - Runs after renewal attempt (success or failure)
- `deploy` - Runs only after successful renewal

**Why reload Nginx?**
- Nginx caches SSL certificates in memory
- Must reload to use new certificates
- Prevents "certificate expired" errors

### Monitor Renewal Logs

```bash
# View Certbot logs
sudo tail -f /var/log/letsencrypt/letsencrypt.log

# Check last renewal
sudo certbot certificates

# Expected output shows expiry dates
```

### Email Notifications

Certbot sends email notifications:
- 20 days before expiry
- 10 days before expiry
- 1 day before expiry

**Ensure email is configured:**
```bash
# Update email if needed
sudo certbot update_account --email newemail@example.com
```

### Renewal Schedule

**Default schedule:**
- Runs twice daily (random time)
- Renews if certificate expires in <30 days
- Let's Encrypt certificates valid for 90 days

**Best practice:**
- Let automatic renewal handle it
- Monitor logs occasionally
- Test dry-run quarterly

---

## Step 10: Security Hardening

### Install Fail2Ban

**Fail2Ban prevents brute force attacks by banning IPs after failed attempts.**

```bash
# Install Fail2Ban
sudo apt install fail2ban -y

# Copy default config
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

### Configure Fail2Ban

```bash
# Edit local configuration
sudo nano /etc/fail2ban/jail.local
```

**Add/modify:**

```ini
[DEFAULT]
# Ban for 1 hour
bantime = 3600

# 5 failures in 10 minutes = ban
findtime = 600
maxretry = 5

# Email notifications (optional)
destemail = your@email.com
sendername = Fail2Ban
action = %(action_mwl)s

[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log
maxretry = 3
bantime = 7200

[nginx-http-auth]
enabled = true
filter = nginx-http-auth
port = http,https
logpath = /var/log/nginx/error.log

[nginx-noscript]
enabled = true
port = http,https
logpath = /var/log/nginx/access.log
maxretry = 6

[nginx-badbots]
enabled = true
port = http,https
logpath = /var/log/nginx/access.log
maxretry = 2

[nginx-noproxy]
enabled = true
port = http,https
logpath = /var/log/nginx/access.log
maxretry = 2

[nginx-limit-req]
enabled = true
filter = nginx-limit-req
port = http,https
logpath = /var/log/nginx/error.log
```

**Start Fail2Ban:**

```bash
# Enable and start
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Check status
sudo fail2ban-client status

# Check specific jail
sudo fail2ban-client status sshd
```

### SSH Hardening

```bash
# Edit SSH config
sudo nano /etc/ssh/sshd_config
```

**Recommended changes:**

```bash
# Disable root login
PermitRootLogin no

# Disable password authentication (use SSH keys only)
PasswordAuthentication no

# Only allow specific users (replace 'yourusername')
AllowUsers yourusername

# Change default port (optional, security through obscurity)
# Port 2222

# Disable empty passwords
PermitEmptyPasswords no

# Limit authentication attempts
MaxAuthTries 3

# Disconnect after 2 minutes of inactivity
ClientAliveInterval 120
ClientAliveCountMax 0
```

**Restart SSH:**

```bash
sudo systemctl restart sshd
```

**⚠️ WARNING:** Test SSH connection in a new terminal BEFORE closing your current session!

### Regular Security Updates

```bash
# Enable automatic security updates
sudo apt install unattended-upgrades -y

# Configure
sudo dpkg-reconfigure --priority=low unattended-upgrades

# Manual update check
sudo apt update && sudo apt upgrade -y
```

### Disable Unnecessary Services

```bash
# List running services
systemctl list-units --type=service --state=running

# Disable unused services (examples)
sudo systemctl disable apache2  # If you installed Nginx
sudo systemctl disable cups      # Print server (if not needed)
```

### Configure File Permissions

```bash
# Secure Nginx config files
sudo chmod 644 /etc/nginx/nginx.conf
sudo chmod 644 /etc/nginx/sites-available/*

# Secure SSL certificates
sudo chmod 600 /etc/letsencrypt/live/*/privkey.pem
sudo chmod 644 /etc/letsencrypt/live/*/fullchain.pem

# Secure credentials
sudo chmod 400 /etc/letsencrypt/cloudflare/credentials.ini
```

### Enable AppArmor (if not enabled)

```bash
# Check status
sudo aa-status

# Enable AppArmor
sudo systemctl enable apparmor
sudo systemctl start apparmor
```

### Install and Configure ModSecurity (Web Application Firewall)

**Optional but highly recommended for production:**

```bash
# Install ModSecurity
sudo apt install libnginx-mod-security -y

# Enable ModSecurity
sudo cp /usr/share/modsecurity-crs/crs-setup.conf.example /etc/nginx/modsecurity/crs-setup.conf
sudo cp /usr/share/modsecurity-crs/rules/*.conf /etc/nginx/modsecurity/rules/

# Configure in Nginx
sudo nano /etc/nginx/modsecurity/modsecurity.conf
```

**Change:**
```
SecRuleEngine On
```

### Regular Security Audits

```bash
# Check for rootkits
sudo apt install rkhunter -y
sudo rkhunter --check

# Check open ports
sudo ss -tulpn

# Check logged in users
who

# Check last logins
last

# Check failed login attempts
sudo grep "Failed password" /var/log/auth.log
```

---

## Monitoring & Maintenance

### Monitor Nginx

```bash
# Real-time access log
sudo tail -f /var/log/nginx/access.log

# Real-time error log
sudo tail -f /var/log/nginx/error.log

# Filter for errors only
sudo grep "error" /var/log/nginx/error.log

# Check Nginx status
sudo systemctl status nginx

# Test configuration
sudo nginx -t

# Reload without downtime
sudo systemctl reload nginx
```

### Monitor SSL Certificates

```bash
# List all certificates and expiry dates
sudo certbot certificates

# Check specific domain
echo | openssl s_client -servername app1.example.com -connect app1.example.com:443 2>/dev/null | openssl x509 -noout -dates

# Monitor renewal logs
sudo tail -f /var/log/letsencrypt/letsencrypt.log
```

### Monitor Firewall

```bash
# UFW status
sudo ufw status verbose

# Recent firewall blocks
sudo grep "UFW BLOCK" /var/log/kern.log

# Real-time firewall log
sudo tail -f /var/log/ufw.log
```

### Monitor Fail2Ban

```bash
# Status of all jails
sudo fail2ban-client status

# Banned IPs in specific jail
sudo fail2ban-client status sshd

# Unban an IP
sudo fail2ban-client unban 192.168.1.100

# Check Fail2Ban logs
sudo tail -f /var/log/fail2ban.log
```

### Monitor Docker Containers

```bash
# Container status
docker ps

# Container logs
docker logs app1 --tail 100 -f

# Container resource usage
docker stats

# Docker Compose status
docker-compose ps
```

### System Monitoring

```bash
# Disk usage
df -h

# Memory usage
free -h

# CPU usage
top
# or
htop  # (install with: sudo apt install htop)

# Network connections
sudo ss -tulpn

# System load
uptime
```

### Log Rotation

**Nginx logs are automatically rotated by default. Verify:**

```bash
# Check logrotate config for Nginx
cat /etc/logrotate.d/nginx
```

**Should contain:**
```
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```

### Set Up Monitoring Alerts (Optional)

**Simple email alerts for critical events:**

```bash
# Install mailutils
sudo apt install mailutils -y

# Test email
echo "Test email from server" | mail -s "Test Subject" your@email.com
```

**Create monitoring script:**

```bash
sudo nano /usr/local/bin/server-monitor.sh
```

```bash
#!/bin/bash
# Simple server monitoring script

# Check if Nginx is running
if ! systemctl is-active --quiet nginx; then
    echo "Nginx is down!" | mail -s "ALERT: Nginx Down" your@email.com
fi

# Check disk space (alert if >90% used)
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 90 ]; then
    echo "Disk usage is at ${DISK_USAGE}%" | mail -s "ALERT: Disk Space" your@email.com
fi

# Check SSL certificate expiry (alert if <30 days)
for cert in /etc/letsencrypt/live/*/cert.pem; do
    EXPIRY=$(openssl x509 -enddate -noout -in "$cert" | cut -d= -f2)
    EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
    NOW_EPOCH=$(date +%s)
    DAYS_LEFT=$(( ($EXPIRY_EPOCH - $NOW_EPOCH) / 86400 ))
    
    if [ $DAYS_LEFT -lt 30 ]; then
        echo "Certificate $cert expires in $DAYS_LEFT days" | mail -s "ALERT: SSL Expiry" your@email.com
    fi
done
```

```bash
# Make executable
sudo chmod +x /usr/local/bin/server-monitor.sh

# Add to cron (run daily at 8 AM)
sudo crontab -e

# Add line:
0 8 * * * /usr/local/bin/server-monitor.sh
```

### Backup Strategy

**Critical files to backup:**

```bash
# Nginx configurations
/etc/nginx/

# SSL certificates (or just keep renewal working)
/etc/letsencrypt/

# Docker compose files
/path/to/your/docker-compose.yml

# Application data volumes
/var/lib/docker/volumes/
# OR your bind mount paths

# UFW rules
sudo ufw status numbered > ~/ufw-backup.txt

# Fail2Ban config
/etc/fail2ban/jail.local
```

**Simple backup script:**

```bash
#!/bin/bash
BACKUP_DIR="/backup/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Backup Nginx configs
sudo tar -czf $BACKUP_DIR/nginx-config.tar.gz /etc/nginx/

# Backup Docker volumes
docker run --rm -v postgres-data:/data -v $BACKUP_DIR:/backup alpine tar -czf /backup/postgres-data.tar.gz -C /data .

# Backup Let's Encrypt
sudo tar -czf $BACKUP_DIR/letsencrypt.tar.gz /etc/letsencrypt/

echo "Backup completed: $BACKUP_DIR"
```

---

## Troubleshooting

### SSL Certificate Issues

**Problem: Certificate not working**

```bash
# Check certificate status
sudo certbot certificates

# Test certificate
curl -vI https://app1.example.com

# Check Nginx SSL config
sudo nginx -t

# Verify certificate files exist
sudo ls -la /etc/letsencrypt/live/app1.example.com/
```

**Problem: Certificate renewal fails**

```bash
# Check renewal logs
sudo cat /var/log/letsencrypt/letsencrypt.log

# Test renewal dry-run
sudo certbot renew --dry-run

# Common causes:
# - Port 80 blocked (HTTP-01)
# - Cloudflare API token expired/invalid (DNS-01)
# - DNS propagation issues (DNS-01)
# - Nginx not serving /.well-known/acme-challenge/
```

**Fix for HTTP-01:**
```bash
# Ensure port 80 is accessible
sudo ufw allow 80/tcp

# Test ACME challenge path
curl http://app1.example.com/.well-known/acme-challenge/test

# Nginx must serve from /var/www/html
```

**Fix for DNS-01:**
```bash
# Verify Cloudflare credentials
sudo cat /etc/letsencrypt/cloudflare/credentials.ini

# Test API token manually
curl -X GET "https://api.cloudflare.com/client/v4/user/tokens/verify" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json"
```

### Nginx Issues

**Problem: Nginx won't start**

```bash
# Check error logs
sudo journalctl -u nginx -n 50

# Test configuration
sudo nginx -t

# Common errors:
# - Syntax error in config file
# - Port already in use
# - SSL certificate file not found
```

**Problem: 502 Bad Gateway**

```bash
# Means Nginx can't reach backend (Docker container)

# Check if container is running
docker ps

# Check if port is correct
sudo ss -tulpn | grep :3000

# Test direct connection to container
curl http://127.0.0.1:3000

# Check Nginx error log
sudo tail -f /var/log/nginx/error.log
```

**Problem: 404 Not Found**

```bash
# Check server_name matches request
# Check location blocks
# Verify proxy_pass target is correct
```

### Firewall Issues

**Problem: Can't connect to website**

```bash
# Check if UFW is blocking
sudo ufw status

# Ensure ports 80 and 443 are allowed
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Check if Nginx is listening
sudo ss -tulpn | grep nginx

# Should show:
# 0.0.0.0:80 and 0.0.0.0:443
```

**Problem: Locked out via SSH**

```bash
# Prevention: Always allow SSH before enabling UFW
sudo ufw allow OpenSSH
sudo ufw enable

# If locked out: Access via console (cloud provider dashboard)
# Then: sudo ufw allow 22/tcp
```

### Docker Issues

**Problem: Container not accessible via Nginx**

```bash
# Check container is running
docker ps | grep app1

# Check port binding
docker port app1
# Should show: 3000/tcp -> 127.0.0.1:3000

# Test direct access
curl http://127.0.0.1:3000

# If this works but Nginx fails, check Nginx config
```

**Problem: Container won't start**

```bash
# Check logs
docker logs app1

# Check docker-compose logs
docker-compose logs app1

# Common issues:
# - Port already in use
# - Missing environment variables
# - Volume mount errors
```

### Cloudflare Issues

**Problem: Infinite redirect loop**

**Cause:** Cloudflare SSL set to "Flexible" but Nginx redirects HTTP to HTTPS

**Fix:**
1. Go to Cloudflare → SSL/TLS
2. Change to "Full (strict)"
3. Ensure Let's Encrypt certificate is installed on server

**Problem: "Too many redirects"**

```bash
# Check Nginx config for redirect loops
sudo nginx -t

# Common issue: Both Cloudflare and Nginx redirecting

# Fix: Disable Cloudflare redirect OR Nginx redirect, not both
```

**Problem: 520/521/522 Errors**

- **520:** Unknown error - Check Nginx error logs
- **521:** Web server is down - Start Nginx
- **522:** Connection timed out - Check firewall, ensure port 443 open

```bash
# Check Nginx status
sudo systemctl status nginx

# Check UFW
sudo ufw status

# Check if Nginx is listening
sudo ss -tulpn | grep :443
```

### Performance Issues

**Problem: Slow website**

```bash
# Check server load
uptime

# Check memory
free -h

# Check disk I/O
iostat

# Check Nginx access log for slow requests
sudo tail -f /var/log/nginx/access.log

# Enable Nginx caching (add to server block)
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m use_temp_path=off;

location / {
    proxy_cache my_cache;
    proxy_cache_valid 200 1h;
    proxy_pass http://127.0.0.1:3000;
}
```

### General Debugging Commands

```bash
# Check all services status
sudo systemctl status nginx docker certbot.timer fail2ban ufw

# Check open ports
sudo ss -tulpn

# Check running processes
ps aux | grep nginx
ps aux | grep docker

# Check system logs
sudo journalctl -xe

# Check Nginx version
nginx -V

# Check Docker version
docker --version

# Check Ubuntu version
lsb_release -a

# Check disk space
df -h

# Check memory usage
free -h

# Network connectivity test
ping -c 4 8.8.8.8
curl -I https://www.cloudflare.com
```

---

## Quick Reference Commands

### Nginx

```bash
# Test configuration
sudo nginx -t

# Reload (no downtime)
sudo systemctl reload nginx

# Restart
sudo systemctl restart nginx

# Stop
sudo systemctl stop nginx

# Start
sudo systemctl start nginx

# View error log
sudo tail -f /var/log/nginx/error.log

# View access log
sudo tail -f /var/log/nginx/access.log
```

### Certbot

```bash
# List certificates
sudo certbot certificates

# Renew all certificates
sudo certbot renew

# Renew with dry-run test
sudo certbot renew --dry-run

# Obtain new certificate (HTTP-01)
sudo certbot --nginx -d example.com

# Obtain new certificate (DNS-01)
sudo certbot certonly --dns-cloudflare --dns-cloudflare-credentials /etc/letsencrypt/cloudflare/credentials.ini -d example.com -d '*.example.com'

# Delete certificate
sudo certbot delete --cert-name example.com
```

### UFW

```bash
# Status
sudo ufw status verbose

# Enable
sudo ufw enable

# Disable
sudo ufw disable

# Allow port
sudo ufw allow 80/tcp

# Delete rule
sudo ufw delete allow 80/tcp

# Reset (delete all rules)
sudo ufw reset
```

### Docker

```bash
# List running containers
docker ps

# List all containers
docker ps -a

# View logs
docker logs app1 -f

# Start container
docker start app1

# Stop container
docker stop app1

# Restart container
docker restart app1

# Execute command in container
docker exec -it app1 /bin/bash

# Docker Compose up
docker-compose up -d

# Docker Compose down
docker-compose down

# View compose logs
docker-compose logs -f
```

### Fail2Ban

```bash
# Status
sudo fail2ban-client status

# Status of specific jail
sudo fail2ban-client status sshd

# Unban IP
sudo fail2ban-client unban 192.168.1.100

# Reload config
sudo fail2ban-client reload
```

---

## Security Checklist

Before going to production, verify:

- [ ] UFW enabled with only ports 22, 80, 443 allowed
- [ ] SSH configured with key-only authentication
- [ ] SSH root login disabled
- [ ] Fail2Ban installed and configured
- [ ] Let's Encrypt SSL certificates installed
- [ ] Nginx SSL configuration uses TLS 1.2+ only
- [ ] HSTS header enabled in Nginx
- [ ] Docker containers bound to 127.0.0.1 only
- [ ] Cloudflare SSL mode set to "Full (strict)"
- [ ] Regular security updates enabled
- [ ] Certificate auto-renewal tested
- [ ] Backup strategy in place
- [ ] Monitoring/logging configured
- [ ] Unnecessary services disabled
- [ ] Strong passwords/secrets used
- [ ] File permissions properly set

---

## Conclusion

This production-grade setup provides:

✅ **Security**
- End-to-end SSL encryption
- Firewall protection
- DDoS mitigation (with Cloudflare)
- Intrusion prevention (Fail2Ban)
- Isolated containers

✅ **Reliability**
- Automatic SSL renewal
- No downtime deployments (Nginx reload)
- Health monitoring
- Automatic restarts (Docker)

✅ **Scalability**
- Easy to add new domains/apps
- Centralized SSL management
- Single Nginx instance for all domains
- Docker Compose orchestration

✅ **Maintainability**
- Clear configuration structure
- Comprehensive logging
- Automated updates
- Simple troubleshooting

✅ **Performance**
- HTTP/2 support
- SSL session caching
- Proxy buffering
- Optional CDN (Cloudflare)

---

**Last Updated:** February 2026  
**Tested On:** Ubuntu 24.04 LTS, 22.04 LTS  
**Nginx Version:** 1.24+  
**Certbot Version:** 2.x+

For questions or updates, refer to official documentation:
- [Nginx Documentation](https://nginx.org/en/docs/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Certbot Documentation](https://eff-certbot.readthedocs.io/)
- [Cloudflare Documentation](https://developers.cloudflare.com/)
- [Docker Documentation](https://docs.docker.com/)
- [UFW Documentation](https://help.ubuntu.com/community/UFW)
