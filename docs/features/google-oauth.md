---
description: "Add Google OAuth social login to FastSvelte — Google Cloud credentials, redirect URIs, environment configuration, the login flow, and adding more providers."
keywords: "fastsvelte google oauth, social login, oauth2, google sign in, fastapi oauth, sveltekit oauth"
---

# Google OAuth

FastSvelte supports social login via Google OAuth out of the box, alongside [email/password authentication](authentication.md). The architecture extends to other providers.

## Setup

1. Go to [console.cloud.google.com](https://console.cloud.google.com) and create (or select) a project.
2. Enable the Google OAuth2 API.
3. Create an **OAuth 2.0 Client ID** (Web application).
4. Add the authorized redirect URI: `http://localhost:8000/auth/oauth/google/callback` (and your production equivalent).

!!! note "JWT secret for OAuth state"

    OAuth flows use a JWT secret to sign the `state` parameter (CSRF protection). `init.py` auto-generates it. To generate manually:
    ```bash
    python -c "import secrets; print(secrets.token_urlsafe(32))"
    ```

## Configuration

```bash
# backend/.env
FS_GOOGLE_CLIENT_ID="your-google-client-id.apps.googleusercontent.com"
FS_GOOGLE_CLIENT_SECRET="GOCSPX-your-google-client-secret"

# Signs the OAuth state parameter (CSRF protection)
FS_JWT_SECRET_KEY="your-jwt-secret-key-here"
```

## The Login Flow

1. **Frontend** requests `GET /auth/oauth/google/authorize-url`.
2. **Backend** returns the Google authorization URL (with a signed `state`).
3. **User** authenticates with Google.
4. **Google** redirects to `GET /auth/oauth/google/callback` with an authorization code.
5. **Backend** validates `state`, exchanges the code, creates/loads the user, and sets the session cookie.
6. **Frontend** lands logged in. OAuth errors redirect to `/login?error=...`.

## Linking with an existing account

Sign-in with Google is matched to a user by **email address**, so a Google login and an email/password login for the same person resolve to one account, not two:

- **The email matches a verified account.** The Google login is linked to it, and that user can then sign in either way.
- **The email matches an account that never verified its email.** Google has now proven the person owns the address, so the account is claimed for them: its old password is removed, any existing sessions are signed out, and the email is marked verified. This closes a gap where someone could park on an email they do not own before the real owner signs in.
- **No account has that email.** A new account is created (in B2C). In B2B, sign-up is invitation-only, so an unknown email is turned away.

## Adding More Providers

To add GitHub, Microsoft, etc.:

1. Extend the helpers in `backend/app/util/oauth_util.py`.
2. Add provider routes in `backend/app/api/route/auth_route.py`.
3. Add the provider credentials to `backend/app/config/settings.py`.

## Troubleshooting

- **Redirect URI mismatch** — it must match the provider config exactly (scheme, host, port, path).
- **Invalid client** — recheck the client ID/secret.
- **HTTPS** — most providers require HTTPS redirect URIs in production.

See **[Authentication](authentication.md)** for sessions, roles, and the rest of the auth model.
