---
description: "Deploy FastSvelte on Fly.io - Global container deployment with PostgreSQL and edge hosting."
keywords: "fly.io deployment, fly postgres, fly containers, fastsvelte fly"
---

# Deploy to Fly.io

Deploy FastSvelte on Fly.io for global edge deployment with low-latency container hosting.

## What You'll Use

- **Fly Machines** - Run your FastAPI backend container globally
- **Fly Postgres** - PostgreSQL database cluster
- **Fly Static Sites** - Host frontend (or use Vercel/Netlify)

## Cost Estimate

For a small production app:
- Machines (shared-cpu-1x): ~$3-5/month
- Postgres (single node): ~$2-5/month
- Static sites: Free tier available

**Total:** ~$5-10/month (very affordable!)

## Why Fly.io?

- **Global edge network** - Deploy close to your users
- **Developer-focused** - Great CLI and DX
- **Affordable** - Generous free tier
- **Fast** - Low latency worldwide
- **Flexible** - Full control over infrastructure

## Deployment Guide

**Coming soon!** This guide is under development.

In the meantime, Fly.io has excellent documentation:
- [Fly.io Documentation](https://fly.io/docs/)
- [Deploy Docker Image](https://fly.io/docs/languages-and-frameworks/dockerfile/)
- [Fly Postgres](https://fly.io/docs/postgres/)

## Quick Start

1. Install Fly CLI: `brew install flyctl`
2. Login: `fly auth login`
3. Deploy backend: `fly launch` in backend directory
4. Create Postgres: `fly postgres create`
5. Attach database: `fly postgres attach`
6. Deploy frontend to Vercel/Netlify or Fly static sites

## Key Features

- Global Anycast network
- Automatic HTTPS/SSL
- Zero-downtime deployments
- Built-in metrics and logging
- Easy horizontal scaling
- Multi-region support

---

Want to contribute this guide? [Open a PR](https://github.com/harunzafer/fastsvelte) with your deployment experience!
