# Self-Host Vaultwarden on a VPS with DuckDNS, Nginx, and HTTPS

This guide shows how to deploy Vaultwarden on an Ubuntu VPS and access it securely using a free DuckDNS hostname.

> **Important:** Vaultwarden is an unofficial, community-maintained Bitwarden-compatible server. It is not the official Bitwarden server. You are responsible for securing, updating, and backing up the service. Review the project documentation before storing important passwords.

## What you'll build

```text
Browser / Bitwarden-compatible app
              |
              | HTTPS (TCP 443)
              v
  yourname.duckdns.org
              |
              v
      Nginx reverse proxy
       + HTTPS certificate
              |
              | HTTP on localhost only
              v
  127.0.0.1:8000 -> Vaultwarden container:80
              |
              v
       Persistent data folder
```

Vaultwarden's port is bound to `127.0.0.1`, not the public network. That is intentional: users should reach it through Nginx and HTTPS, not by exposing port `8000` directly.

## Before you start

You need:

- An Ubuntu VPS with SSH access and a public IPv4 address.
- A DuckDNS account and a hostname, for example `yourname.duckdns.org`.
- Docker Engine and the Docker Compose plugin.
- Nginx installed on the VPS.
- Permission to open TCP ports `80` and `443` in your VPS/provider firewall.

Keep your SSH port open. If Nginx already serves other sites, do not overwrite their configuration. This guide adds a separate Nginx site for Vaultwarden.

In the commands below, replace `yourname.duckdns.org` with your DuckDNS hostname and `YOUR_VPS_IP` with your VPS public IP where applicable.

---

## 1. Point DuckDNS to the VPS

1. Sign in at [DuckDNS](https://www.duckdns.org/).
2. Create a subdomain, such as `yourname.duckdns.org`.
3. Set its IPv4 address to your VPS public IP.
4. If your VPS IP changes, update the DuckDNS record or configure a DuckDNS update client.

Check the DNS result from the VPS:

```bash
curl -4 -s https://icanhazip.com
getent ahostsv4 yourname.duckdns.org
```

The addresses should match. If they do not, fix the DuckDNS record before continuing.

Allow inbound TCP ports `80` and `443` in your VPS provider firewall. If UFW is enabled, check its status:

```bash
sudo ufw status
```

If needed, allow web traffic:

```bash
sudo ufw allow 'Nginx Full'
```

## 2. Install Docker

Check whether Docker and Compose are already installed:

```bash
docker --version
docker compose version
```

If either command is missing, install Docker Engine and the Compose plugin using the [official Docker Ubuntu instructions](https://docs.docker.com/engine/install/ubuntu/).

## 3. Deploy Vaultwarden

Create a directory for the deployment and persistent data:

```bash
mkdir -p ~/vaultwarden/data
cd ~/vaultwarden
```

Create the Docker Compose file:

```bash
nano compose.yaml
```

Paste the following configuration. Replace `yourname.duckdns.org` with your hostname:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: "https://yourname.duckdns.org"
      SIGNUPS_ALLOWED: "true"
    volumes:
      - ./data:/data
    ports:
      - "127.0.0.1:8000:80"
```

Save the file (`Ctrl+O`, Enter, then `Ctrl+X` in nano) and start the container:

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=50 vaultwarden
```

Check the local endpoint:

```bash
curl -I http://127.0.0.1:8000
```

An HTTP response means the service is reachable locally. If it fails, inspect the container logs before continuing.

## 4. Create the first account privately

Do this before opening Vaultwarden to the public internet.

On your own computer—not in the VPS terminal—open a terminal and create an SSH tunnel:

```bash
ssh -L 8000:127.0.0.1:8000 YOUR_SSH_USER@YOUR_VPS_IP
```

Replace `YOUR_SSH_USER` and `YOUR_VPS_IP` with your SSH username and VPS IP. Keep this SSH connection open.

Open this address in your computer's browser:

```text
http://127.0.0.1:8000
```

Create your account with a long, unique master password. The connection is forwarded through SSH; do not expose port `8000` publicly to create the account.

## 5. Disable public registration

After creating the account, return to the VPS and edit the Compose file:

```bash
cd ~/vaultwarden
nano compose.yaml
```

Change:

```yaml
SIGNUPS_ALLOWED: "true"
```

to:

```yaml
SIGNUPS_ALLOWED: "false"
```

Apply the setting:

```bash
docker compose up -d --force-recreate vaultwarden
```

Confirm the service is still running:

```bash
docker compose ps
curl -I http://127.0.0.1:8000
```

Keep registrations disabled. Do not publish the service while registrations are enabled.

## 6. Configure Nginx

Create a new Nginx site file. Do not overwrite any existing site configuration:

```bash
sudo nano /etc/nginx/sites-available/vaultwarden
```

Paste this configuration, using your DuckDNS hostname:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name yourname.duckdns.org;

    client_max_body_size 128M;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 300s;
    }
}
```

Save the file, then enable the new site:

```bash
sudo ln -s /etc/nginx/sites-available/vaultwarden /etc/nginx/sites-enabled/vaultwarden
```

If the symbolic link already exists, do not create it a second time.

Test the Nginx configuration:

```bash
sudo nginx -t
```

Only if the test succeeds, reload Nginx:

```bash
sudo systemctl reload nginx
```

Check the HTTP route locally:

```bash
curl -I -H 'Host: yourname.duckdns.org' http://127.0.0.1
```

An HTTP response from Vaultwarden indicates that Nginx is forwarding requests to the service. HTTPS is configured in the next step.

## 7. Enable HTTPS with Certbot

The hostname must resolve to the VPS, and the certificate authority must be able to reach the server over TCP port `80`. Confirm that ports `80` and `443` are reachable through your provider firewall.

Install Certbot and its Nginx plugin:

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

Request a certificate, replacing the example hostname:

```bash
sudo certbot --nginx -d yourname.duckdns.org
```

Follow the prompts. If asked, choose to redirect HTTP traffic to HTTPS.

Test and reload Nginx:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Open this URL:

```text
https://yourname.duckdns.org
```

Confirm your browser displays a valid HTTPS certificate before entering your master password.

## 8. Connect the browser extension and mobile app

1. Install the official Bitwarden browser extension or mobile app from its official store.
2. On the login screen, open the server or region selector.
3. Choose the self-hosted server option.
4. Enter your URL, for example `https://yourname.duckdns.org`.
5. Save the setting and sign in with the account created earlier.
6. Add a test login and verify that it syncs between your devices.

Vaultwarden is compatible with many Bitwarden clients, but because it is an unofficial implementation, not every official Bitwarden feature or client version is guaranteed to work.

## 9. Back up the vault

Vaultwarden's persistent data is stored in `~/vaultwarden/data`. Create a backup before updates and on a regular schedule. This example briefly stops the Vaultwarden container to make the archive consistent:

```bash
cd ~/vaultwarden
docker compose stop vaultwarden

tar -czf "$HOME/vaultwarden-backup-$(date +%F-%H%M).tar.gz" data compose.yaml

docker compose start vaultwarden
chmod 600 "$HOME"/vaultwarden-backup-*.tar.gz
```

Store an additional encrypted copy on another device or backup service. A backup kept only on the same VPS may be lost if the VPS or its disk fails. Periodically test restoration on a separate test instance.

## 10. Routine checks and updates

Check container status and logs:

```bash
cd ~/vaultwarden
docker compose ps
docker compose logs --tail=100 vaultwarden
```

Check Nginx and certificate status:

```bash
sudo nginx -t
sudo certbot certificates
sudo certbot renew --dry-run
```

Back up the data before updating. Then pull and recreate the container:

```bash
cd ~/vaultwarden
docker compose pull
docker compose up -d
```

Review release notes before updating and verify the vault still works afterwards. For more controlled deployments, consider pinning a specific image version instead of using `latest`.

## Troubleshooting

### The public IP with port 8000 does not open

This is expected. Docker binds Vaultwarden to `127.0.0.1:8000` so that it is not exposed directly to the internet. Use the HTTPS hostname after Nginx and Certbot are configured. Do not change the port mapping to `0.0.0.0:8000` merely to make it public.

### DuckDNS does not resolve to the VPS

```bash
curl -4 -s https://icanhazip.com
getent ahostsv4 yourname.duckdns.org
```

Make sure the addresses match. Update the DuckDNS record if necessary.

### Nginx returns `502 Bad Gateway`

Check the container and local endpoint:

```bash
cd ~/vaultwarden
docker compose ps
docker compose logs --tail=100 vaultwarden
curl -I http://127.0.0.1:8000
```

### Certbot fails

Common causes include an incorrect DNS record, TCP port `80` being blocked by the provider firewall, or an invalid Nginx configuration. Check:

```bash
sudo nginx -t
getent ahostsv4 yourname.duckdns.org
sudo ss -ltnp | grep -E ':(80|443)\b'
```

Do not delete other Nginx site configurations to fix a Vaultwarden issue. Resolve the reported error, test with `sudo nginx -t`, and only then reload Nginx.

## References

- Vaultwarden source and documentation: https://github.com/dani-garcia/vaultwarden
- Vaultwarden Nginx reverse-proxy examples: https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples
- Docker Engine installation for Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Certbot instructions: https://certbot.eff.org/instructions
- DuckDNS: https://www.duckdns.org/
