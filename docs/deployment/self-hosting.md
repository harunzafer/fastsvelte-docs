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

## Why Docker Compose?

- **Full control** - Your server, your rules
- **Cost-effective** - Cheapest option for single deployment
- **Simple** - One configuration file
- **Portable** - Works on any Docker host
- **No vendor lock-in** - Standard Docker setup

## Prerequisites

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh

# Install Docker Compose
sudo apt-get update
sudo apt-get install docker-compose-plugin
```

## FastSvelte Includes docker-compose.yml

FastSvelte repository includes a `docker-compose.yml` file as a starting point for deployment.

```yaml
# Example structure (simplified)
services:
  db:
    image: postgres:16
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: fastsvelte
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}

  api:
    build: ./backend
    depends_on:
      - db
    environment:
      FS_DB_URL: postgres://postgres:${DB_PASSWORD}@db:5432/fastsvelte
      FS_JWT_SECRET_KEY: ${JWT_SECRET}
    ports:
      - "8000:3100"

  frontend:
    build: ./frontend
    environment:
      PUBLIC_API_BASE_URL: https://api.yourdomain.com
    ports:
      - "80:80"
      - "443:443"
```

## Deployment Steps

### 1. Clone Your Repository

```bash
git clone <your-fastsvelte-repo>
cd fastsvelte
```

### 2. Configure Environment

```bash
# Create .env file
cat > .env << EOF
DB_PASSWORD=your_secure_password
JWT_SECRET=your_jwt_secret_key
STRIPE_API_KEY=your_stripe_key
EOF
```

### 3. Deploy

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f
```

### 4. Run Migrations

```bash
# Access backend container
docker-compose exec api bash

# Run migrations
cd /app
# Your migration commands here
```

## Setting Up Nginx Reverse Proxy

For production, use Nginx as a reverse proxy:

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

server {
    server_name app.yourdomain.com;

    location / {
        proxy_pass http://localhost:80;
        proxy_set_header Host $host;
    }
}
```

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

    location / {
        proxy_pass http://localhost:80;
        proxy_set_header Host $host;
    }
}
```

Replace `https://api.yourdomain.com` in `connect-src` with your real API URL, or the browser will block the app from calling it. See [Security](../features/security.md#frontend-add-at-your-host).

## SSL with Let's Encrypt

```bash
# Install Certbot
sudo apt-get install certbot python3-certbot-nginx

# Get certificates
sudo certbot --nginx -d api.yourdomain.com -d app.yourdomain.com
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

# Rebuild and restart
docker-compose down
docker-compose up -d --build
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
