---
description: "Deploy FastSvelte with Coolify - Open-source self-hosted PaaS alternative to Heroku and Vercel."
keywords: "coolify deployment, self-hosted paas, coolify docker, fastsvelte coolify"
---

# Deploy with Coolify

Deploy FastSvelte using Coolify - an open-source, self-hosted Platform-as-a-Service that runs on your own server.

## What is Coolify?

Coolify is like running your own Heroku or Netlify:
- **Self-hosted** - Runs on your VPS/server
- **Open source** - Free and transparent
- **All-in-one** - Deploy apps, databases, and static sites
- **280+ services** - One-click PostgreSQL, Redis, etc.

## Cost Estimate

- Coolify software: **Free** (open source)
- VPS/server: $5-20/month (DigitalOcean, Hetzner, etc.)
- **Total:** ~$5-20/month for everything

## Why Coolify?

- **Cost-effective** - Cheapest option for multiple projects
- **Full control** - Your infrastructure, your rules
- **Privacy** - Data stays on your servers
- **No vendor lock-in** - It's your hardware
- **Great for agencies** - Host many client projects on one server

## Requirements

- A Linux server (Ubuntu 22.04+ recommended)
- 2GB RAM minimum (4GB recommended)
- SSH access to your server
- A domain name

## Installation

```bash
# Install Coolify on your server
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```

Access Coolify at `http://your-server-ip:8000`

## Setup FastSvelte

1. **Add your server** in Coolify dashboard
2. **Create project** for FastSvelte
3. **Deploy backend:**
   - Add "Docker Compose" service
   - Connect GitHub repository
   - Coolify auto-detects Dockerfile
4. **Create PostgreSQL database:**
   - One-click PostgreSQL from service catalog
5. **Deploy frontend:**
   - Add "Static Site" service
   - Point to your frontend directory
   - Configure build command: `npm run build`

## Features

- Automatic HTTPS with Let's Encrypt
- GitHub/GitLab integration
- Built-in database backups
- One-click services (Postgres, Redis, etc.)
- Resource monitoring
- Multiple projects on one server

## Security Note

Coolify had security vulnerabilities disclosed in early 2026 that have been patched. Always run the latest version and keep your installation updated.

## Recommended VPS Providers

- **Hetzner** - €4-20/month, great EU performance
- **DigitalOcean** - $6-20/month, reliable and simple
- **Vultr** - $6-20/month, global locations
- **Linode** - $5-20/month, good performance

## Learn More

- [Coolify Documentation](https://coolify.io/docs)
- [Coolify GitHub](https://github.com/coollabsio/coolify)
- [Coolify Community](https://coolify.io/community)

---

Coolify is perfect if you want to self-host and have full control over your infrastructure at the lowest cost!
