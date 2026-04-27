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

Stop the Server
To safely shut down the containers without losing your database volumes or rotated logs:
```bash
docker compose down
```
Warning: If you want to completely wipe the system and delete the Postgres database, add the -v flag (docker compose  down -v).

## Server Uptime Monitoring
To ensure you are alerted if the VPS drops or a container crashes, it is highly recommended to run Uptime Kuma independently of this compose file (so monitoring stays active even if you run down on the Prefect stack). For now the Uptime Kuma container will be in the same docker compose file (let's see what happens).
Run this command once on your VPS to start the monitoring dashboard:

```bash
docker run -d --restart=always -p 3001:3001 -v uptime_kuma_data:/app/data -v /var/run/docker.sock:/var/run/docker.sock:ro --name uptime-kuma louislam/uptime-kuma:1
```

Access the dashboard at http://<your-vps-ip>:3001, add a "Docker Container" monitor, and select your prefect-api or jobhunter-worker-local containers to receive instant notifications via Telegram or Discord if they go down.