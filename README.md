# Prefect Orchestration Server

Docker Compose configuration for a Prefect server. It includes resource limits, persistent log rotation, and automated database migrations to ensure high availability on a VPS

## Services Included

- **Postgres (14)**: The core database for Prefect.
- **Redis (7)**: Used for Prefect's messaging broker and cache.
- **Migrate**: A short-lived container that ensures the Postgres database is up to date before the API starts.
- **Prefect API**: The main Prefect 3 server and web dashboard (exposed on port `4200`).
- **Prefect Background**: Handles scheduled runs and background tasks for the Prefect server.
- **Local Worker**: Custom worker that polls the Prefect server and executes the actual flows.

## Prerequisites

Ensure you have **Docker Compose V2** installed.
Verify by running:
```bash
docker compose version
```

## Deployment Instructions
Start the Server
To start the entire orchestration stack in the background, run:

```bash
docker compose up -d
```

### Check Logs
Logs are configured to securely rotate (max 3 files, 10MB each) so they won't fill up your disk.
To view the live logs of all/specific services:

```bash
docker compose logs -f 
# or
docker compose logs -f prefect-api
```

To view persistent log files on the VPS:
```bash
ls -la ./logs/
cat ./logs/prefect-api/*.log
tail -f ./logs/prefect-api/*.log
```

Stop the Server
To safely shut down the containers while preserving your database volumes and logs:
```bash
docker compose down
```

**Log Persistence**: 
Logs are mounted to the `./logs/` directory on your VPS host. With this bind mount configuration, logs persist even after `docker compose down` removes the containers. You can review and archive historical logs from the VPS filesystem as needed.

Warning: If you want to completely wipe the system and delete the Postgres database, add the -v flag (docker compose down -v). Note: This does NOT delete the `./logs/` directory—logs will still be preserved on the host.

## Server Uptime Monitoring
To ensure you are alerted if the VPS drops or a container crashes, it is highly recommended to run Uptime Kuma independently of this compose file (so monitoring stays active even if you run down on the Prefect stack). For now the Uptime Kuma container will be in the same docker compose file (let's see what happens).
Run this command once on your VPS to start the monitoring dashboard:

```bash
docker run -d --restart=always -p 3001:3001 -v uptime_kuma_data:/app/data -v /var/run/docker.sock:/var/run/docker.sock:ro --name uptime-kuma louislam/uptime-kuma:1
```

Access the dashboard at http://<your-vps-ip>:3001, add a "Docker Container" monitor, and select your prefect-api or jobhunter-worker-local containers to receive instant notifications via Telegram or Discord if they go down.

## Reverse Proxy Configuration

When hosting the Prefect UI behind a reverse proxy (Nginx, Traefik, etc.), you must configure the UI to connect to the API using the external proxy URL. This ensures the browser can reach the API.

### Option B: DuckDNS + Nginx + Let's Encrypt (Free SSL)

This guide sets up HTTPS for free using DuckDNS (free domain) and Let's Encrypt (free SSL certificate).

**Step 1: Set up DuckDNS**

1. Go to https://www.duckdns.org
2. Sign in with GitHub/Google
3. Create a subdomain (e.g., `myserver.duckdns.org`)
4. Set the IP address to your VPS IP
5. Note your DuckDNS token (you'll need it later)

**Step 2: Install Nginx and Certbot on your VPS**

```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx -y
```

**Step 3: Create Nginx Configuration**

Create file `/etc/nginx/sites-available/prefect`:

```bash
sudo nano /etc/nginx/sites-available/prefect
```

Paste this config (replace `myserver.duckdns.org` with YOUR DuckDNS domain):

```nginx
server {
    listen 80;
    server_name myserver.duckdns.org;

    location / {
        proxy_pass http://localhost:4200;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 600s;
        proxy_connect_timeout 600s;
    }
}
```

**Step 4: Enable the Nginx Site**

```bash
sudo ln -s /etc/nginx/sites-available/prefect /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default  # Remove default site
sudo nginx -t  # Test configuration
sudo systemctl restart nginx
```

**Step 5: Get Free SSL Certificate from Let's Encrypt**

```bash
sudo certbot --nginx -d myserver.duckdns.org
```

When prompted:
- Enter your email
- Agree to terms
- Choose to redirect HTTP to HTTPS when asked

This automatically renews every 90 days.

**Step 6: Update Your `.env` File**

Add your DuckDNS domain to your Prefect server config:

```bash
# In /path/to/orchestrator-server/.env
PREFECT_UI_API_URL=https://myserver.duckdns.org/api
```

**Step 7: Restart Prefect**

```bash
cd /path/to/orchestrator-server
docker compose down
docker compose up -d
```

**Step 8: Access Your Prefect Server**

- UI: https://myserver.duckdns.org
- Login with credentials from `.env` (e.g., `admin:pass`)

**Keep DuckDNS IP Updated (Optional)**

If your VPS IP changes, update it at https://www.duckdns.org or use a cron job:

```bash
# Add to crontab: crontab -e
*/5 * * * * curl "https://www.duckdns.org/update?domains=myserver&token=YOUR_TOKEN&ip=" >> /tmp/duckdns.log 2>&1
```

**Verify HTTPS is Working**

```bash
curl -I https://myserver.duckdns.org
# Should show: HTTP/2 401 (401 is expected - auth required)
```

### Additional Security Notes

- ✅ Traffic is now encrypted (HTTPS)
- ✅ SSL certificate auto-renews every 90 days
- ✅ Authentication protects against unauthorized access
- ⚠️ Keep your `.env` file secure (contains credentials)
- ⚠️ Run `docker compose` commands with appropriate permissions

Configure in your `.env` file:
```
PREFECT_UI_API_URL=https://prefect-server.example.com/api
```

Replace `prefect-server.example.com` with your domain or VPS IP address. The default is `http://localhost:4200/api` for local development.

**Example Nginx configuration:**
```nginx
server {
    listen 443 ssl;
    server_name prefect-server.example.com;

    ssl_certificate /etc/letsencrypt/live/prefect-server.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/prefect-server.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:4200;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

This ensures that API calls from the browser are sent to the correct external URL instead of attempting to reach `localhost:4200` from the client machine.