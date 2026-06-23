---
description: "FastSvelte security defaults — Argon2 password hashing, hashed session tokens, role-based access control, OAuth CSRF protection, AI spend caps, and per-environment cookie and CORS hardening."
keywords: "fastsvelte security, argon2, session security, rbac, csrf, cors, secure cookies, ai budget cap, saas security"
---

# Security

FastSvelte ships sensible security defaults so you start on solid ground.

## Passwords

Passwords are hashed with **Argon2id** (`backend/app/util/hash_util.py`), a modern memory-hard algorithm. Plaintext passwords are never stored.

## Sessions

- Tokens are 256-bit random values (`secrets.token_urlsafe(32)`).
- Only a **SHA-256 hash** of the token is stored server-side — a database leak doesn't expose usable session tokens.
- The cookie is **HttpOnly** (JavaScript can't read it), **Secure** outside `dev`, and **SameSite** `strict` outside `dev` (`lax` in dev).
- Logout invalidates the session server-side; expired sessions are pruned by the [cron job](../reference/configuration.md#background-jobs-cron).

See [Authentication](authentication.md) for the full model.

## Access control

Precedence-based roles (`readonly` < `member` < `org_admin` < `sys_admin`) gate every route via `min_role_required(...)`. All business data is organization-scoped, so tenants are isolated — see [Multi-Tenancy](multi-tenancy.md).

## OAuth CSRF protection

The Google OAuth flow signs a `state` parameter (JWT, `FS_JWT_SECRET_KEY`) and validates it on callback. See [Google OAuth](google-oauth.md).

## AI spend protection

AI usage is **hard-capped by default**: when an organization exhausts its allotment + credits, calls are blocked rather than silently billed. Overage requires turning on *both* an org setting and a system-level kill switch, neither enabled by default — see [AI Usage & Credit Billing](ai-billing.md#overage-settings). This protects you from runaway model spend.

## CORS & email verification

Allowed origins are configured per environment in `backend/app/config/settings.py` and tighten in production. Accounts must verify their email before they can log in.

## Not included (add as needed)

To stay lean, FastSvelte does **not** bundle rate limiting or security-header middleware (CSP/HSTS). Add them when your threat model calls for it — e.g. a rate-limiting dependency in front of auth/AI endpoints, and a middleware that sets security headers.

## Production checklist

- Strong, unique `FS_JWT_SECRET_KEY` and `FS_CRON_SECRET`.
- Serve over HTTPS so `Secure` cookies take effect.
- Strict `FS_CORS_ORIGINS` for production.
- Stripe live keys + verified webhook secret (see [Billing & Subscriptions](billing.md)).
