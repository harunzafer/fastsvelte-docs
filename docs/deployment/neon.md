---
description: "Use Neon serverless PostgreSQL with FastSvelte - Affordable, scalable database with generous free tier."
keywords: "neon postgres, neon database, serverless postgres, fastsvelte neon"
---

# Neon PostgreSQL

Use Neon as your PostgreSQL database for FastSvelte - a modern serverless Postgres platform with autoscaling and branching.

## What is Neon?

Neon is a serverless PostgreSQL platform that separates storage and compute, offering:
- **Autoscaling** - Scale compute up and down automatically
- **Branching** - Instant database branches for development
- **Serverless** - Pay only for what you use
- **Fast** - Low-latency connections

## Cost Estimate

- **Free tier:** 0.5GB storage, always available
- **Pro tier:** $19/month + usage (~$20-30/month typical)
- **Much cheaper than:** Azure PostgreSQL (~$50-100/month)

## Why Neon?

- **Great free tier** - Perfect for side projects
- **Affordable** - 50-70% cheaper than traditional managed databases
- **Modern** - Git-like branching for databases
- **Fast provisioning** - Database ready in seconds
- **Great for FastSvelte** - Works perfectly with container platforms

## Setup Guide

### 1. Create Neon Account

1. Go to [Neon Console](https://console.neon.tech/)
2. Sign up with GitHub
3. Create a new project

### 2. Get Connection String

In Neon Console:
1. Go to your project dashboard
2. Copy the connection string
3. It looks like: `postgres://user:pass@ep-xxx.region.aws.neon.tech/dbname?sslmode=require`

### 3. Configure FastSvelte

Add to your backend `.env` file:

```bash
# For development
DATABASE_URL_DEV="postgres://user:pass@ep-xxx.region.aws.neon.tech/dbname?sslmode=require"

# For production
DATABASE_URL_PROD="postgres://user:pass@ep-xxx.region.aws.neon.tech/dbname?sslmode=require"
```

### 4. Run Migrations

```bash
cd db

# Deploy migrations
./sqitch.sh prod deploy

# Verify
./sqitch.sh prod status
```

### 5. Create Admin User

```bash
cd ../backend
uv run scripts/create_admin.py \
  --env prod \
  --password <password> \
  --domain yourdomain.com
```

## Database Branching

Neon's killer feature - create instant database copies:

```bash
# Create a branch for testing
neon branches create --name testing

# Get connection string for branch
neon connection-string testing
```

Use branches for:
- Testing migrations before production
- Development environments
- Feature branch databases

## Connection Pooling

Neon includes connection pooling built-in. Use the pooled connection string for better performance:

```
postgres://user:pass@ep-xxx.region.aws.neon.tech/dbname?sslmode=require&pgbouncer=true
```

## Recommended Platform Combinations

**With Vercel:**
```
Vercel (frontend) + Railway (backend) + Neon (database)
Total: ~$0-20 (free tiers) or ~$30-50/month (paid)
```

**With Azure:**
```
Azure Container Apps (backend) + Azure Static Web Apps (frontend) + Neon (database)
Total: ~$35-50/month (much cheaper than Azure PostgreSQL)
```

**With DigitalOcean:**
```
DigitalOcean App Platform (backend + frontend) + Neon (database)
Total: ~$25-35/month
```

## Key Features

- Autoscaling compute
- Instant database branches
- Point-in-time restore
- Connection pooling (PgBouncer)
- Read replicas
- Automatic backups
- Monitoring and insights

## Learn More

- [Neon Documentation](https://neon.tech/docs)
- [Neon Pricing](https://neon.tech/pricing)
- [Neon GitHub Integration](https://neon.tech/docs/guides/vercel-integration)

---

Neon is highly recommended for FastSvelte deployments when you want to save on database costs without sacrificing features!
