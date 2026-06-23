---
description: "Deploy FastSvelte on DigitalOcean App Platform — FastAPI container, managed PostgreSQL, and the SvelteKit app + landing as static sites, with custom domains."
keywords: "deploy fastsvelte digitalocean, app platform fastapi, do managed postgres, digitalocean sveltekit"
---

# Deploy to DigitalOcean

DigitalOcean **App Platform** runs all three pieces in one app — the FastAPI container, a managed PostgreSQL, and the static app + landing. (Prefer a single server you control? See [Self-Hosting](self-hosting.md) for a droplet + Docker Compose.)

**Outcome:** API at `api.yourdomain.com`, app at `app.yourdomain.com`, landing at `yourdomain.com`.

## 1. Database

Create a **Managed PostgreSQL** database (Databases → Create) and copy its connection string for `FS_DB_URL`.

## 2. Backend (API)

1. Create an App from your repo and add a **Service** built from `backend/Dockerfile`.
2. Set environment variables ([Configuration](../reference/configuration.md)):
    - `FS_DB_URL` — the managed database connection string
    - `FS_ENVIRONMENT=prod`, `FS_BASE_API_URL=https://api.yourdomain.com`, `FS_BASE_WEB_URL=https://app.yourdomain.com`
    - `FS_JWT_SECRET_KEY`, `FS_CRON_SECRET`, plus Stripe/email keys
3. Run migrations against the managed DB: `./sqitch.sh prod deploy` (from a console or your machine, with the prod `FS_DB_URL`).
4. Add the domain `api.yourdomain.com` to the service.

## 3. App + landing (static sites)

Add two **Static Site** components from the same repo:

- **frontend** (`frontend/`) — build `npm run build`; env `PUBLIC_API_BASE_URL=https://api.yourdomain.com`; domain `app.yourdomain.com`.
- **landing** (`landing/`) — build `npm run build`; domain `yourdomain.com`.

## 4. Wire it together

Point DNS at the App Platform domains, confirm the `FS_BASE_*` URLs match the live ones, add the Stripe webhook at `https://api.yourdomain.com/webhooks/stripe` ([Billing & Subscriptions](../features/billing.md)), and run the [Security](../features/security.md) checklist.
