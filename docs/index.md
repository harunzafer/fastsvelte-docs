---
description: "FastSvelte is a fullstack SaaS starter kit and boilerplate built with FastAPI (Python) and SvelteKit SPA (TypeScript). Get session auth, Google OAuth, Stripe, multi-tenancy, and admin dashboards out of the box."
keywords: "fastapi, sveltekit, saas starter kit, fastapi svelte, fastapi sveltekit starter, sveltekit spa fastapi, saas starter kit python, fastapi fullstack template, sveltekit saas boilerplate, fastapi boilerplate, python saas boilerplate, orval sveltekit, svelte 5, typescript, fullstack, authentication, multi-tenant, stripe payments"
---

# FastSvelte

[FastSvelte](https://fastsvelte.dev) is a fullstack SaaS starter kit built with **FastAPI** (Python) and **SvelteKit** (TypeScript). Get authentication, multi-tenancy, and payments out of the box.

## Quick Start

Get up and running in minutes:

```bash
# Clone the repository
git clone https://github.com/harunzafer/fastsvelte.git my-project
cd my-project

# Run the interactive setup
uv run init.py

# Create admin user
cd backend
uv run scripts/create_admin.py

# Start the backend
uv run uvicorn app.main:app --reload

# In a new terminal, start the frontend
cd frontend
npm run dev
```

Visit `http://localhost:5173` and login with your admin credentials.

**→ [Complete Setup Guide](getting-started.md)** for detailed configuration and troubleshooting.

---

## Why FastSvelte?

- **Skip months of setup work** - Authentication, payments, and multi-tenancy are already built and tested.
- **Type-safe from API to UI** - Auto-generated TypeScript client means fewer bugs and better DX.
- **Built for B2B and B2C** - Organization-based architecture adapts to your business model. See [B2B Mode](b2b-mode.md).
- **Production-ready from day one** - Session auth, migrations, Docker deployment all configured.
- **Clean, maintainable code** - Dependency injection, repository patterns, and clear separation of concerns make scaling effortless.
- **AI-friendly architecture** - Structured patterns and comprehensive docs help AI assistants generate correct code.

## Technology Stack

- **Backend**: FastAPI with PostgreSQL, Sqitch migrations, and uv for package management
- **Frontend**: SvelteKit with TypeScript, TailwindCSS, and DaisyUI
- **Authentication**: Session-based with HTTP-only cookies and Google OAuth
- **Payments**: Stripe integration with webhook handling
- **Deployment**: Docker containers

## What's Included

Four main components:

- **Backend API** - FastAPI application with authentication, multi-tenancy, and payments
- **Frontend Dashboard** - SvelteKit interface with system admin, organization admin (B2B mode), and user dashboards
- **Landing Page** - Marketing site template
- **Database Schema** - PostgreSQL schema with migration system

### Demo Application: AI Note Improver

FastSvelte includes a complete **AI Note Improver** demo application that showcases the kit's capabilities:

- **Full CRUD Operations** - Create, read, update, and delete notes
- **AI Integration** - OpenAI-powered note organization and improvement
- **Usage-Based Billing** - Token consumption tracking with plan-based limits
- **Multi-tenant Architecture** - Organization-level quotas and permissions

This demo application demonstrates how to build AI-powered SaaS features while maintaining proper authentication, authorization, and billing controls. **Replace this demo with your own core functionality** while keeping the underlying infrastructure.

---

## Next Steps

- **[Getting Started](getting-started.md)** - Complete setup guide with all configuration options
- **[Development Guide](development.md)** - Development workflows and common tasks
- **[Architecture Overview](architecture-overview.md)** - Understand how FastSvelte is built
- **[Tutorials](tutorials.md)** - Learn to customize and extend your app
- **[Integrations](integrations/index.md)** - Configure Stripe, SendGrid, OAuth, and more
- **[Deployment](deployment/index.md)** - Deploy to Azure, Docker, or other platforms
- **[Troubleshooting](troubleshooting.md)** - Common issues and solutions
