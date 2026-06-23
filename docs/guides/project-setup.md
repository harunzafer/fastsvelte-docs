---
description: "How a FastSvelte project is set up by init.py, where configuration lives, and how to onboard teammates without re-running init."
keywords: "fastsvelte setup, init.py, project setup, env files, team onboarding, monorepo setup"
---

# Project Setup

After this guide you'll understand what `init.py` configured, where configuration lives, and how to onboard a teammate.

## What `init.py` does

You run `init.py` once during [Quick Start](../index.md). It:

- prompts for your app name, mode (`b2c` / `b2b`), and database;
- generates the `.env` files across `backend/`, `frontend/`, `landing/`, and `backend/db/`;
- starts Docker PostgreSQL and runs migrations (with seed data);
- installs backend (`uv`) and frontend/landing (`npm`) dependencies;
- generates the type-safe API client (see [Type-Safe API Client](orval.md)).

Pass `--dry-run` to preview without making changes. In development the email provider defaults to `stub`, so nothing is sent — verification and invitation links appear as `[STUB EMAIL]` blocks in the backend logs (see [Transactional Email](../features/email.md)). Once setup works, you can safely delete `init.py`; it's only needed once.

## Onboarding teammates

**Only the first developer runs `init.py`.** Everyone else starts from your committed, pre-configured repository.

1. **Push to your own repository:**

    ```bash
    git remote add origin git@github.com:your-org/your-project.git
    git push -u origin main
    ```

2. **Teammates clone and configure:**

    - Clone your repository (not the FastSvelte template).
    - Install the [prerequisites](../index.md) (uv, Node.js 22+, Sqitch, Docker).
    - Create `.env` files by copying each `.env.example` (`backend/`, `frontend/`, `landing/`, `backend/db/`).
    - Start the database: `docker compose -f backend/docker-compose.yml up db -d`
    - Run migrations: `cd backend/db && ./sqitch.sh dev deploy`
    - Install deps: `uv sync` in `backend/`, `npm install` in `frontend/` and `landing/`
    - Create a local admin: `cd backend && uv run scripts/create_admin.py`

!!! warning "Never commit `.env` files"

    They contain secrets and are gitignored. Share values through a secure channel (1Password, Doppler, a shared secrets manager).

See [Configuration](../reference/configuration.md) for the full variable reference.
