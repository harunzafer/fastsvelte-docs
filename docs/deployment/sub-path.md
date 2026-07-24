---
description: "Serve the FastSvelte app from a sub-path like yourdomain.com/app: one frontend setting, one backend setting, and a list of what stays unchanged."
keywords: "fastsvelte sub-path, base path, paths.base, sveltekit base path, subdirectory deployment, host under path"
---

# Serving the app from a sub-path

The standard setups serve the app from its own subdomain (`app.yourdomain.com`). If you serve it from a path prefix instead, such as `yourdomain.com/app`, two settings change: one in the frontend, one in the backend.

## 1. Frontend: set `paths.base`

In `frontend/svelte.config.js`:

```js
const config = {
	kit: { adapter: adapter(), paths: { base: '/app' } }
};
```

No trailing slash. Every internal link, redirect, and image in the app goes through SvelteKit's `resolve()` and `asset()` helpers, so all of them follow this setting. The eslint rule `svelte/no-navigation-without-resolve` keeps it that way: a hardcoded `<a href="/billing">` fails lint.

## 2. Backend: include the path in `FS_BASE_WEB_URL`

```bash
FS_BASE_WEB_URL=https://yourdomain.com/app
```

No trailing slash here either. Every link the backend hands out is built on this value: password reset and verification emails, invitation links, the OAuth redirect after Google sign-in, and the Stripe portal return URL.

## 3. Host: serve the build under the prefix

The app must be reachable under `/app`, and every `/app/*` path that is not a real file must answer with the app's `index.html` (HTTP 200, not a 404 or redirect). This step lives in your hosting setup, and each deployment guide has a **Serving the app from a sub-path** section with the exact change for that stack:

- [Self-Hosting](self-hosting.md#serving-the-app-from-a-sub-path) (nginx location block)
- [Railway](railway.md#serving-the-app-from-a-sub-path) (landing's Caddyfile)
- [DigitalOcean](digitalocean.md#serving-the-app-from-a-sub-path) (component Route setting)
- [Fly.io + Neon + Vercel](fly-neon-vercel.md#serving-the-app-from-a-sub-path) (landing's vercel.json rewrite)
- [Azure](azure.md#serving-the-app-from-a-sub-path) (Front Door path routing)

## What does not change

- **`PUBLIC_API_BASE_URL`**: the backend's absolute URL, independent of where the frontend lives.
- **CORS**: origins are scheme, host, and port. A path prefix plays no role.
- **Cookies**: the session cookie is set with `path=/` and works under any prefix.
- **Google OAuth console**: the authorized redirect URI points at the API, not the app.

## Verify locally

Build and preview with the base set, then click through login, dashboard, and billing. This uses `npm run preview` rather than the usual `npm run dev` on purpose: the dev server honors the base too, but preview serves the actual production build, which is what your host will serve.

```bash
cd frontend
npm run build
npm run preview
# visit http://localhost:4173/app/login
```
