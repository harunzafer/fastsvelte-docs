---
description: "Deploy FastSvelte best-of-breed: FastAPI on Fly.io, PostgreSQL on Neon, and the SvelteKit app + landing on Vercel."
keywords: "deploy fastsvelte fly.io, neon postgres, vercel sveltekit, fastapi fly, best-of-breed saas deploy"
---

# Deploy: Fly.io + Neon + Vercel

A best-of-breed split: the **API on Fly.io** (global containers), **PostgreSQL on Neon** (serverless), and the **app + landing on Vercel** (static, edge CDN). Pick this when you want each piece on the platform that does it best.

**Outcome:** API at `api.yourdomain.com`, app at `app.yourdomain.com`, landing at `yourdomain.com`.

## 1. Database: Neon

1. Create a project at [Neon](https://neon.tech) and copy the **connection string** (use the pooled one for serverless).
2. You'll set it as `FS_DB_URL` on the API in the next step.

## 2. API: Fly.io

1. Install [flyctl](https://fly.io/docs/flyctl/install/), then run `fly launch` in `backend/` (it detects the Dockerfile, but don't deploy yet).
2. Set secrets ([Configuration](../reference/configuration.md)):

    ```bash
    fly secrets set \
      FS_DB_URL="<neon-connection-string>" \
      FS_ENVIRONMENT=prod \
      FS_BASE_API_URL=https://api.yourdomain.com \
      FS_BASE_WEB_URL=https://app.yourdomain.com \
      FS_JWT_SECRET_KEY=... FS_CRON_SECRET=...
    ```

3. `fly deploy`, then run migrations against Neon: `./sqitch.sh prod deploy` (prod target = `FS_DB_URL`).
4. Add the domain: `fly certs add api.yourdomain.com`, then point DNS at Fly.

## 3. App + landing: Vercel

Create two Vercel projects from the same repo:

- **frontend**: root directory `frontend/`, build `npm run build`, env `PUBLIC_API_BASE_URL=https://api.yourdomain.com`; domain `app.yourdomain.com`.
- **landing**: root directory `landing/`, build `npm run build`; domain `yourdomain.com`.

Vercel auto-deploys on push and provisions SSL.

## 4. Wire it together

- DNS: `api` → Fly.io; `app` and the apex → Vercel.
- Confirm `FS_BASE_API_URL` / `FS_BASE_WEB_URL` match the live URLs so CORS and cookies work ([Configuration](../reference/configuration.md)).
- Stripe webhook → `https://api.yourdomain.com/webhooks/stripe` ([Billing & Subscriptions](../features/billing.md)).
- Review the [Security](../features/security.md) checklist before launch.

## Next steps

### Security headers

Add security headers to the app by creating `frontend/vercel.json`:

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "Content-Security-Policy",
          "value": "default-src 'self'; connect-src 'self' https://api.yourdomain.com; img-src 'self' data:; style-src 'self' 'unsafe-inline'; script-src 'self'; font-src 'self'; object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none'"
        },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
        { "key": "Strict-Transport-Security", "value": "max-age=31536000; includeSubDomains" },
        { "key": "Permissions-Policy", "value": "camera=(), microphone=(), geolocation=()" }
      ]
    }
  ]
}
```

Replace `https://api.yourdomain.com` in `connect-src` with your real API URL, or the browser will block the app from calling it. See [Security](../features/security.md#frontend-add-at-your-host).
