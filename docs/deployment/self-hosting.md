---
description: "Deploy FastSvelte with Docker Compose - Self-hosted deployment on your own server or VPS."
keywords: "docker compose deployment, self-hosted fastsvelte, vps deployment"
---

# Deploy with Docker Compose

Deploy FastSvelte on your own server using Docker Compose for full control over your infrastructure.

## What You'll Need

- A Linux server (Ubuntu 22.04+ recommended)
- Docker and Docker Compose installed
- A domain name (optional but recommended)
- Basic Linux and DevOps knowledge

## Cost Estimate

- VPS (2GB RAM): $5-12/month
- Domain: $10-15/year
- **Total:** ~$5-15/month

??? note "Why Docker Compose?"

    - **Full control**: your server, your rules
    - **Cost-effective**: the cheapest option for a single deployment
    - **Simple**: one configuration file
    - **Portable**: works on any Docker host
    - **No vendor lock-in**: standard Docker setup

## Prerequisites

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh

# Install Docker Compose
sudo apt-get update
sudo apt-get install docker-compose-plugin
```

## The Backend Runs in Docker Compose

The kit ships `backend/docker-compose.yml`, which runs the two server pieces: PostgreSQL and the FastAPI backend.

```yaml
# backend/docker-compose.yml (shipped with the kit, simplified)
services:
  db:
    image: postgres:17
    volumes:
      - db-data:/var/lib/postgresql/data

  api:
    build:
      context: .
    depends_on:
      - db
    ports:
      - "8000:3100"
    env_file:
      - .env
```

The frontend and landing are **not** compose services. They are static builds (see [Architecture](../reference/architecture.md#rendering-model-app-and-landing)): the app is an SPA with an `index.html` fallback, the landing is prerendered HTML. [Deploy the Frontends](#deploy-the-frontends-static-files) below covers serving them.

## Deployment Steps

### 1. Clone Your Repository

```bash
git clone <your-fastsvelte-repo>
cd fastsvelte
```

### 2. Configure Environment

```bash
cd backend
cp .env.example .env
# Fill in the secrets: database credentials, session secret, Stripe keys, ...
```

### 3. Deploy

```bash
# Start db + api (from backend/)
docker compose up -d

# View logs
docker compose logs -f
```

### 4. Run Migrations

```bash
# Access backend container
docker-compose exec api bash

# Run migrations
cd /app
# Your migration commands here
```

## Deploy the Frontends (Static Files)

`frontend/` and `landing/` build to plain static files, and `PUBLIC_*` variables are baked in at build time, so set them before `npm run build`. There are two ways to serve the output; Option A is simpler.

### Option A: nginx serves the builds (recommended)

Build on the server (Node 24+) or in CI, then copy the output into place:

```bash
sudo mkdir -p /var/www/fastsvelte/app /var/www/fastsvelte/landing

cd frontend
npm ci
PUBLIC_API_BASE_URL=https://api.yourdomain.com npm run build
sudo cp -r build/. /var/www/fastsvelte/app/

cd ../landing
npm ci
npm run build
sudo cp -r build/. /var/www/fastsvelte/landing/
```

Rebuild and re-copy whenever the code or any `PUBLIC_*` value changes. The landing is prerendered, so marketing content edits also need a rebuild.

### Option B: Frontends in Docker

Prefer everything containerized? Create `frontend/Dockerfile` (the kit does not ship one):

```dockerfile
FROM node:24-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG PUBLIC_API_BASE_URL
ENV PUBLIC_API_BASE_URL=$PUBLIC_API_BASE_URL
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY <<'EOF' /etc/nginx/conf.d/default.conf
server {
    listen 80;
    root /usr/share/nginx/html;
    try_files $uri /index.html;
}
EOF
```

`PUBLIC_API_BASE_URL` must be a **build arg** (under `build.args` in your compose service), not a runtime variable: a static build cannot read the environment after it is built. Repeat for `landing/`, changing the nginx line to `try_files $uri $uri.html =404;` (the landing is prerendered HTML, not an SPA). With this option, the nginx blocks below `proxy_pass` to the containers instead of serving files.

## Setting Up Nginx

nginx fronts everything: it proxies the API container and, with Option A, serves the static builds directly:

```nginx
# /etc/nginx/sites-available/fastsvelte
server {
    server_name api.yourdomain.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# App (static SPA): serve files, fall back to index.html for deep links
server {
    server_name app.yourdomain.com;
    root /var/www/fastsvelte/app;
    try_files $uri /index.html;
}

# Landing (prerendered): serve files, map /about to about.html
server {
    server_name yourdomain.com;
    root /var/www/fastsvelte/landing;
    try_files $uri $uri.html =404;
}
```

With Option B, replace the two static blocks with `proxy_pass` blocks pointing at the frontend and landing containers' published ports.

### Security headers

In the `app.yourdomain.com` server block above, add the app's security headers:

```nginx
server {
    server_name app.yourdomain.com;

    add_header Content-Security-Policy "default-src 'self'; connect-src 'self' https://api.yourdomain.com; img-src 'self' data:; style-src 'self' 'unsafe-inline'; script-src 'self'; font-src 'self'; object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none'" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

    root /var/www/fastsvelte/app;
    try_files $uri /index.html;
}
```

Replace `https://api.yourdomain.com` in `connect-src` with your real API URL, or the browser will block the app from calling it. See [Security](../features/security.md#frontend-add-at-your-host).

### Serving the app from a sub-path

Sub-path serving is simplest with Option B, where the app container keeps serving the build at its root. To serve the app at `yourdomain.com/app` instead of its own subdomain, drop the `app.yourdomain.com` server block and add a location to the main site's block that proxies to the app container:

```nginx
location /app/ {
    proxy_pass http://localhost:80/;  # trailing slash strips the /app prefix
    proxy_set_header Host $host;
}
```

The container keeps serving the build at its root with its own `index.html` fallback; the stripped prefix makes that work unchanged. The frontend and backend settings that go with this are in [Serving from a Sub-Path](sub-path.md).

## SSL with Let's Encrypt

```bash
# Install Certbot
sudo apt-get install certbot python3-certbot-nginx

# Get certificates
sudo certbot --nginx -d api.yourdomain.com -d app.yourdomain.com -d yourdomain.com
```

## Maintenance

### Backups

```bash
# Backup database
docker-compose exec db pg_dump -U postgres fastsvelte > backup.sql

# Backup volumes
docker run --rm -v fastsvelte_postgres_data:/data -v $(pwd):/backup ubuntu tar czf /backup/postgres_backup.tar.gz /data
```

### Updates

```bash
# Pull latest changes
git pull

# Rebuild and restart the backend
(cd backend && docker compose up -d --build)

# Rebuild and redeploy the frontends (Option A)
(cd frontend && npm run build) && sudo cp -r frontend/build/. /var/www/fastsvelte/app/
(cd landing && npm run build) && sudo cp -r landing/build/. /var/www/fastsvelte/landing/
```

### Monitoring

```bash
# View logs
docker-compose logs -f

# Check container status
docker-compose ps

# View resource usage
docker stats
```

## Recommended VPS Providers

- **Hetzner** - €4-20/month, excellent EU performance
- **DigitalOcean** - $6-20/month, reliable and well-documented
- **Vultr** - $6-20/month, global locations
- **Linode** - $5-20/month, good performance

## Alternative: Use Coolify

If you want a GUI for managing Docker deployments, consider [Coolify](https://coolify.io), a self-hostable web interface for Docker Compose deployments.

## Security Considerations

- Keep Docker and system packages updated
- Use strong passwords and secrets
- Configure firewall (ufw or iptables)
- Set up automated backups
- Monitor logs for suspicious activity
- Use fail2ban for SSH protection

---

Docker Compose is best for: single deployments, learning, development servers, or when you want maximum control at minimum cost.
