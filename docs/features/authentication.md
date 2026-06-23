---
description: "FastSvelte authentication — session-cookie auth, email/password and Google OAuth login, email verification, password reset, and role-based access control."
keywords: "fastsvelte authentication, session auth, google oauth, email verification, password reset, rbac, roles, fastapi auth"
---

# Authentication

FastSvelte ships session-based authentication out of the box: email/password and Google OAuth login, email verification, password reset, and role-based access control. Sessions are server-side and carried in an HTTP-only cookie.

## Roles

Roles are precedence-ordered (`backend/app/model/role_model.py`). Endpoints are guarded by `min_role_required(Role.X)` — a user satisfies the requirement if their role's precedence is **at least** the required role's.

| Role | DB name | Precedence |
|------|---------|-----------|
| `READONLY` | `readonly` | 0 |
| `MEMBER` | `member` | 1 |
| `ORG_ADMIN` | `org_admin` | 2 |
| `SYSTEM_ADMIN` | `sys_admin` | 3 |

## Sessions

On login the backend creates a server-side session and sets a cookie (`backend/app/util/cookie_util.py`):

- **Name** — `session_id` (configurable via `FS_SESSION_COOKIE_NAME`).
- **`HttpOnly`** — always, so JavaScript can't read the token.
- **`Secure`** — on in `beta`/`prod`, off in `dev`.
- **`SameSite`** — `strict` in `beta`/`prod`, `lax` in `dev`.
- **Max age** — `FS_SESSION_COOKIE_MAX_AGE` (default 24h).

`POST /auth/logout` invalidates the session server-side and clears the cookie. Expired sessions are pruned by the cron job (`/cron`, gated by `FS_CRON_SECRET`; retention `FS_CRON_SESSION_RETENTION_DAYS`, default 7 days).

## Email & password

| Endpoint | Purpose |
|----------|---------|
| `POST /auth/signup` | Create a user (B2C — see [modes](#b2c-vs-b2b)) |
| `POST /auth/signup-org` | Create an organization + its admin (B2C) |
| `POST /auth/login` | Log in; sets the session cookie |
| `POST /password/forgot` | Email a password-reset link |
| `POST /password/reset` | Reset the password with a token |
| `POST /password/update` | Change the password while logged in (`READONLY`+) |

Login requires a **verified email** — unverified accounts get `EmailNotVerified`. Password-reset emails are sent through the configured [email provider](email.md).

## Google OAuth

| Endpoint | Purpose |
|----------|---------|
| `GET /auth/oauth/google/authorize-url` | Returns the Google authorization URL |
| `GET /auth/oauth/google/callback` | Handles the redirect, creates the session, redirects to the app |

The flow uses a signed `state` parameter for CSRF protection (`backend/app/util/oauth_util.py`), validated on callback. Errors (cancelled, invalid state, etc.) redirect back to `/login?error=...`. For Google Cloud credentials and redirect-URI setup, see the [Integrations guide](google-oauth.md); to add other providers, extend `oauth_util.py` and add routes in `auth_route.py`.

## B2C vs B2B

Signup behavior depends on the app mode:

- **B2C** — public `signup` and `signup-org` are open.
- **B2B** — both return `404`; users join only via [organization invitations](multi-tenancy.md). Only a `SYSTEM_ADMIN` creates organizations.

See **[B2B Mode](multi-tenancy.md)** for the full multi-tenant model.

## Configuration

```bash
# Google OAuth
FS_GOOGLE_CLIENT_ID="your-google-client-id.apps.googleusercontent.com"
FS_GOOGLE_CLIENT_SECRET="GOCSPX-your-google-client-secret"

# Signs the OAuth state parameter (CSRF protection)
FS_JWT_SECRET_KEY="your-jwt-secret-key"

# Sessions
FS_SESSION_COOKIE_NAME="session_id"
FS_SESSION_COOKIE_MAX_AGE=86400

# Session-cleanup cron
FS_CRON_SECRET="your-secure-cron-secret"
FS_CRON_SESSION_RETENTION_DAYS=7
```

## Next steps

- **[Integrations](google-oauth.md)** — Google OAuth provider setup and email configuration.
- **[B2B Mode](multi-tenancy.md)** — organizations, invitations, and roles.
