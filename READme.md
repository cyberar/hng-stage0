# HNG DevOps Stage 0: Linux Server Setup, Nginx Configuration & SSL with Let's Encrypt

## Introduction

This is a complete walkthrough of the HNG Internship 14 DevOps Track Stage 0 task. The goal is to provision a Linux server from scratch, harden it, configure Nginx to serve a static page and a JSON API endpoint, and secure everything with a valid SSL certificate from Let's Encrypt - no Docker, no Compose, no automation tools.

By the end of this guide, you will have:

- A hardened Ubuntu server with a non-root user and key-based SSH only
- Nginx serving a static HTML page at `/` and a JSON response at `/api`
- A valid Let's Encrypt SSL certificate with automatic HTTP -> HTTPS (301) redirect
- UFW firewall allowing only the required ports

---

## Prerequisites

- A cloud provider account (AWS, GCP, DigitalOcean, Hetzner, etc.)
- A domain name with DNS access (required for Let's Encrypt)
- Your local machine terminal (PowerShell on Windows, Terminal on Mac/Linux)
- Basic familiarity with the command line

---

## Phase 1: Provision the Server

### Step 1.1 — Launch an Ubuntu server

Spin up an **Ubuntu 22.04 LTS** server on your preferred cloud provider. For AWS EC2:

1. Go to EC2 → Launch Instance
2. Choose **Ubuntu Server 22.04 LTS**
3. Select an instance type (t2.micro is sufficient for this task)
4. Under **Key pair**, create a new key pair or use an existing one — download the `.pem` file and keep it safe
5. Allow inbound traffic on ports 22, 80, and 443 in the security group
6. Launch the instance and note the public IP address

### Step 1.2 — Point your domain to the server

Before setting up SSL, create an **A record** in your DNS provider pointing your domain to the server's public IP address and a **CNAME record**. For example:

```
kemicodes.online -> A record -> <your_server_public_IP_address>
kemicodes.online -> CNAME record -> <yourdomain_name>
```

DNS propagation can take a few minutes up to few hours. You can verify it with:

```bash
nslookup yourdomain.com
```

### Step 1.3 — SSH into the server

**On Windows (PowerShell):**

```powershell
# Fix permissions on the .pem file first if you saved it in the D drive (external drive)
icacls D:\your-key-name.pem /inheritance:r /grant:r "$env:USERNAME:(R)"

# Connect
ssh -i D:\your-key-name.pem ubuntu@YOUR_SERVER_IP
```

**On Mac/Linux:**

```bash
chmod 400 ~/Downloads/your-key-name.pem
ssh -i ~/Downloads/your-key-name.pem ubuntu@YOUR_SERVER_IP
```

> **Note:** The default user on AWS Ubuntu instances is `ubuntu`, not `root`.

---

## Phase 2: Server Hardening

### Step 2.1 — Create the hngdevops user

```bash
sudo adduser hngdevops
sudo usermod -aG sudo hngdevops
```

You will be prompted to set a password. Set one — you will need it for `sudo` commands.

### Step 2.2 — Set up SSH key authentication for hngdevops

Copy your public key to the hngdevops user's authorized_keys file:

```bash
sudo mkdir -p /home/hngdevops/.ssh
sudo cp /home/ubuntu/.ssh/authorized_keys /home/hngdevops/.ssh/authorized_keys
sudo chown -R hngdevops:hngdevops /home/hngdevops/.ssh
sudo chmod 700 /home/hngdevops/.ssh
sudo chmod 600 /home/hngdevops/.ssh/authorized_keys
```

Now test that you can log in as hngdevops **in a new terminal before continuing**:

```bash
ssh -i D:\your-key-name.pem hngdevops@YOUR_SERVER_IP
```

> **Critical:** Keep your original session open until you confirm the new login works.

### Step 2.3 — Configure passwordless sudo for specific commands

The task requires hngdevops to run `sshd -T` and `ufw status` without a password prompt - but only those two commands, not everything.

```bash
sudo visudo -f /etc/sudoers.d/hngdevops
```

Add this exact line:

```
hngdevops ALL=(root) NOPASSWD:/usr/sbin/sshd,/usr/sbin/ufw
```

Save and exit. Test it:

```bash
sudo /usr/sbin/sshd -T | head -5
sudo /usr/sbin/ufw status
```

Neither command should ask for a password.

### Step 2.4 — Harden SSH configuration

```bash
sudo nano /etc/ssh/sshd_config
```

Find and set these values (or add them if missing):

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

> **Warning:** Test login in a new terminal after this step. If key-based auth fails, you will be locked out.

### Step 2.5 — Configure UFW firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

The output should show `Status: active` with only ports 22, 80, and 443 allowed.

---

## Phase 3: Install and Configure Nginx

### Step 3.1 — Install Nginx

```bash
sudo apt update && sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

### Step 3.2 — Create the static HTML page

```bash
sudo nano /var/www/html/index.html
```

Paste the following (replace `HNG_USERNAME` with your actual username or any name of your choice):

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>HNG DevOps Stage 0</title>
</head>
<body>
  <h1>HNG_USERNAME</h1>
  <p>HNG Internship DevOps Track - Stage 0</p>
</body>
</html>
```

> **Important:** The username must be visible text in the page body. Do not hide it with CSS or put it only in a comment.

### Step 3.3 - Create the Nginx server block

```bash
sudo nano /etc/nginx/sites-available/hng
```

Paste the following configuration (replace `yourdomain.com` and `HNG_USERNAME`):

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # Redirect all HTTP traffic to HTTPS with a 301
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name yourdomain.com www.yourdomain.com;

    # SSL certificates - Certbot will populate these in the next phase
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    root /var/www/html;
    index index.html;

    # GET / - serve the static HTML page
    location / {
        try_files $uri $uri/ =404;
    }

    # GET /api - return JSON response
    location = /api {
        add_header Content-Type application/json;
        return 200 '{"message":"HNGI14 Stage 0","track":"DevOps","username":"HNG_USERNAME"}';
    }
}
```

> **Critical:** The `return 301` redirect must only exist in the port 80 block. If it appears in the 443 block as well, you will get an infinite redirect loop.

### Step 3.4 - Enable the site

```bash
sudo ln -s /etc/nginx/sites-available/hng /etc/nginx/sites-enabled/
sudo nginx -t
```

The output must say `syntax is ok` and `test is successful`. If there are errors, review the config file for typos.

> **Note:** Do not reload Nginx yet — the SSL cert paths in the config don't exist until Certbot runs in the next step. Reloading now will cause an error.

---

## Phase 4: SSL with Let's Encrypt (Certbot)

### Step 4.1 - Install Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### Step 4.2 - Obtain the SSL certificate

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Certbot will:
1. Ask for your email address (for renewal alerts)
2. Ask you to agree to the Terms of Service
3. Verify domain ownership via HTTP challenge on port 80
4. Issue the certificate and update your Nginx config automatically

> **If Certbot fails:** Make sure port 80 is open in UFW (`sudo ufw status`) and that your DNS A record is pointing to the correct IP. Certbot uses HTTP on port 80 to verify you own the domain.

### Step 4.3 - Verify auto-renewal (Optional)

Let's Encrypt certificates expire every 90 days. Certbot installs a cron job or systemd timer to handle renewal automatically. Test it:

```bash
sudo certbot renew --dry-run
```

You should see: `Congratulations, all renewals succeeded.`

Confirm the Nginx config and ensure there are no typos.

### Step 4.4 - Reload Nginx

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## Phase 5: Verification

Run each of these checks to confirm everything is working correctly.

### Check HTTP to HTTPS redirect (must be 301)

```bash
curl -I http://yourdomain.com
```

Expected output includes:

```
HTTP/1.1 301 Moved Permanently
Location: https://yourdomain.com/
```

### Check the homepage loads over HTTPS

```bash
curl -I https://yourdomain.com
```

Expected: `HTTP/1.1 200 OK`

### Check the /api endpoint headers

```bash
curl -I https://yourdomain.com/api
```

Expected: `HTTP/1.1 200 OK` with `Content-Type: application/json`

### Check the /api JSON body

```bash
curl https://yourdomain.com/api
```

Expected output:

```json
{"message":"HNGI14 Stage 0","track":"DevOps","username":"hng-username"}
```

### Check the homepage contains your username

```bash
curl https://yourdomain.com
```

Your HNG username should appear as visible text in the HTML output.

### Final system checks

```bash
# UFW must be active
sudo ufw status

# Nginx must be running
sudo systemctl status nginx

# Passwordless sudo must work for hngdevops
sudo /usr/sbin/sshd -T | head -5
sudo /usr/sbin/ufw status
```

---

## Common Errors and Fixes

### "Too many redirects" in the browser

**Cause:** The `return 301` redirect exists in both the port 80 and port 443 server blocks, causing an infinite loop.

**Fix:** Remove the `return 301` line from the 443 block. It should only be in the port 80 block.

### "Permission denied (publickey)" on SSH

**Cause:** You are either pointing `-i` at the wrong file, or the public key was not added to `authorized_keys`.

**Fix:** The `-i` flag takes your **private key** (the `.pem` file on your local machine), not the `authorized_keys` file on the server.

```bash
ssh -i /path/to/your-key.pem hngdevops@YOUR_SERVER_IP
```

### Certbot fails with "connection refused"

**Cause:** Port 80 is not open, or Nginx is not running.

**Fix:**

```bash
sudo ufw allow 80/tcp
sudo systemctl start nginx
```

### SSH locked out after disabling password auth

**Cause:** You disabled `PasswordAuthentication` before adding your public key to `authorized_keys`.

**Fix:** Use your cloud provider's console/web terminal to access the server and restore the key.

---

## Conclusion

That's it! You now have a fully configured, hardened Linux server running Nginx with HTTPS, serving both a static HTML page and a JSON API endpoint. The setup covers the core fundamentals every DevOps engineer needs to know: user management, SSH hardening, firewall rules, web server configuration, and SSL certificate management.

If you found this walkthrough helpful, kindly star this repo and give a follow. Would love to connect with you. Happy coding!