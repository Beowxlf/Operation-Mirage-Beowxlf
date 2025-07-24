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

