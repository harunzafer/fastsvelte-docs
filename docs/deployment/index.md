---
description: "Deploy FastSvelte — API at api.yourdomain.com, app at app.yourdomain.com, landing at yourdomain.com, plus PostgreSQL. The most common, straightforward provider setups."
keywords: "fastsvelte deployment, deploy fastapi sveltekit, railway, vercel, digitalocean, fly.io, render, docker, postgres hosting, self-hosting"
---

# Deployment

FastSvelte deploys as three pieces plus a database. A typical production setup maps them to three URLs:

| Piece | What it is | Typical URL |
|-------|-----------|-------------|
| **API** | FastAPI backend (Docker container) | `api.yourdomain.com` |
| **App** | SvelteKit SPA dashboard (static files) | `app.yourdomain.com` |
| **Landing** | SvelteKit marketing site (static files) | `yourdomain.com` |
| **Database** | PostgreSQL | managed or self-hosted |

Anything that can run a Docker container, serve static files, and provide PostgreSQL works — so you can **mix any providers you like**. Below are the most common, straightforward setups. Pick one and you're live.

## Common setups

Five complete, end-to-end guides — pick one:

- **[Railway](railway.md)** — all-in-one (API + app + landing + Postgres). The simplest, near one-click.
- **[DigitalOcean](digitalocean.md)** — App Platform: container + managed Postgres + static sites.
- **[Fly.io + Neon + Vercel](fly-neon-vercel.md)** — best-of-breed: API on Fly, Postgres on Neon, app + landing on Vercel.
- **[Azure](azure.md)** — Container Apps + PostgreSQL Flexible Server + Static Web Apps (enterprise).
- **[Self-Hosting (Docker Compose)](self-hosting.md)** — all three + Postgres on one VPS.

They reach the same outcome — point each provider at the right subdomain and set the URLs/CORS so the app can reach the API ([Configuration](../reference/configuration.md)).

## Every setup needs

- **Environment variables** — backend `FS_*` (see [Configuration](../reference/configuration.md)); the frontend and landing use `PUBLIC_*` (e.g. `PUBLIC_API_BASE_URL`).
- **A PostgreSQL database** — put its connection string in `FS_DB_URL`, then run migrations (`./sqitch.sh <env> deploy`).
- **The Stripe webhook** — point it at `https://api.yourdomain.com/webhooks/stripe` (see [Billing & Subscriptions](../features/billing.md)).
- **HTTPS** — most platforms provision SSL automatically; when self-hosting, terminate TLS at your reverse proxy.
- **A production pass** — review the [Security](../features/security.md) checklist before launch.
