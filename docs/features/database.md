---
description: "FastSvelte database — multi-tenant PostgreSQL schema, Sqitch migrations, and the db_config connection layer."
keywords: "fastsvelte database, postgresql, sqitch migrations, multi-tenant schema, raw sql, database versioning, fastapi postgres"
---

# Database

FastSvelte uses **PostgreSQL** with a multi-tenant schema and **Sqitch** for migrations. Repositories use raw SQL (no ORM), so you always see exactly what runs.

## Multi-tenant schema

All business data is scoped to an **organization** (the tenant boundary). Core entities:

- `user` — individual accounts
- `organization` — the tenant; all business data belongs to one
- `role` — `readonly`, `member`, `org_admin`, `sys_admin` (see [Authentication](authentication.md))
- `session` — server-side sessions
- `plan` / `organization_plan` — subscription tiers and the org's current plan

This supports both individual users and teams without changing the data model. See [Multi-Tenancy (B2B)](multi-tenancy.md) for the organization/invitation model.

## Connection config

Connection settings come from `backend/.env` (`FS_DB_URL`, `FS_DB_SCHEMA`) via `backend/app/config/settings.py`, and are injected through the DI container's `db_config` into every repository — so repositories never construct their own connections.

## Migrations with Sqitch

Migrations are plain SQL files in `backend/db/`, versioned with Sqitch and reviewed alongside code in pull requests.

Create a migration:

```bash
cd backend/db
./sqitch.sh add add_feature -n "Add feature table"
```

This creates three files:

- `deploy/add_feature.sql` — how to apply the change
- `revert/add_feature.sql` — how to undo it
- `verify/add_feature.sql` — how to verify it worked

Deploy:

```bash
./sqitch.sh dev deploy
```

The `sqitch.sh` wrapper runs Sqitch via the official Docker image (no local Sqitch install needed) and adds FastSvelte-specific safety: per-environment database URLs (dev/beta/gamma/prod/test), a check that every migration is wrapped in `BEGIN;`/`COMMIT;`, revert protection (requires `--to`), and `.env` loading per environment.

### Why Sqitch and raw SQL?

Sqitch is language-agnostic and works with plain SQL — no ORM lock-in. You review and optimize the exact SQL that runs, and can still adopt SQLAlchemy or another tool later if your project needs it. See [Architecture](../reference/architecture.md) for the repository-layer reasoning.
