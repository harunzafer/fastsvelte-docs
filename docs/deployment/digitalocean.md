---
description: "Deploy FastSvelte on DigitalOcean - App Platform for containers, Managed Databases for PostgreSQL, and simple static site hosting."
keywords: "digitalocean deployment, digitalocean app platform, digitalocean managed database, fastsvelte digitalocean"
---

# Deploy to DigitalOcean

Deploy FastSvelte using DigitalOcean's App Platform and Managed Databases. Known for simplicity and developer-friendly pricing.

## What You'll Use

- **App Platform** - Deploy your FastAPI backend container
- **Managed Database (PostgreSQL)** - Hosted PostgreSQL database
- **App Platform Static Sites** - Host your SvelteKit frontend and landing

## Cost Estimate

For a small production app:
- App Platform (Basic): $5-12/month
- Managed Database (PostgreSQL): $15/month
- Static Sites: Free

**Total:** ~$20-27/month

## Why DigitalOcean?

- Simple, predictable pricing
- Great documentation
- Fast deployment
- Good performance for the price
- Excellent for small to medium apps

## Deployment Guide

**Coming soon!** This guide is under development.

In the meantime, DigitalOcean has excellent documentation:
- [App Platform Overview](https://docs.digitalocean.com/products/app-platform/)
- [Managed Databases](https://docs.digitalocean.com/products/databases/)

## What You'll Need

1. **Backend:** Deploy Docker container via App Platform (auto-builds from Git)
2. **Database:** Create PostgreSQL Managed Database (1GB RAM minimum)
3. **Frontend:** Deploy static sites via App Platform (auto-builds from Git)

Connect your GitHub repository and DigitalOcean handles the rest!

## Key Features

- Automatic HTTPS/SSL
- Built-in CI/CD from GitHub
- Easy environment variable management
- Database connection pooling included
- Automatic database backups
- Simple scaling options

---

Want to contribute this guide? [Open a PR](https://github.com/harunzafer/fastsvelte) with your deployment experience!
