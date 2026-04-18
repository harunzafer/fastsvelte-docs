---
description: "FastSvelte troubleshooting guide - Solutions for common setup, database, authentication, and deployment issues with FastAPI, SvelteKit, PostgreSQL, and Docker."
keywords: "fastsvelte troubleshooting, fastapi errors, sveltekit issues, postgresql problems, docker debugging, database migration errors, authentication issues, cors errors, deployment problems"
---

# Troubleshooting

Common issues and solutions for [FastSvelte](https://fastsvelte.dev) development and deployment.

## Setup Issues

**Init script prerequisites not met**

```bash
✗ Python 3.12 not found
```

**Solution:** Install all required tools:

- **Python 3.12+**: https://www.python.org/downloads/
- **Docker**: https://docs.docker.com/get-docker/
- **Node.js 22+**: https://nodejs.org/
- **Sqitch**: https://sqitch.org/download/

**Init script permission denied**

```bash
bash: ./init.py: Permission denied
```

**Solution:** Make the script executable:

```bash
chmod +x init.py
chmod +x db/sqitch.sh
```

**Port conflicts during init**

```bash
Error: Port 5432 already in use
```

**Solution:** Stop services using required ports (5432, 8000, 5173, 5174):

```bash
# Find what's using the port
lsof -i :5432

# Stop the service or change port in docker-compose.yml
```

**Docker daemon not running**

```bash
ERROR: Cannot connect to Docker daemon
```

**Solution:** Start Docker:

```bash
# macOS/Windows: Start Docker Desktop
# Linux:
sudo systemctl start docker
```

**pip install fails with dependency conflicts**

```bash
ERROR: Cannot install fastapi>=0.104.0 and pydantic<2.0.0
```

**Solution:** Update Python to 3.12+ and use clean virtual environment:

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

**PostgreSQL connection refused**

```bash
asyncpg.exceptions.ConnectionRefusedError: Connection refused
```

**Solution:** Ensure PostgreSQL is running:

```bash
# Using Docker
docker compose up db -d

# Check if running
docker ps | grep postgres

# Verify connection
psql postgres://postgres:postgres@localhost/fastsvelte -c "SELECT 1;"
```

**Database does not exist**

```bash
asyncpg.exceptions.InvalidCatalogNameError: database "fastsvelte" does not exist
```

**Solution:** Create the database:

```bash
# Connect to PostgreSQL and create database
psql postgres://postgres:postgres@localhost -c "CREATE DATABASE fastsvelte;"

# Or use Docker
docker exec -it fastsvelte-db psql -U postgres -c "CREATE DATABASE fastsvelte;"
```

---

## Development Issues

**API client generation fails**

```bash
Error: Could not fetch OpenAPI spec from http://localhost:8000/openapi.json
```

**Solution:** Ensure backend is running before generating client:

```bash
# Start backend first
cd backend && uvicorn app.main:app --reload

# Then generate in another terminal
cd frontend && npm run generate
```

**Import errors with absolute imports**

```bash
ModuleNotFoundError: No module named 'app.service'
```

**Solution:** Always use absolute imports from `app` package:

```python
# ✅ Correct
from app.service.user_service import UserService

# ❌ Incorrect
from ..service.user_service import UserService
```

**Dependency injection not working**

```bash
TypeError: 'NoneType' object is not callable
```

**Solution:** Ensure your module is added to wiring configuration:

```python
# app/config/container.py
wiring_config = containers.WiringConfiguration(
    modules=[
        "app.api.route.your_new_route",  # Add this line
        # ... existing modules
    ]
)
```

**Hot reload not working**

```bash
Changes not reflected in browser
```

**Solution:** Check file watchers and ports:

```bash
# Backend
uv run uvicorn app.main:app --reload

# Frontend
npm run dev -- --host 0.0.0.0 --port 5173
```

---

## Database Issues

**Migration fails with "relation already exists"**

```bash
psycopg2.errors.DuplicateTable: relation "user" already exists
```

**Solution:** Use `IF NOT EXISTS` in migrations:

```sql
CREATE TABLE IF NOT EXISTS fastsvelte."user" (...);
```

**Sqitch deploy fails with permission denied**

```bash
bash: ./sqitch.sh: Permission denied
```

**Solution:** Make script executable:

```bash
chmod +x db/sqitch.sh
```

**Connection pool exhausted**

```bash
asyncpg.exceptions.TooManyConnectionsError: too many connections
```

**Solution:** Adjust connection pool settings:

```python
# app/data/db_config.py
self.pool = await asyncpg.create_pool(
    self.dsn,
    min_size=5,    # Reduce if needed
    max_size=10,   # Reduce if needed
    command_timeout=60
)
```

**Docker volumes not mounting**

```bash
Database data lost after container restart
```

**Solution:** Create external volume:

```bash
docker volume create fastsvelte-data
docker compose up db -d
```

---

## Frontend Issues

**CORS errors in development**

```bash
Access to fetch blocked by CORS policy
```

**Solution:** Check CORS configuration in backend:

```python
# app/config/settings.py
@property
def cors_origins(self) -> list[str]:
    return {
        "dev": ["http://localhost:5173", "http://localhost:4173"],
        # ... other environments
    }.get(self.environment, [])
```

**Authentication not persisting**

```bash
User logged out after page refresh
```

**Solution:** Ensure cookies are configured properly:

```javascript
// src/lib/api/axios.js
export const axiosInstance = Axios.create({
  baseURL: PUBLIC_API_BASE_URL,
  withCredentials: true, // This is crucial
});
```

**Svelte components not updating**

```bash
Component state not reactive
```

**Solution:** Use Svelte 5 runes correctly:

```typescript
// ✅ Correct
let count = $state(0);

// ❌ Incorrect
let count = 0;
```

---

## Production Issues

**Static assets not loading**

```bash
404 Not Found for /assets/app.js
```

**Solution:** Check build configuration and base path:

```javascript
// frontend/vite.config.js
export default {
  build: {
    outDir: "build",
    assetsDir: "assets",
  },
};
```

**Database connection timeouts in production**

```bash
asyncpg.exceptions.ServerTimeoutError: timeout
```

**Solution:** Increase connection timeout and pool settings:

```bash
# Environment variables
FS_DB_URL="postgres://user:pass@host/db?connect_timeout=60"
```

**High memory usage**

```bash
Container killed: Out of memory
```

**Solution:** Optimize container resources and add memory limits:

```dockerfile
# Dockerfile
FROM python:3.12-slim  # Use slim image

# Add memory-efficient settings
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1
```

**SSL certificate errors**

```bash
SSL_CERT_VERIFY_FAILED
```

**Solution:** Ensure proper SSL configuration:

```bash
# Check certificate
openssl s_client -connect yourdomain.com:443

# Verify DNS
nslookup yourdomain.com
```

---

## Performance Issues

**Slow database queries**

```bash
Query execution time > 1000ms
```

**Solution:** Add indexes and optimize queries:

```sql
-- Find slow queries
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
WHERE mean_exec_time > 100
ORDER BY mean_exec_time DESC;

-- Add missing indexes
CREATE INDEX CONCURRENTLY idx_user_organization_active
ON fastsvelte."user"(organization_id) WHERE is_active = true;
```

**Memory usage constantly increasing**

```bash
Memory usage: 85%+
```

**Solution:** Check for memory leaks:

```python
# Add connection cleanup
async def cleanup_connections():
    await db_config.pool.close()

# Monitor connection pools
async def get_pool_status():
    return {
        "size": db_config.pool.get_size(),
        "active": db_config.pool.get_active_count(),
        "idle": db_config.pool.get_idle_count()
    }
```

---

## Security Issues

**Users accessing protected routes without login**

```bash
Unauthorized access to /admin
```

**Solution:** Verify route protection:

```python
@router.get("/admin/users")
async def list_users(
    current_user: CurrentUser = Depends(min_role_required(Role.SYSTEM_ADMIN))
):
    # Ensure dependency is applied
```

**Sessions being stolen or reused**
**Solution:** Implement proper session security:

```python
# Rotate session tokens on login
async def login(email: str, password: str):
    # ... authenticate user

    # Generate new session token
    new_token = secrets.token_urlsafe(32)

    # Invalidate old sessions for this user
    await session_repo.delete_by_user_id(user.id)

    # Create new session
    await session_repo.create(user.id, hash_token(new_token))
```

---

## Monitoring & Debugging

**View backend logs:**

```bash
# Docker logs
docker logs fastsvelte-api --follow

# Local development
tail -f backend/app.log
```

**View frontend logs:**

- Open browser DevTools → Console
- Check Network tab for API errors
- Monitor Application tab for localStorage/cookies

**Check database connections:**

```sql
SELECT count(*) as active_connections
FROM pg_stat_activity
WHERE state = 'active';
```

**API health check:**

```bash
curl https://yourdomain.com/health
```

---

## Frequently Asked Questions

**Why doesn't FastSvelte use `__init__.py` files?**

FastSvelte is an **application, not a library**. While `__init__.py` has benefits for library code, they don't apply here:

**Benefits of `__init__.py` (that don't apply to FastSvelte):**

- **Shorter imports** (`from app.service import X`) - We prefer explicit paths that show exactly where code lives
- **Public API facade** - Applications don't need API contracts; our layers (controller/service/repo) are already the boundaries
- **Hiding internal structure** - Not needed when the team owns the full codebase
- **Relative imports** - We deliberately use absolute imports (`from app.service.x`) everywhere, avoiding relative imports entirely

**Problems avoided by not using `__init__.py`:**

- **No circular import traps** - Re-exports often create hidden import cycles
- **No hidden side effects** - Importing a package won't unexpectedly initialize clients or load config
- **Simpler refactoring** - Moving files is mechanical; no export lists to maintain
- **Explicit dependencies** - `from app.service.rule_service import RuleService` shows the exact file and ownership
- **Less decision fatigue** - No need to decide what to export from each package

**Note:** Some tools like `fastapi dev` expect `__init__.py` for package detection. Use `uvicorn app.main:app --reload` instead, which works without them and is what production uses anyway.

---

## Getting Help

When reporting issues, include:

1. **Environment details** - Development/production, OS, versions
2. **Error messages** - Complete error traces and logs
3. **Configuration** - Relevant environment variables (sanitized)
4. **Steps to reproduce** - Minimal reproduction steps
5. **Expected behavior** - What should happen vs what actually happens

**Resources:**

- [Architecture Overview](architecture-overview.md) - Understanding the system
- [Development Guide](development.md) - Development workflows
- [Integrations](integrations/index.md) - External service configuration
