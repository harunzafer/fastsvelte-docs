---
description: "FastSvelte transactional email — configure Resend, SendGrid, or Azure Communication Services (or a dev stub) for verification, password reset, and invitation emails."
keywords: "fastsvelte email, transactional email, resend, sendgrid, azure communication services, email provider, fastapi email"
---

# Transactional Email

FastSvelte sends transactional email — verification, password reset, and organization invitations — through a pluggable provider. Pick one of **Resend**, **SendGrid**, or **Azure Communication Services**, or use the **stub** provider in development.

## Choosing a Provider

Set `FS_EMAIL_PROVIDER` in `backend/.env` to one of: `resend`, `sendgrid`, `azure`, or `stub`.

For the providers you don't use, clean up all three places:

1. **`backend/pyproject.toml`** — delete the two package lines you don't need, then run `uv sync`:

    ```toml
    "azure-communication-email>=1.1.0",  # FS_EMAIL_PROVIDER=azure
    "resend[async]>=2.30.1",             # FS_EMAIL_PROVIDER=resend
    "sendgrid>=6.12.5",                  # FS_EMAIL_PROVIDER=sendgrid
    ```

    ```bash
    cd backend && uv sync
    ```

2. **`backend/.env`** — remove the env vars for the unused providers.
3. **`backend/app/config/settings.py`** — remove the corresponding settings fields (e.g. `resend_api_key`, `resend_sender_address`, `resend_sender_name`).

## Resend

A modern email API with a generous free tier — recommended for most new projects.

1. Sign up at [resend.com](https://resend.com) and create an API key at [resend.com/api-keys](https://resend.com/api-keys).
2. Add and verify your sending domain at [resend.com/domains](https://resend.com/domains).
3. Configure `backend/.env`:

```bash
FS_EMAIL_PROVIDER="resend"
FS_RESEND_API_KEY="re_your-resend-api-key-here"
FS_RESEND_SENDER_ADDRESS="noreply@yourdomain.com"
FS_RESEND_SENDER_NAME="Your App Name"
```

## SendGrid

```bash
FS_EMAIL_PROVIDER="sendgrid"
FS_SENDGRID_API_KEY="SG.your-sendgrid-api-key-here"
FS_SENDGRID_SENDER_ADDRESS="noreply@yourdomain.com"
FS_SENDGRID_SENDER_NAME="Your App Name"
```

Verify your sender email/domain in the SendGrid dashboard first.

## Azure Communication Services

For Azure-based deployments:

```bash
FS_EMAIL_PROVIDER="azure"
FS_AZURE_EMAIL_CONNECTION_STRING="endpoint=https://..."
FS_AZURE_EMAIL_SENDER_ADDRESS="noreply@yourdomain.com"
```

## Development (Stub)

For local development without sending real email:

```bash
FS_EMAIL_PROVIDER="stub"
```

Emails are logged to the backend console instead of sent — look for `[STUB EMAIL]` blocks containing verification and invitation links.

## Testing

```bash
curl -X POST http://localhost:8000/password/forgot \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com"}'
```

## Troubleshooting

- **Not sending** — verify the API key, confirm the sender domain/address is verified with the provider, and check the backend logs.
- **Production** — set up SPF/DKIM for your sending domain.
