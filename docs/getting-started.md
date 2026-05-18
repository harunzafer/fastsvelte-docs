---
description: "Get FastSvelte running in under 10 minutes. Complete setup guide for FastAPI backend, SvelteKit frontend, PostgreSQL database, and authentication system."
keywords: "fastsvelte setup, getting started, fastapi setup, sveltekit installation, postgresql, docker, development environment"
---

# Getting Started

This guide will get you up and running with [FastSvelte](https://fastsvelte.dev) in under 10 minutes. By the end, you'll have a fully functional SaaS application running locally with authentication, user management, and payment processing.

## Prerequisites

Before you begin, make sure you have the following installed:

- **uv** - Fast Python package and project manager ([installation guide](https://docs.astral.sh/uv/#installation))
- **Node.js 22+** - For the SvelteKit frontend
- **Sqitch** - For database migrations ([installation guide](https://sqitch.org/download/))
- **Docker** - For running PostgreSQL and containerized setup

### For Integrations (Install When Needed)

- **Stripe CLI** - Required for payment webhooks ([Stripe integration guide](integrations/stripe.md))
- **Google OAuth** - Required for social login ([OAuth integration guide](integrations/index.md#google-oauth))

### Optional but Recommended

- **pgAdmin** or similar PostgreSQL client for database management
- [**REST Client** VS Code extension](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) (Recommended), **Insomnia**, or **Postman** for testing endpoints

## Quick Setup (3 Steps)

### 1. Clone the Repository

```sh
# Clone the repository (choose SSH or HTTPS)

# Option 1: SSH (recommended if you have SSH keys configured)
git clone git@github.com:harunzafer/fastsvelte.git my-project

# Option 2: HTTPS (requires Personal Access Token)
git clone https://github.com/harunzafer/fastsvelte.git my-project

cd my-project

# Remove the original remote (prevents accidental pushes to FastSvelte repo)
git remote remove origin
```

!!! note

    GitHub no longer supports password authentication. If you don't have SSH keys set up, you'll need to use a [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) instead of your password.

### 2. Run Interactive Setup

```sh
# Run the interactive setup script
uv run init.py

# Or preview changes without executing (dry-run mode)
uv run init.py --dry-run
```

The script will:

- Check that all prerequisites are installed
- Ask you for configuration (app name, mode, database settings, etc.)
- Generate all `.env` files with your configuration
- Update database schema names
- Set up Docker PostgreSQL database
- Run database migrations (includes seed data)
- Set up backend (Python virtual environment + dependencies)
- Set up frontend (npm install + API client generation)
- Set up landing page (npm install)

This takes about 3-5 minutes depending on your internet speed.

!!! tip

    Use `--dry-run` to preview what the script will do without making any changes.

!!! info "Cleanup After Setup"

    Once your setup is complete and working, you can safely delete `init.py` - it's only needed for initial setup. The same applies to the `/docs` folder: either update it for your own project documentation or delete it entirely.

#### Review and Commit Changes

After init.py completes successfully, review what was changed:

```zsh
# See all modified files
git status

# Review all changes
git diff

# Commit the customizations
git add .
git commit -m "Initial project setup and customization"
```

!!!info "LLM Instructions"

    Update CLAUDE.md with your project description. Edit the Project Overview section to describe your application and replace the TODO placeholder with your app's purpose and features

### 3. Create Your First Admin User

Once the init script completes successfully, create a system administrator account:

```bash
cd backend
uv run scripts/create_admin.py
```

The script will prompt you for:

- **Email address** - Defaults to `admin+dev@{yourapp}.com` for localhost databases
- **Password** - Minimum 8 characters
- **First name** - Defaults to "System"
- **Last name** - Defaults to "Admin"

**What happens:**

- Creates a `sys_admin` user in the "System Administration" organization
- The `sys_admin` role has application-wide permissions (not limited to one organization)
- No email verification required

!!! tip "Email Aliases for Environments"

    We recommend different email aliases for each environment:

    - **Development (localhost)**: `admin+dev@yourapp.com`
    - **Beta/Staging**: `admin+beta@yourapp.com`
    - **Production**: `admin@yourapp.com`

    All emails go to the same inbox, but you can easily identify which environment they're from.

**Troubleshooting:**

- If you see "Organization 'System Administration' not found", run database migrations first
- If the user already exists, the script will notify you

!!! tip "Email Verification in Development"

    In development mode, the email service defaults to `stub` (no actual emails sent). When registering new users, look for `[STUB EMAIL]` blocks in the backend logs - they contain the verification link you can copy/paste to complete registration.

### 4. Start Your Application

After creating your admin user, start the services:

```bash
# Terminal 1: Start the backend
cd backend
uv run uvicorn app.main:app --reload
```

```bash
# Terminal 2: Start the frontend
cd frontend
npm run dev
```

```bash
# Terminal 3 (optional): Start the landing page
cd landing
npm run dev
```

### 5. Access Your Application

- **Frontend Dashboard**: [http://localhost:5173](http://localhost:5173)
- **Backend API**: [http://localhost:8000](http://localhost:8000)
- **Landing Page**: [http://localhost:5174](http://localhost:5174)

Navigate to [http://localhost:5173/login](http://localhost:5173/login) and login with the admin credentials you created in **Step 3**.

#### Testing User Registration (B2C Mode Only)

If you're running in **B2C mode**, you can test the public registration flow:

1. **Access the signup page**: Navigate to [http://localhost:5173/signup](http://localhost:5173/signup)
2. **Create a new account**: Fill in email, password, first name, and last name
3. **Check backend logs for verification link**: Since email service defaults to `stub` in development, look for `[STUB EMAIL]` blocks in your backend terminal
4. **Copy the verification link**: Find the URL in the email verification message
5. **Verify your email**: Paste the verification link in your browser to complete registration
6. **Log in**: Use your new credentials at [http://localhost:5173/login](http://localhost:5173/login)

!!! info "B2B Mode Registration"

    In **B2B mode**, the `/signup` page returns 404. Users can only join via invitation from a system administrator or organization admin. See the [B2B Mode Guide](b2b-mode.md) for details on the invitation workflow.

### 6. Explore the Application

After logging in, you can:

- **User management** - View and manage users
- **Organization settings** - Configure your organization
- **System analytics** - View usage statistics
- **AI Note Improver** - Test the demo application with note creation and AI-powered improvements

!!! info "Explore the API"

    Visit [http://localhost:8000/docs](http://localhost:8000/docs) for interactive API documentation.

## Setting Up for Your Team

Once your initial setup is working, getting teammates onboarded is straightforward — **only the first developer runs `init.py`**. Everyone else starts from your committed, pre-configured repository.

### 1. Push to Your Own Repository

Create a new repository on GitHub (or your preferred host), then push:

```bash
# Add your own repository as the remote
git remote add origin git@github.com:your-org/your-project.git
git push -u origin main
```

!!! tip

    You can safely delete `init.py` before your first commit — it's only needed for initial setup. The same applies to the `/docs` folder.

### 2. Team Members Clone and Configure

Other developers should:

1. **Clone your repository** — not the FastSvelte template
2. **Install prerequisites** — `uv`, Node.js 22+, Sqitch, Docker
3. **Create their `.env` files** — copy the `.env.example` files in `backend/`, `frontend/`, `landing/`, and `db/` and fill in the values (share secrets securely, never via git)
4. **Start the database** — `docker compose -f backend/docker-compose.yml up db -d` from the project root
5. **Run database migrations** — `cd db && ./sqitch.sh dev deploy`
6. **Install dependencies** — `uv sync` in `backend/`, `npm install` in `frontend/` and `landing/`
7. **Create a local admin user** — `cd backend && uv run scripts/create_admin.py`

!!! warning "Never commit `.env` files"

    `.env` files contain secrets and are excluded from git. Share environment variable values with teammates through a secure channel (1Password, Doppler, a shared secrets manager, etc.).

## Environment Variables

The backend environment variables are in `backend/.env` using the `FS_` prefix. These are loaded using [Pydantic Settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/). Here are the core variables you should be aware of:

- `FS_APP_NAME`: Your application name
- `FS_MODE`: b2c or b2b (see [B2B Mode](b2b-mode.md) for details)
- `FS_ENVIRONMENT`: dev, beta, or prod
- `FS_DB_URL` / `FS_DB_SCHEMA`: Database connection
- `FS_BASE_WEB_URL` / `FS_BASE_API_URL`: Frontend and backend URLs
- `FS_JWT_SECRET_KEY`: OAuth state validation (auto-generated)
- `FS_CRON_SECRET`: Cron job authentication (auto-generated)

Additional `.env` files exist in `frontend/`, `landing/`, and `db/` directories for their respective configurations.

!!! success "Automatic Configuration"

    The `init.py` script automatically generates all `.env` files with core variables. You typically only need to manually configure environment variables when adding integrations (email, payments, OAuth).

See the `.env.example` files in each directory for a full list of available environment variables.

See the **[Integrations Guide](integrations/index.md)** for instructions on configuring email services, payment gateways, and OAuth providers.

## Next Steps

Now that you have FastSvelte running:

- **[Development Guide](development.md)** - Learn common development workflows
- **[Architecture Overview](architecture-overview.md)** - Understand how FastSvelte is built
- **[Tutorials](tutorials.md)** - Step-by-step guides for adding features
- **[Integrations](integrations/index.md)** - Configure email, payments, and OAuth
- **[Deployment](deployment/index.md)** - Deploy to production
- **[Troubleshooting](troubleshooting.md)** - Common issues and solutions
