---
description: "Deploy FastSvelte end-to-end on Railway — the FastAPI backend, PostgreSQL, and the SvelteKit app + landing, with custom domains."
keywords: "deploy fastsvelte railway, railway fastapi, railway postgres, railway sveltekit, railway docker deploy"
---

# Deploy to Railway

Railway is the simplest all-in-one path: it runs your FastAPI container, a managed PostgreSQL, and the static app + landing from one project, redeploying on every git push.

**Outcome:** API at `api.yourdomain.com`, app at `app.yourdomain.com`, landing at `yourdomain.com`.

## 1. Create the project + database

1. Create a [Railway](https://railway.app) project from your repo — it detects `backend/Dockerfile`.
2. Add a **PostgreSQL** service (New → Database → PostgreSQL); Railway provisions its connection string.

## 2. Deploy the backend (API)

1. In the backend service → **Variables**, set the FastSvelte env (full list in [Configuration](../reference/configuration.md)):
    - `FS_DB_URL` — reference the Postgres service's connection string
    - `FS_ENVIRONMENT=prod`, `FS_BASE_API_URL=https://api.yourdomain.com`, `FS_BASE_WEB_URL=https://app.yourdomain.com`
    - `FS_JWT_SECRET_KEY`, `FS_CRON_SECRET`, plus Stripe/email keys as needed
2. Run migrations once against the prod database — `./sqitch.sh prod deploy` (with your prod sqitch target pointed at `FS_DB_URL`), or from a one-off Railway shell.
3. Under **Settings → Networking**, add the custom domain `api.yourdomain.com`.

## 3. Deploy the app + landing (static)

`frontend/` and `landing/` are static SvelteKit builds. Add a static service for each that runs `npm install && npm run build` and serves the build output:

- Set `PUBLIC_API_BASE_URL=https://api.yourdomain.com` (and any other `PUBLIC_*`) before the build.
- Attach `app.yourdomain.com` to the frontend and `yourdomain.com` to the landing.

Prefer Vercel for the static sites? The Vercel steps in [Fly.io + Neon + Vercel](fly-neon-vercel.md) apply to any backend host.

## 4. Wire it together

- Point DNS for `api`, `app`, and the apex at the Railway domains.
- Confirm `FS_BASE_API_URL` / `FS_BASE_WEB_URL` match the live URLs so CORS and cookies work ([Configuration](../reference/configuration.md)).
- Add the Stripe webhook at `https://api.yourdomain.com/webhooks/stripe` ([Billing & Subscriptions](../features/billing.md)).
- Review the [Security](../features/security.md) checklist before launch.
