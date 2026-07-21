---
description: "FastSvelte security defaults: Argon2 password hashing, hashed session tokens, role-based access control, OAuth CSRF protection, AI spend caps, per-IP rate limiting, HTTP security headers, and per-environment cookie and CORS hardening."
keywords: "fastsvelte security, argon2, session security, rbac, csrf, cors, secure cookies, ai budget cap, rate limiting, security headers, content security policy, saas security"
---

# Security

FastSvelte ships sensible security defaults so you start on solid ground.

## Passwords

Passwords are hashed with **Argon2id** (`backend/app/util/hash_util.py`), a modern memory-hard algorithm. Plaintext passwords are never stored.

Changing a password from the profile page requires the current one, so a stolen session cookie is not enough to take over the account. Accounts created through Google have no password, so they see an account panel instead of the form.

A password change also signs the user out on other devices (`backend/app/service/password_service.py`):

- **Changed from the profile page:** other sessions are revoked, the current one is kept.
- **Reset with an emailed link:** all sessions are revoked, including any an attacker is holding.

## Sessions

- Tokens are 256-bit random values (`secrets.token_urlsafe(32)`).
- Only a **SHA-256 hash** of the token is stored server-side, so a database leak doesn't expose usable session tokens.
- The cookie is **HttpOnly** (JavaScript can't read it), **Secure** outside `dev`, and **SameSite** `strict` outside `dev` (`lax` in dev).
- Logout invalidates the session server-side; expired sessions are pruned by the [cron job](../reference/configuration.md#background-jobs-cron).

See [Authentication](authentication.md) for the full model.

## Access control

Precedence-based roles (`readonly` < `member` < `org_admin` < `sys_admin`) gate every route via `min_role_required(...)`. All business data is organization-scoped, so tenants are isolated. See [Multi-Tenancy](multi-tenancy.md).

## OAuth CSRF protection

The Google OAuth flow signs a `state` parameter (JWT, `FS_JWT_SECRET_KEY`) and validates it on callback. See [Google OAuth](google-oauth.md).

## AI spend protection

AI usage is **hard-capped by default**: when an organization exhausts its allotment + credits, calls are blocked rather than silently billed. Overage requires turning on *both* an org setting and a system-level kill switch, neither enabled by default. See [AI Usage & Credit Billing](ai-billing.md#overage-settings). This protects you from runaway model spend.

## CORS & email verification

Allowed origins are configured per environment in `backend/app/config/settings.py` and tighten in production. Accounts must verify their email before they can log in.

## Rate limiting

The abuse-prone public endpoints are rate limited per client IP, so one source can't hammer them:

| Endpoint | Limit |
|----------|-------|
| `POST /auth/login` | 5 / 15 min |
| `POST /auth/signup`, `POST /auth/signup-org` | 3 / hour |
| `POST /password/forgot` | 3 / hour |
| `POST /auth/resend-verification` | 10 / min |

The priority is the endpoints that **send email to a user-supplied address** (signup, forgot-password, resend-verification): left open they let an attacker burn your email spend and sender reputation regardless of how much traffic you have. Login is capped against credential stuffing. A breach returns the standard `ErrorResponse` with a `Retry-After` header.

To rate limit another route, add the dependency to it:

```python
from fastapi import Depends
from app.util.rate_limit import rate_limit

@router.post("/expensive", dependencies=[Depends(rate_limit("10/minute"))])
```

### Storage (read before scaling)

Counters live in **process memory by default** (`FS_RATE_LIMIT_STORAGE_URI=async+memory://`). The shipped container runs a single process, so this is correct for **one instance**. But each instance keeps its own counters, so if you run **more than one instance** (horizontal scaling / replicas behind a load balancer) the effective limit multiplies by the number of instances. For a real limit across instances, point it at a shared store:

```
FS_RATE_LIMIT_STORAGE_URI=async+redis://your-redis-host:6379/0
```

Async Redis needs the `coredis` package (`uv add coredis`).

### Behind a proxy

The limiter reads the client IP from `X-Forwarded-For`, which hosting platforms set for you. If you'd rather rate limit at the edge, a reverse proxy like nginx can do it too. See its [`limit_req` docs](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html).

### Deliberately left out

Kept out to stay lean; add if your threat model calls for it:

- **Per-account login limiting** (keying login on the target email, not just the IP) defends a distributed attack against one account. Worth adding once you have accounts worth attacking.
- **The AI endpoint** is authenticated, so it's not a public abuse surface, and it's already hard-capped by the AI spend protection above. It gets no separate HTTP limit.
- **Reset and verify token endpoints** rely on high-entropy tokens rather than a request limit.

## HTTP security headers

FastSvelte protects API responses by default and provides a ready-to-use header policy for your frontend.

### API: already protected

FastSvelte adds these headers to every API response, including errors:

- **No caching:** private API data is not stored in browser or shared caches.
- **No content guessing:** browsers cannot treat JSON as HTML or another unsafe type.
- **No embedding:** API responses cannot be displayed inside another website.
- **No referrer leakage:** URLs and IDs are not passed to third-party sites.

No setup is required. These are set in `backend/app/api/middleware/security_headers.py`.

### Frontend: add at your host

Your frontend host, CDN, or reverse proxy must send these headers for the app. They help ensure the app loads only trusted resources, stays HTTPS-only, cannot be embedded by another site, and does not enable unused browser features.

The headers, and where to put them, are in the guide for your host: [Vercel](../deployment/fly-neon-vercel.md#security-headers), [Railway](../deployment/railway.md#security-headers), [Azure](../deployment/azure.md#security-headers), [DigitalOcean](../deployment/digitalocean.md#security-headers), [self-hosting](../deployment/self-hosting.md#security-headers).

!!! note "Why the policy allows inline styles"

    The policy sets `style-src 'self' 'unsafe-inline'`. Svelte animations inject an inline `<style>` element at runtime, so transitions break without it. This applies to styles only. Scripts stay limited to your own domain, which is where the real risk lies.

### Check after deploy

```bash
curl -sI https://app.yourdomain.com | grep -i "content-security\|x-content-type\|referrer\|strict-transport"
curl -sI https://api.yourdomain.com/ping | grep -i "content-security\|x-content-type"
```

If the app does not load correctly, check the browser console for CSP errors. Most often, the API URL is missing from `connect-src`.

## Production checklist

- Strong, unique `FS_JWT_SECRET_KEY` and `FS_CRON_SECRET`.
- Serve over HTTPS so `Secure` cookies take effect.
- Strict `FS_CORS_ORIGINS` for production.
- Stripe live keys + verified webhook secret (see [Billing & Subscriptions](billing.md)).
- Add the [frontend security headers](#frontend-add-at-your-host) at your host, with your real API URL in `connect-src`.
