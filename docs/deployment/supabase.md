---
description: "Use Supabase PostgreSQL with FastSvelte - Open-source Firebase alternative with generous free tier."
keywords: "supabase postgres, supabase database, fastsvelte supabase"
---

# Supabase PostgreSQL

Use Supabase as your PostgreSQL database for FastSvelte - an open-source Firebase alternative with Postgres, auth, and more.

## What is Supabase?

Supabase is more than just a database - it's a full Backend-as-a-Service platform built on PostgreSQL:
- **PostgreSQL database** - Full Postgres with extensions
- **Authentication** - Built-in auth (can replace FastSvelte auth if desired)
- **Storage** - File storage built-in
- **Realtime** - Database change subscriptions
- **Edge Functions** - Serverless functions

## Cost Estimate

- **Free tier:** 500MB database, 2GB storage, 50,000 monthly active users
- **Pro tier:** $25/month + usage (~$25-40/month typical)
- **Comparable to Neon** but with more features

## Why Supabase?

- **Generous free tier** - Great for side projects
- **All-in-one** - Database + Auth + Storage
- **Open source** - Can self-host if needed
- **Great DX** - Beautiful dashboard and docs
- **PostgreSQL** - Full Postgres compatibility

## Setup Guide

### 1. Create Supabase Project

1. Go to [Supabase Dashboard](https://supabase.com/dashboard)
2. Sign up and create new project
3. Choose region closest to your users
4. Wait for database provisioning (~2 minutes)

### 2. Get Connection String

In Supabase Dashboard:
1. Go to Project Settings → Database
2. Copy the connection string (Direct connection)
3. Look for: `postgres://postgres.[project-ref]:[password]@aws-0-[region].pooler.supabase.com:6543/postgres`

### 3. Configure FastSvelte

Add to your backend `.env`:

```bash
DATABASE_URL_PROD="postgres://postgres.[project-ref]:[password]@aws-0-[region].pooler.supabase.com:6543/postgres"
```

### 4. Run Migrations

```bash
cd db
./sqitch.sh prod deploy
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

## Connection Pooling

Supabase includes Supavisor (connection pooler) built-in. Use the pooled connection string for better performance with serverless platforms.

## Optional: Use Supabase Auth

If you want to replace FastSvelte's built-in auth with Supabase Auth:
- Supabase provides JWT-based authentication
- Social logins (Google, GitHub, etc.)
- Email/password auth
- Magic links

**Note:** This requires code changes to integrate Supabase Auth SDK.

## Optional: Use Supabase Storage

Replace file storage implementation with Supabase Storage:
- Built-in file uploads
- Image transformations
- CDN delivery

## Recommended Platform Combinations

**Budget-friendly:**
```
Vercel (frontend) + Railway (backend) + Supabase (database + storage)
Total: ~$0-20 (free tiers) or ~$45-65/month (paid)
```

**With Azure:**
```
Azure Container Apps (backend) + Azure Static Web Apps (frontend) + Supabase (database)
Total: ~$50-70/month
```

## Key Features

- Full PostgreSQL compatibility
- Connection pooling (Supavisor)
- Real-time subscriptions
- Row-level security
- Automatic backups
- Point-in-time recovery
- Database branching
- Built-in authentication
- File storage

## Learn More

- [Supabase Documentation](https://supabase.com/docs)
- [Supabase Pricing](https://supabase.com/pricing)
- [Supabase with Python](https://supabase.com/docs/reference/python/introduction)

---

Supabase is a great choice if you want an all-in-one backend platform with generous free tier!
