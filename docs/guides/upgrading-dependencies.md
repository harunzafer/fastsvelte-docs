---
description: "Keep a FastSvelte project's dependencies current and secure — the Dependabot policy, backend (uv) and frontend (npm) upgrade workflows, and major-version upgrades."
keywords: "fastsvelte dependencies, dependabot, uv upgrade, npm update, dependency management, major version upgrade"
---

# Upgrading Dependencies

After this guide you can keep dependencies current and secure using Dependabot plus a verify workflow.

## Upgrade policy

FastSvelte ships a [Dependabot config](https://github.com/) (`.github/dependabot.yml`) so routine upgrades are automated and reviewable:

- **Automated, weekly:** Dependabot opens **grouped** PRs once a week per ecosystem — `npm` (frontend), `npm` (landing), and `uv` (backend). The backend uses a 7-day cooldown so brand-new releases settle before they're proposed.
- **Minor & patch upgrades:** handled by those weekly PRs. Review and merge once CI passes (`backend.yml`, `frontend.yml`, `landing.yml`).
- **Major versions:** **excluded** from Dependabot (`version-update:semver-major` is ignored) and done deliberately, one component at a time, since they may require code changes — follow the major-version workflow below.
- **Security updates:** fast-track immediately, outside the weekly cadence.
- **Version pinning:** `pyproject.toml` and `package.json` declare lower-bound (`>=`) ranges; the lockfiles (`uv.lock`, `package-lock.json`) pin exact versions and are committed — so installs are reproducible while ranges stay flexible.

## Backend dependencies (Python)

FastSvelte uses `uv` to manage Python dependencies.

```bash
cd backend

# Upgrade all dependencies (or a specific one)
uv sync --upgrade
uv add --upgrade package_name

# Verify compatibility
uv run pytest
```

**Note:** Smoke tests in `backend/test/smoke/` verify app startup, database connectivity, and API functionality. Database tests require PostgreSQL running (`docker compose up db -d`) but skip gracefully if unavailable.

## Frontend dependencies (npm)

The frontend and landing page use `package.json` + `package-lock.json`.

```bash
# Frontend
cd frontend
npm outdated                 # check what's outdated
npm update                   # update within semver ranges
npm run build && npm run check && npm run lint && npm run test

# Landing (simpler, no tests by default)
cd ../landing
npm outdated && npm update
npm run build && npm run check && npm run lint
```

### Major version upgrades

```bash
npm install -g npm-check-updates   # one-time

cd frontend   # or cd landing
ncu           # preview
ncu -u        # bump package.json to latest
npm install
npm run build && npm run check && npm run lint
npm run test  # frontend only (landing has no tests by default)
```

**Important:**

- Commit `package-lock.json` after upgrades.
- Test thoroughly after major updates, and upgrade one component at a time (backend, frontend, landing) for easier troubleshooting.
- Frontend smoke tests live in `frontend/src/tests/smoke/` and run with `npm run test`.
- To add tests to landing, run `npx sv add vitest`.
