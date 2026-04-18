---
description: "Deploy FastSvelte frontend and landing page on Vercel - Fast, simple deployment for SvelteKit applications."
keywords: "vercel deployment, vercel sveltekit, fastsvelte vercel, vercel frontend"
---

# Deploy to Vercel

Deploy your FastSvelte frontend and landing page on Vercel for blazing-fast static site hosting with excellent developer experience.

## What You'll Use

- **Vercel** - Frontend and landing page hosting
- **Backend elsewhere** - Use Azure, Railway, Fly.io, or DigitalOcean for backend
- **Database elsewhere** - Use Neon, Supabase, or another provider

**Note:** Vercel is perfect for the frontend, but you'll need separate hosting for your FastAPI backend container.

## Cost Estimate

- Vercel Hobby (personal): **Free**
- Vercel Pro (production): ~$20/month
- Plus backend hosting: $10-30/month
- Plus database: $10-20/month

**Total:** ~$40-70/month (Free + $20-50 for backend/DB)

## Why Vercel?

- **Best DX** - Deploy with `git push`
- **Fast** - Global edge network
- **Preview deployments** - Every PR gets a URL
- **Zero config** - Works with SvelteKit out of the box
- **Great free tier** - Perfect for side projects

## Deployment Guide

### 1. Deploy Frontend

```bash
cd frontend

# Install Vercel CLI
npm i -g vercel

# Login
vercel login

# Deploy
vercel

# Deploy to production
vercel --prod
```

Or connect your GitHub repository via [Vercel Dashboard](https://vercel.com/new) for automatic deployments.

### 2. Configure Environment Variables

In Vercel Dashboard → Settings → Environment Variables:

```
PUBLIC_API_BASE_URL=https://your-backend-url.com
PUBLIC_APP_NAME=Your App Name
```

### 3. Deploy Landing Page

Repeat the process for the landing directory:

```bash
cd landing
vercel --prod
```

### 4. Deploy Backend Elsewhere

Choose a platform for your backend:
- **[Railway](railway.md)** - Simplest, ~$10/month
- **[Fly.io](fly.md)** - Global edge, ~$5/month
- **[Azure Container Apps](azure.md)** - Enterprise-ready, ~$15/month
- **[DigitalOcean](digitalocean.md)** - Simple, ~$10/month

### 5. Set Up Database

Choose a database provider:
- **[Neon](neon.md)** - Serverless Postgres, free tier available
- **[Supabase](supabase.md)** - Postgres + extras, free tier available
- **[Railway](railway.md)** - Simple Postgres, ~$5/month

## Custom Domains

Add custom domains in Vercel Dashboard:
1. Go to your project → Settings → Domains
2. Add your domain (e.g., `app.yourdomain.com`)
3. Update DNS records as instructed

Vercel automatically provisions SSL certificates.

## Automatic Deployments

When connected to GitHub:
- Push to `main` → Production deployment
- Open PR → Preview deployment
- Every commit gets a unique URL

## Key Features

- Global edge network (CDN)
- Automatic HTTPS/SSL
- Preview deployments for every PR
- Built-in analytics
- Web Vitals monitoring
- Zero configuration

## Example Architecture

```
Users → Vercel (Frontend) → Railway/Fly.io (Backend API) → Neon (Database)
     ↓
     Vercel (Landing Page)
```

---

**Recommended combination:** Vercel (frontend) + Railway (backend + DB) = ~$15-25/month total
