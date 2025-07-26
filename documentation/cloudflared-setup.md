## Operation Mirage: Server & Tunnel Hardening (Project MATI)

This document captures every step taken to securely deploy the Operation Mirage web infrastructure — a hybrid deception portfolio designed to showcase not just what I know, but what I can build, harden, and operate.

---

### 🔧 Initial Setup

1. Downloaded **Ubuntu 24.04 Server ISO**
2. Created a Virtual Machine via VMware using the ISO
3. Used **default guided installation**
   - **User:** `REDACTED`  
   - **Password:** `REDACTED`
4. Took a VM **snapshot** immediately after installation
5. Verified DHCP lease and **assigned IP: `192.168.xx.xx`**
6. Configured **shared folder** between host and guest for media/assets transfer

> ✅ This machine will serve as the public-facing webserver for Operation Mirage.

---

### 🛠️ Post-Installation Security Steps

#### 1. System Patching
```bash
sudo apt update && sudo apt upgrade -y
```
- Ensures the OS and all base packages are up-to-date and protected against known CVEs

#### 2. Install Firewall + Brute Force Defense
```bash
sudo apt install ufw fail2ban -y
```
- `ufw`: uncomplicated firewall, used to lock down exposed ports
- `fail2ban`: monitors log files and bans IPs that show malicious signs (e.g. too many failed logins)

---

### 🔐 SSH Hardening & Key Authentication

Edited `/etc/ssh/sshd_config` to apply:
```text
AllowUsers beowulf
LoginGraceTime 1m
PermitRootLogin no
MaxAuthTries 3
MaxSessions 10
KerberosAuthentication no
UsePAM no
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 0
```

> ✅ These settings help defend against brute-force and session hijack attempts.

#### Key-Based SSH Authentication
On my **SOC analyst VM**, I ran:
```bash
ssh-keygen -t ed25519 -C "beowxlf-operation-mirage"
ssh-copy-id user@192.168.xx.xx
```

Then confirmed the key in:
```bash
cat ~/.ssh/authorized_keys
```

#### Legal Banner (Optional)
Created `/etc/issue.net` with a custom warning message, then enabled it in `sshd_config`:
```text
Banner /etc/issue.net
```

---

### 🌐 NGINX Installation & Configuration
```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```
- NGINX will serve all static HTML/CSS content and route traffic from Cloudflare

---

### ☁️ Cloudflared Tunnel Setup
This allows the server to be **publicly accessible** through a secure Cloudflare tunnel — no public IP or open ports needed.

#### 1. Install Cloudflared
```bash
wget -O cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb
```

#### 2. Authenticate & Link Domain
```bash
cloudflared login
```
- Authenticates with Cloudflare and selects domain for tunnel use

#### 3. Create Tunnel Identity
```bash
cloudflared tunnel create mirage-tunnel
```
- Generates `MIRAGE-TUNNEL-ID.json` and stores it in `~/.cloudflared`

#### 4. Create Tunnel Config File
```bash
sudo nano /etc/cloudflared/config.yml
```
Contents:
```yaml
tunnel: MIRAGE-TUNNEL-ID
credentials-file: /root/.cloudflared/MIRAGE-TUNNEL-ID.json

ingress:
  - hostname: operation-mirage-beowxlf.me
    service: http://localhost:80
  - service: http_status:404
```

#### 5. Bind Tunnel to Domain
```bash
cloudflared tunnel route dns mirage-tunnel operation-mirage-beowxlf.me
```

#### 6. Autostart on Boot
```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```
Verify:
```bash
sudo systemctl status cloudflared
```

---

### 🔥 UFW Firewall Lockdown
```bash
sudo ufw allow OpenSSH
sudo ufw allow from 127.0.0.1 to any port 80
sudo ufw enable
```
> ✅ This ensures NGINX is only accessible **locally** — Cloudflared is the only gateway.

---

### 🛡️ NGINX Security Hardening
Edited `/etc/nginx/sites-enabled/default`:
```nginx
add_header X-Content-Type-Options "nosniff";
add_header X-Frame-Options "SAMEORIGIN";
add_header X-XSS-Protection "1; mode=block";
add_header Referrer-Policy "no-referrer-when-downgrade";
add_header Permissions-Policy "geolocation=(), microphone=()";
autoindex off;
```

#### 📖 Header Explanations
- `X-Content-Type-Options: nosniff` — prevents MIME sniffing attacks
- `X-Frame-Options: SAMEORIGIN` — blocks clickjacking via iframe embedding
- `X-XSS-Protection: 1; mode=block` — enables basic XSS filter on legacy browsers
- `Referrer-Policy: no-referrer-when-downgrade` — protects sensitive referrer URLs
- `Permissions-Policy: geolocation=(), microphone=()` — disables unnecessary APIs
- `autoindex off` — prevents directory listing enumeration

---

### ✅ Verified Functionality
- Tested headers using:
```bash
curl -I http://localhost
```
- Validated tunnel from external IP
- Confirmed only 127.0.0.1 has access to port 80

---

### 🧠 What I Learned
- How to securely expose a web service using Cloudflare Tunnel
- SSH hardening strategies for public servers
- Basic Linux firewall and service management
- Manual NGINX configuration for maximum control
- How to create layered access controls for a deception-based web platform

---

### 📌 Next Steps
- Log ingestion (Wazuh or Sysmon → SOC VM)
- Begin `/projects/` page for MITRE T1059: PowerShell case log
- Auto-deploy GitHub repo to production on file change

> This documentation is part of the public “Operation Mirage” deception portfolio build. It lives in `operation-documentation/OP-Mirage/documentation.md`.

Total control. No shortcuts. Just systems, security, and structure.


## Installing cloudflared

#### Install cloudflared
Run the following commands in VM
```
wget -O cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb

```
Confirm its Installed
`cloudflared --version`

#### Authenticate with Cloudflare
Run this to link VM and CLoudflare account
`cloudflare login`
This will open a url in your browser. login to cloudflare and select the domain

#### Create a tunnel
Create a named tunnel (ex: mirage-tunnel):
`cloudflared tunnel create mirage-tunnel`
This generates a tunnel ID and credentials JSON (saved to `~/.cloudflared`)

#### Configure the Tunnel
Create this config file:
`sudo nano /etc/cloudflared/config.yml`
Paste the following:
```
tunnel: MIRAGE-TUNNEL-ID   # replace with actual ID
credentials-file: /root/.cloudflared/MIRAGE-TUNNEL-ID.json

ingress:
  - hostname: operation-mirage-beowxlf.me
    service: http://localhost:80
  - service: http_status:404

```
Then run:
`sudo cloudflared tunnel route dns mirage-tunnel operation-mirage-beowxlf.me`
This binds your domain to the tunnel

#### Enable Tunnel on Boot
Create and enable the service
```
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```
Check status:
`sudo systemctl status cloudflared`

#### Firewall Rules
Block everything except SSH and local host on port 80
```
sudo ufw allow OpenSSH
sudo ufw allow from 127.0.0.1 to any port 80
sudo ufw enable
```
NGINX is now only reachable through the Cloudflare tunnel

