---
description: "FastSvelte is a fullstack SaaS starter kit built with FastAPI (Python) and SvelteKit SPA (TypeScript). Get session auth, Google OAuth, Stripe, multi-tenancy, AI copilot with credit billing, and admin dashboards out of the box."
keywords: "fastapi, sveltekit, saas starter kit, fastapi svelte, fastapi sveltekit starter, sveltekit spa fastapi, saas starter kit python, ai saas starter kit, orval sveltekit, svelte 5, typescript, fullstack, authentication, multi-tenant, stripe payments, ai credit billing"
---

# Quick Start

[FastSvelte](https://fastsvelte.dev) is a fullstack SaaS starter kit built with **FastAPI** (Python) and **SvelteKit** (TypeScript). Get it running locally in a few minutes, then browse [Features](features/authentication.md) for everything that's included.

The interactive `init.py` script handles setup end to end. Clone the repo, run `init.py`, then create an admin account and start the servers:

```sh
# 1. Clone, then drop the template remote
git clone https://github.com/harunzafer/fastsvelte.git my-project
cd my-project && git remote remove origin

# 2. Interactive setup — sets up your project for you (--dry-run to preview)
uv run init.py

# 3. Create a system admin, then start the backend
cd backend
uv run scripts/create_admin.py
uv run uvicorn app.main:app --reload

# 4. In a new terminal, start the frontend
cd frontend && npm run dev
```

!!! info "Prerequisites"

    You'll need [uv](https://docs.astral.sh/uv/#installation), **Node.js 22+**, [Sqitch](https://sqitch.org/download/), and **Docker** (running). `init.py` checks all four first and exits with install instructions if any are missing — so you can just run it and fix what it flags.

Open **[localhost:5173](http://localhost:5173)** and log in with the admin you created. The backend and interactive API docs are at [localhost:8000/docs](http://localhost:8000/docs); the landing template runs at [localhost:5174](http://localhost:5174).

## Next steps

- **[Guides](guides/project-setup.md)** — set up your project, develop, and ship
- **[Features](features/authentication.md)** — auth, billing, AI, multi-tenancy, and more
- **[Architecture](reference/architecture.md)** · **[Deployment](deployment/index.md)**
