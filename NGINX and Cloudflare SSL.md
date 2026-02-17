# Complete Guide: Setup Subdomain with Nginx, SSL & Auto-Renewal

This guide shows how to set up a subdomain pointing to a Docker container with automatic SSL certificate renewal using Cloudflare DNS authentication.

## Prerequisites

- Nginx installed
- Certbot with Cloudflare plugin installed
- Domain managed by Cloudflare
- Cloudflare API token with DNS edit permissions
- Docker container running (e.g., on port 9090)

---

## Step 1: Create Subdomain in Cloudflare

1. Log in to Cloudflare Dashboard
2. Select your domain (e.g., `touchit.click`)
3. Go to **DNS** → **Records**
4. Click **Add record**
5. Configure:
   - **Type**: A
   - **Name**: your-subdomain (e.g., `kano`)
   - **IPv4 address**: Your server's public IP
   - **Proxy status**: Either Gray cloud (DNS only) or Orange cloud (Proxied) - both work with DNS challenge
   - **TTL**: Auto
6. Click **Save**

**Result**: Your subdomain will be `your-subdomain.touchit.click`

---

## Step 2: Verify Cloudflare Credentials

Make sure your Cloudflare API credentials file exists:
```bash
sudo nano /etc/letsencrypt/cloudflare.ini
```

It should contain:
```ini
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN
```

**To get your API token** (if you don't have one):
1. Go to: https://dash.cloudflare.com/profile/api-tokens
2. Click **Create Token**
3. Use **Edit zone DNS** template
4. Zone Resources: Include → Specific zone → `touchit.click`
5. Continue to summary → **Create Token**
6. Copy the token and paste it in the cloudflare.ini file

Secure the file:
```bash
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
```

---

## Step 3: Get SSL Certificate

Replace the values in the command below:
- `your-subdomain.touchit.click` - your full subdomain
- `your-email@example.com` - your email address
```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d your-subdomain.touchit.click \
  --email your-email@example.com \
  --agree-tos \
  --non-interactive
```

**Expected output**: Certificate successfully obtained and saved.

Certificate files will be at:
- `/etc/letsencrypt/live/your-subdomain.touchit.click/fullchain.pem`
- `/etc/letsencrypt/live/your-subdomain.touchit.click/privkey.pem`

---

## Step 4: Create Nginx Configuration

Create a new Nginx configuration file:
```bash
sudo nano /etc/nginx/sites-available/your-subdomain.touchit.click
```

Add this configuration (replace `your-subdomain.touchit.click` and port `9090` with your values):
```nginx
# HTTP - Redirect all traffic to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name your-subdomain.touchit.click;
    return 301 https://$server_name$request_uri;
}

# HTTPS - Main configuration
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name your-subdomain.touchit.click;

    # SSL Certificate
    ssl_certificate /etc/letsencrypt/live/your-subdomain.touchit.click/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-subdomain.touchit.click/privkey.pem;
    
    # SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;

    # Reverse Proxy to Docker Container
    location / {
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        
        # WebSocket support (if needed)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

**Save and exit**: `Ctrl + X`, then `Y`, then `Enter`

---

## Step 5: Enable the Site

Create a symbolic link to enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/your-subdomain.touchit.click /etc/nginx/sites-enabled/
```

Test Nginx configuration:
```bash
sudo nginx -t
```

**Expected output**: `syntax is okay` and `test is successful`

Reload Nginx:
```bash
sudo systemctl reload nginx
```

---

## Step 6: Test Your Setup

### Test HTTP to HTTPS redirect:
```bash
curl -I http://your-subdomain.touchit.click
```

Should return: `301 Moved Permanently` with `Location: https://...`

### Test HTTPS:
```bash
curl -I https://your-subdomain.touchit.click
```

Should return: `200 OK` (or your app's response)

### Visit in browser:

Open: `https://your-subdomain.touchit.click`

You should see:
- Green padlock (secure connection)
- Your Docker application running

---

## Step 7: Verify Auto-Renewal

Test the renewal process (dry run - doesn't actually renew):
```bash
sudo certbot renew --dry-run
```

**Expected output**: All simulations should renew successfully.

Check the renewal timer status:
```bash
sudo systemctl status certbot.timer
```

Should show: `active (waiting)`

---

## Certificate Auto-Renewal Details

- **Renewal frequency**: Certbot checks twice daily
- **Renewal timing**: Certificates renew automatically 30 days before expiration
- **Certificate validity**: 90 days
- **Method**: Cloudflare DNS challenge (automatic, no downtime)
- **Nginx reload**: Automatic after renewal

---

## Troubleshooting

### Issue: "Certificate not found" error in Nginx

**Solution**: Check certificate path matches your subdomain:
```bash
sudo ls -la /etc/letsencrypt/live/
```

### Issue: 502 Bad Gateway

**Solution**: 
1. Check if your Docker container is running:
```bash
   docker ps
```
2. Verify the port (9090) is correct:
```bash
   curl http://127.0.0.1:9090
```

### Issue: Nginx configuration test fails

**Solution**: Check syntax errors:
```bash
sudo nginx -t
```

### Issue: Certbot renewal fails

**Solution**: 
1. Check Cloudflare API token is valid
2. Verify credentials file permissions:
```bash
   ls -la /etc/letsencrypt/cloudflare.ini
```
   Should show: `-rw-------` (600)

### Issue: DNS not resolving

**Solution**: Verify DNS record in Cloudflare and wait 1-2 minutes for propagation:
```bash
dig +short your-subdomain.touchit.click
```

---

## Managing Multiple Subdomains

To add more subdomains, repeat this process:

1. Create new DNS record in Cloudflare
2. Get certificate: `sudo certbot certonly --dns-cloudflare --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini -d new-subdomain.touchit.click`
3. Create new Nginx config in `/etc/nginx/sites-available/`
4. Enable and reload

All certificates will auto-renew together.

---

## Useful Commands

### List all certificates:
```bash
sudo certbot certificates
```

### Manually renew a specific certificate:
```bash
sudo certbot renew --cert-name your-subdomain.touchit.click
```

### Delete a certificate:
```bash
sudo certbot delete --cert-name your-subdomain.touchit.click
```

### Check Nginx error logs:
```bash
sudo tail -f /var/log/nginx/error.log
```

### Check Certbot logs:
```bash
sudo tail -f /var/log/letsencrypt/letsencrypt.log
```

### Restart Nginx:
```bash
sudo systemctl restart nginx
```

### Check Nginx status:
```bash
sudo systemctl status nginx
```

---

## Security Notes

1. **Keep Cloudflare API token secure** - never commit to git or share publicly
2. **File permissions** - cloudflare.ini should be 600 (readable only by root)
3. **SSL protocols** - Configuration uses TLSv1.2 and TLSv1.3 only (secure)
4. **Cloudflare SSL Mode** - Set to "Full (strict)" in Cloudflare for best security

---

## Quick Reference: Complete Setup for New Subdomain
```bash
# 1. Create DNS record in Cloudflare (A record pointing to server IP)

# 2. Get SSL certificate
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d new-subdomain.touchit.click \
  --email your-email@example.com \
  --agree-tos

# 3. Create Nginx config
sudo nano /etc/nginx/sites-available/new-subdomain.touchit.click
# (paste configuration from Step 4 above, update subdomain and port)

# 4. Enable site
sudo ln -s /etc/nginx/sites-available/new-subdomain.touchit.click /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# 5. Test
curl -I https://new-subdomain.touchit.click
```

---

## Example: Setting Up kano.touchit.click for Port 9090
```bash
# 1. DNS: Add A record "kano" pointing to your server IP in Cloudflare

# 2. Get certificate
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d kano.touchit.click \
  --email admin@touchit.click \
  --agree-tos

# 3. Create Nginx config
sudo nano /etc/nginx/sites-available/kano.touchit.click
```

Paste this configuration:
```nginx
server {
    listen 80;
    listen [::]:80;
    server_name kano.touchit.click;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name kano.touchit.click;

    ssl_certificate /etc/letsencrypt/live/kano.touchit.click/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/kano.touchit.click/privkey.pem;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;

    location / {
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```
```bash
# 4. Enable and reload
sudo ln -s /etc/nginx/sites-available/kano.touchit.click /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# 5. Test
curl -I https://kano.touchit.click
```

**Done!** Visit `https://kano.touchit.click` in your browser.

---

## Summary

✅ SSL certificate obtained via Cloudflare DNS challenge  
✅ Nginx reverse proxy configured  
✅ HTTP to HTTPS redirect enabled  
✅ Auto-renewal configured (every 60 days)  
✅ Secure SSL/TLS configuration  
✅ WebSocket support included  

Your subdomain is now live with automatic SSL renewal!
