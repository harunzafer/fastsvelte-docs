---
description: "Deploy FastSvelte to production - Simple guide to getting your SaaS application online with popular cloud platforms."
keywords: "fastsvelte deployment, deploy saas, vercel deployment, azure deployment, docker deployment"
---

# Deployment

Ready to take your FastSvelte app live? Great! Deploying is simpler than you might think.

## What You Need

FastSvelte needs three things to run in production:

1. **A place for your backend** - Any platform that can run Docker containers (Azure, AWS, DigitalOcean, etc.)
2. **A PostgreSQL database** - Many providers offer this (Neon, Supabase, Azure, AWS, etc.)
3. **Hosting for your frontend** - Vercel, Netlify, Cloudflare Pages, or similar

That's it! Most modern cloud platforms make this straightforward with just a few setup steps.

## Choose Your Platform

FastSvelte's flexible architecture means you're not locked into any specific platform—you can run it anywhere that supports Docker containers, PostgreSQL, and static file hosting. Below we've included guides for the most popular, affordable, and production-ready options.

**Full-stack platforms (backend + database + frontend):**

- **[Azure](azure.md)** - Container Apps + PostgreSQL Flexible Server + Static Web Apps
- **[DigitalOcean](digitalocean.md)** - App Platform + Managed Database + Static Sites
- **[Railway](railway.md)** - Containers + PostgreSQL + Static Sites (simple, affordable)
- **[Fly.io](fly.md)** - Containers + Postgres + Static Sites
- **[AWS](aws.md)** - ECS/Fargate + RDS + S3/CloudFront
- **[Google Cloud](gcp.md)** - Cloud Run + Cloud SQL + Cloud Storage

**Frontend-only platforms (pair with above for backend):**

- **[Vercel](vercel.md)** - Fast frontend deployment
- **[Netlify](netlify.md)** - Frontend with great DX
- **[Cloudflare Pages](cloudflare.md)** - Global edge deployment

**Affordable PostgreSQL databases (pair with any platform):**

- **[Neon](neon.md)** - Serverless Postgres with free tier
- **[Supabase](supabase.md)** - Postgres + extras, generous free tier
- **[Railway](railway.md)** - Simple Postgres hosting
- **[Render](render.md)** - Managed Postgres from $7/month

**Self-hosted platforms (run on your own server):**

- **[Coolify](coolify.md)** - Open-source PaaS (like your own Heroku)
- **[Docker Compose](docker-compose.md)** - Manual container orchestration

## Quick Overview

Here's what happens when you deploy:

```
Your Users → Frontend (Vercel/Netlify) → Backend API (Container) → Database (PostgreSQL)
```

The frontend is just HTML/CSS/JS files hosted anywhere. It talks to your backend API, which connects to your database.

## Security Checklist

Before going live:

- [ ] Use strong passwords for database and admin accounts
- [ ] Set proper CORS origins (your actual domain, not `*`)
- [ ] Use HTTPS everywhere (usually automatic with modern platforms)
- [ ] Don't commit secrets to git - use environment variables
- [ ] Enable database backups

Platform guides include specific security recommendations.

## Alternative: Run on Your Own Server

You can also deploy FastSvelte on a traditional VPS or bare metal server using Docker Compose or directly with Python/Node.js, though this requires DevOps knowledge. FastSvelte includes a `docker-compose.yml` you can use as a starting point.

**Note:** We recommend cloud platforms for most users - they handle scaling, backups, and security updates automatically.
