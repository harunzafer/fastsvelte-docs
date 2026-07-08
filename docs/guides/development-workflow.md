---
description: "The FastSvelte development loop — make changes across database, backend, and frontend, regenerate the API client, and run tests and linters."
keywords: "fastsvelte workflow, development loop, sqitch, api client generation, pytest, vitest, hot reload"
---

# Development Workflow

After this guide you can make a change across the stack and run the app. Work in the **DB → Backend → Frontend** order so the API client regenerates against an up-to-date backend.

## Database changes

```bash
cd backend/db
./sqitch.sh add feature_name -n "Description"
# Edit the generated deploy/, revert/, verify/ SQL files
./sqitch.sh dev deploy
```

See [Database](../features/database.md) for the migration model.

## Backend changes

```bash
cd backend

uv add package_name                    # add a dependency (--dev for dev-only)
uv run ruff check .                    # lint
uv run pytest                          # test
uv run uvicorn app.main:app --reload   # run (auto-reloads on change)
```

## Frontend changes

After backend API changes, regenerate the typed client (see [Type-Safe API Client](orval.md)):

```bash
cd frontend

npm run generate    # regenerate the API client (backend must be running)
npm run format      # Prettier
npm run check       # type checking
npm run test        # tests (also test:unit / test:e2e)
npm run dev         # dev server (hot reload)
```
