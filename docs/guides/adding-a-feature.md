---
description: "FastSvelte tutorials - Step-by-step guides for adding new entities, integrating APIs, implementing features, and extending the FastAPI backend and SvelteKit frontend."
keywords: "fastsvelte tutorials, fastapi tutorial, sveltekit tutorial, add new entity, database migration, api integration, fullstack development"
---

# Tutorials

This section provides step-by-step guides for extending [FastSvelte](https://fastsvelte.dev) with new functionality.

## Adding a New Entity (End-to-End)

This tutorial walks through adding a complete "Projects" feature to demonstrate how to extend every layer of the FastSvelte stack. You'll learn to add database tables, backend APIs, and frontend interfaces.

### Overview

We'll build a Projects feature that allows users to:

- Create, read, update, and delete projects
- Associate projects with the current organization
- Display projects in a list and detail view

### Step 1: Database Schema

First, create a new migration for the projects table.

```bash
cd backend/db
sqitch add projects -n "Add projects table"
```

Edit the generated migration file `backend/db/deploy/projects.sql`:

```sql
-- Deploy {schema_name}:projects to pg
-- Replace {schema_name} with your actual schema name (set during init.py)

BEGIN;

CREATE TABLE IF NOT EXISTS {schema_name}.project (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    organization_id INTEGER NOT NULL REFERENCES {schema_name}.organization(id) ON DELETE CASCADE,
    user_id INTEGER NOT NULL REFERENCES {schema_name}."user"(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_project_organization ON {schema_name}.project(organization_id);
CREATE INDEX IF NOT EXISTS idx_project_user ON {schema_name}.project(user_id);

COMMIT;
```

Create the revert script `backend/db/revert/projects.sql`:

```sql
-- Revert {schema_name}:projects from pg

BEGIN;

DROP INDEX IF EXISTS {schema_name}.idx_project_user;
DROP INDEX IF EXISTS {schema_name}.idx_project_organization;
DROP TABLE IF EXISTS {schema_name}.project;

COMMIT;
```

Apply the migration:

```bash
./sqitch.sh dev deploy
```

### Step 2: Backend Models

Create the Pydantic models in `backend/app/model/project.py`:

```python
from datetime import datetime
from typing import Optional
from pydantic import BaseModel


class ProjectEntity(BaseModel):
    id: int
    name: str
    description: str | None = None
    organization_id: int
    user_id: int
    created_at: datetime
    updated_at: datetime


class CreateProjectRequest(BaseModel):
    name: str
    description: str | None = None


class UpdateProjectRequest(BaseModel):
    name: str | None = None
    description: str | None = None


class ProjectResponse(BaseModel):
    id: int
    name: str
    description: str | None = None
    created_at: datetime
    updated_at: datetime
```

### Step 3: Repository Layer

Create `backend/app/data/repo/project_repo.py`:

```python
from app.data.repo.base_repo import BaseRepo
from app.model.project import ProjectEntity


class ProjectRepo(BaseRepo):
    async def create_project(
        self,
        name: str,
        description: str | None,
        user_id: int,
        organization_id: int
    ) -> ProjectEntity:
        query = f"""
            INSERT INTO {self.schema}.project (name, description, user_id, organization_id)
            VALUES ($1, $2, $3, $4)
            RETURNING id, name, description, organization_id, user_id, created_at, updated_at
        """
        row = await self.fetch_one(query, name, description, user_id, organization_id)
        return ProjectEntity(**row)

    async def get_project_by_id(self, project_id: int, organization_id: int) -> ProjectEntity | None:
        query = f"""
            SELECT id, name, description, organization_id, user_id, created_at, updated_at
            FROM {self.schema}.project
            WHERE id = $1 AND organization_id = $2
        """
        row = await self.fetch_one(query, project_id, organization_id)
        return ProjectEntity(**row) if row else None

    async def get_projects_by_organization(self, organization_id: int) -> list[ProjectEntity]:
        query = f"""
            SELECT id, name, description, organization_id, user_id, created_at, updated_at
            FROM {self.schema}.project
            WHERE organization_id = $1
            ORDER BY created_at DESC
        """
        rows = await self.fetch_all(query, organization_id)
        return [ProjectEntity(**row) for row in rows]

    async def update_project(
        self,
        project_id: int,
        name: str | None,
        description: str | None,
        organization_id: int
    ) -> ProjectEntity | None:
        query = f"""
            UPDATE {self.schema}.project
            SET name = COALESCE($3, name),
                description = COALESCE($4, description),
                updated_at = now()
            WHERE id = $1 AND organization_id = $2
            RETURNING id, name, description, organization_id, user_id, created_at, updated_at
        """
        row = await self.fetch_one(query, project_id, organization_id, name, description)
        return ProjectEntity(**row) if row else None

    async def delete_project(self, project_id: int, organization_id: int) -> None:
        query = f"DELETE FROM {self.schema}.project WHERE id = $1 AND organization_id = $2"
        await self.execute(query, project_id, organization_id)
```

### Step 4: Service Layer

Create `backend/app/service/project_service.py`:

```python
from app.data.repo.project_repo import ProjectRepo
from app.model.project import CreateProjectRequest, ProjectEntity, UpdateProjectRequest


class ProjectService:
    def __init__(self, project_repo: ProjectRepo):
        self.project_repo = project_repo

    async def create_project(
        self,
        organization_id: int,
        user_id: int,
        data: CreateProjectRequest
    ) -> ProjectEntity:
        return await self.project_repo.create_project(
            data.name, data.description, user_id, organization_id
        )

    async def list_projects(self, organization_id: int) -> list[ProjectEntity]:
        return await self.project_repo.get_projects_by_organization(organization_id)

    async def get_project(
        self,
        project_id: int,
        organization_id: int
    ) -> ProjectEntity | None:
        return await self.project_repo.get_project_by_id(project_id, organization_id)

    async def update_project(
        self,
        project_id: int,
        organization_id: int,
        data: UpdateProjectRequest
    ) -> ProjectEntity | None:
        return await self.project_repo.update_project(
            project_id, data.name, data.description, organization_id
        )

    async def delete_project(self, project_id: int, organization_id: int) -> None:
        await self.project_repo.delete_project(project_id, organization_id)
```

### Step 5: API Routes

Create `backend/app/api/route/project_route.py`:

```python
from app.api.middleware.auth_handler import min_role_required
from app.config.container import Container
from app.exception.common_exception import ResourceNotFound
from app.model.project import CreateProjectRequest, ProjectResponse, UpdateProjectRequest
from app.model.role_model import Role
from app.model.user_model import CurrentUser
from app.service.project_service import ProjectService
from dependency_injector.wiring import Provide, inject
from fastapi import APIRouter, Depends

router = APIRouter()


@router.post("", response_model=ProjectResponse, operation_id="createProject")
@inject
async def create_project(
    data: CreateProjectRequest,
    user: CurrentUser = Depends(min_role_required(Role.MEMBER)),
    project_service: ProjectService = Depends(Provide[Container.project_service]),
):
    project = await project_service.create_project(
        user.organization_id, user.id, data
    )
    return ProjectResponse.model_validate(project.model_dump())


@router.get("", response_model=list[ProjectResponse], operation_id="listProjects")
@inject
async def list_projects(
    user: CurrentUser = Depends(min_role_required(Role.MEMBER)),
    project_service: ProjectService = Depends(Provide[Container.project_service]),
):
    projects = await project_service.list_projects(user.organization_id)
    return [ProjectResponse.model_validate(p.model_dump()) for p in projects]


@router.get("/{project_id}", response_model=ProjectResponse, operation_id="getProject")
@inject
async def get_project(
    project_id: int,
    user: CurrentUser = Depends(min_role_required(Role.MEMBER)),
    project_service: ProjectService = Depends(Provide[Container.project_service]),
):
    project = await project_service.get_project(project_id, user.organization_id)
    if not project:
        raise ResourceNotFound("project", project_id)
    return ProjectResponse.model_validate(project.model_dump())


@router.put("/{project_id}", response_model=ProjectResponse, operation_id="updateProject")
@inject
async def update_project(
    project_id: int,
    data: UpdateProjectRequest,
    user: CurrentUser = Depends(min_role_required(Role.MEMBER)),
    project_service: ProjectService = Depends(Provide[Container.project_service]),
):
    project = await project_service.update_project(
        project_id, user.organization_id, data
    )
    if not project:
        raise ResourceNotFound("project", project_id)
    return ProjectResponse.model_validate(project.model_dump())


@router.delete("/{project_id}", status_code=204, operation_id="deleteProject")
@inject
async def delete_project(
    project_id: int,
    user: CurrentUser = Depends(min_role_required(Role.MEMBER)),
    project_service: ProjectService = Depends(Provide[Container.project_service]),
):
    project = await project_service.get_project(project_id, user.organization_id)
    if not project:
        raise ResourceNotFound("project", project_id)
    await project_service.delete_project(project_id, user.organization_id)
```

### Step 6: Dependency Injection Setup

Update `backend/app/config/container.py` to include the new components:

```python
# 1. Add to imports at the top of the file
from app.data.repo.project_repo import ProjectRepo
from app.service.project_service import ProjectService

# 2. Add to the Container class in the repositories section (around line 47)
project_repo = providers.Singleton(ProjectRepo, db_config=db_config)

# 3. Add to the Container class in the services section (around line 157)
project_service = providers.Factory(
    ProjectService,
    project_repo=project_repo,
)
```

### Step 7: Register Routes

Update `backend/app/api/router.py` to include the new routes:

```python
# 1. Add to imports at the top
from app.api.route.project_route import router as project_router

# 2. Add to include_all_routers() function (around line 31)
app.include_router(project_router, prefix="/projects", tags=["Projects"])
```

### Step 8: Test the Backend API

Before building the frontend, test your new API endpoints to ensure they work correctly.

**1. Start the backend server:**

```bash
cd backend
uv run uvicorn app.main:app --reload
```

**2. Create a test file** at `backend/http/14_projects.http`:

```http
### FastSvelte Projects API Tests
### Login first using 01_auth.http - cookies persist across all files
### Variables loaded from .env file in this directory

@baseUrl = {{$dotenv BASE_URL}}


### ============================================
### CREATE PROJECT
### ============================================

### Create Project
POST {{baseUrl}}/projects
Content-Type: application/json

{
  "name": "My First Project",
  "description": "A sample project for testing"
}

###


### ============================================
### LIST & GET PROJECTS
### ============================================

### List All Projects
GET {{baseUrl}}/projects

###

### Get Specific Project
# @prompt projectId Project ID to retrieve
GET {{baseUrl}}/projects/{{projectId}}


### ============================================
### UPDATE PROJECT
### ============================================

### Update Project
# @prompt projectId Project ID to update
PUT {{baseUrl}}/projects/{{projectId}}
Content-Type: application/json

{
  "name": "Updated Project Name",
  "description": "Updated description"
}


### ============================================
### DELETE PROJECT
### ============================================

### Delete Project
# @prompt projectId Project ID to delete
DELETE {{baseUrl}}/projects/{{projectId}}


### ============================================
### TEST ERROR CASES
### ============================================

### Get non-existent project (should return 404)
GET {{baseUrl}}/projects/99999

###

### Update non-existent project (should return 404)
PUT {{baseUrl}}/projects/99999
Content-Type: application/json

{
  "name": "Updated",
  "description": "Updated description"
}

###

### Delete non-existent project (should return 404)
DELETE {{baseUrl}}/projects/99999
```

**3. Authenticate first:**

Before testing the Projects API, you need to log in to get an authenticated session.

If you're using VSCode with the [REST Client extension](https://marketplace.visualstudio.com/items?itemName=humao.rest-client):

1. Open `backend/http/01_auth.http`
2. Run the "Register" or "Login" request to authenticate
3. The session cookie will persist across all `.http` files
4. Now open `backend/http/14_projects.http` and run the requests

**4. Test your API:**

Click "Send Request" above each test in the `14_projects.http` file to test all CRUD operations.

**Alternative: Test with curl:**

```bash
# First, login and save the session cookie
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "your-email@example.com", "password": "your-password"}' \
  -c cookies.txt

# Create a project (using saved cookies)
curl -X POST http://localhost:8000/api/projects \
  -H "Content-Type: application/json" \
  -d '{"name": "My First Project", "description": "A sample project"}' \
  -b cookies.txt

# List all projects
curl http://localhost:8000/api/projects -b cookies.txt

# Get a specific project (replace 1 with actual ID)
curl http://localhost:8000/api/projects/1 -b cookies.txt

# Update a project (replace 1 with actual ID)
curl -X PUT http://localhost:8000/api/projects/1 \
  -H "Content-Type: application/json" \
  -d '{"name": "Updated Name", "description": "Updated description"}' \
  -b cookies.txt

# Delete a project (replace 1 with actual ID)
curl -X DELETE http://localhost:8000/api/projects/1 -b cookies.txt
```

!!! tip "View API Documentation"

    Visit `http://localhost:8000/docs` to see the auto-generated OpenAPI documentation for all your endpoints, including the new Projects API.

### Step 9: Generate Frontend API Client

After adding the backend routes, regenerate the TypeScript API client:

```bash
cd frontend
npm run generate
```

This creates the API functions in `src/lib/api/gen/` that you'll use in the frontend.

### Step 10: Build Frontend Components

Create the projects list page at `frontend/src/routes/(protected)/projects/+page.svelte`:

```html
<script lang="ts">
  import { onMount } from "svelte";
  import { goto } from "$app/navigation";
  import { listProjects, deleteProject } from "$lib/api/gen/projects";
  import type { ProjectResponse } from "$lib/api/gen/model";

  let projects = $state<ProjectResponse[]>([]);
  let loading = $state(true);
  let error = $state<string | null>(null);

  onMount(loadProjects);

  async function loadProjects() {
    loading = true;
    try {
      projects = (await listProjects()) || [];
    } catch (err) {
      error = "Failed to load projects";
      console.error("Error loading projects:", err);
    } finally {
      loading = false;
    }
  }

  async function handleDelete(projectId: number) {
    if (!confirm("Are you sure you want to delete this project?")) return;

    try {
      await deleteProject(projectId);
      await loadProjects();
    } catch (err) {
      error = "Failed to delete project";
      console.error("Error deleting project:", err);
    }
  }
</script>

<div class="container mx-auto p-6">
  <div class="flex justify-between items-center mb-6">
    <h1 class="text-3xl font-bold">Projects</h1>
    <button class="btn btn-primary" onclick={() => goto('/projects/new')}>
      Create Project
    </button>
  </div>

  {#if loading}
  <div class="flex justify-center">
    <span class="loading loading-spinner loading-lg"></span>
  </div>
  {:else if error}
  <div class="alert alert-error">
    <span>{error}</span>
  </div>
  {:else if projects.length === 0}
  <div class="text-center py-8">
    <p class="text-gray-500 mb-4">No projects yet</p>
    <button class="btn btn-primary" onclick={() => goto('/projects/new')}>
      Create Your First Project
    </button>
  </div>
  {:else}
  <div class="grid gap-4">
    {#each projects as project}
    <div class="card bg-base-100 shadow-xl">
      <div class="card-body">
        <h2 class="card-title">{project.name}</h2>
        {#if project.description}
        <p>{project.description}</p>
        {/if}
        <p class="text-sm text-gray-500">
          Created: {new Date(project.created_at).toLocaleDateString()}
        </p>
        <div class="card-actions justify-end">
          <button class="btn btn-outline btn-sm" onclick={() => goto(`/projects/${project.id}`)}>
            View
          </button>
          <button class="btn btn-outline btn-sm" onclick={() => goto(`/projects/${project.id}/edit`)}>
            Edit
          </button>
          <button class="btn btn-error btn-outline btn-sm" onclick={() => handleDelete(project.id)}>
            Delete
          </button>
        </div>
      </div>
    </div>
    {/each}
  </div>
  {/if}
</div>
```

Create the new project form at `frontend/src/routes/(protected)/projects/new/+page.svelte`:

```html
<script lang="ts">
  import { goto } from "$app/navigation";
  import { createProject } from "$lib/api/gen/projects";
  import type { CreateProjectRequest } from "$lib/api/gen/model";

  let form = $state<CreateProjectRequest>({
    name: "",
    description: "",
  });
  let loading = $state(false);
  let error = $state<string | null>(null);

  async function handleSubmit() {
    if (!form.name.trim()) {
      error = "Project name is required";
      return;
    }

    loading = true;
    error = null;

    try {
      await createProject(form);
      goto("/projects");
    } catch (err) {
      error = "Failed to create project";
      console.error("Error creating project:", err);
    } finally {
      loading = false;
    }
  }
</script>

<div class="container mx-auto p-6 max-w-md">
  <h1 class="text-3xl font-bold mb-6">Create New Project</h1>

  {#if error}
  <div class="alert alert-error mb-4">
    <span>{error}</span>
  </div>
  {/if}

  <form onsubmit={(e) => { e.preventDefault(); handleSubmit(); }} class="space-y-4">
    <div class="form-control">
      <label class="label" for="name">
        <span class="label-text">Project Name *</span>
      </label>
      <input
        id="name"
        type="text"
        class="input input-bordered"
        bind:value={form.name}
        required
        disabled={loading}
      />
    </div>

    <div class="form-control">
      <label class="label" for="description">
        <span class="label-text">Description</span>
      </label>
      <textarea
        id="description"
        class="textarea textarea-bordered"
        rows="3"
        bind:value={form.description}
        disabled={loading}
      ></textarea>
    </div>

    <div class="flex gap-2">
      <button
        type="submit"
        class="btn btn-primary flex-1"
        class:loading
        disabled={loading}
      >
        {loading ? 'Creating...' : 'Create Project'}
      </button>
      <button type="button" class="btn btn-outline" onclick={() => goto('/projects')} disabled={loading}>
        Cancel
      </button>
    </div>
  </form>
</div>
```

### Step 11: Add Navigation

Update your main navigation to include the projects link. In your sidebar component, add:

```html
<li>
  <a href="/projects" class="flex items-center gap-2">
    <span class="iconify lucide--folder"></span>
    Projects
  </a>
</li>
```

### Step 12: Test the Complete Feature

**1. Start the database** (if not already running):

```bash
docker compose up db -d
```

**2. Start the backend**:

```bash
cd backend
uv run uvicorn app.main:app --reload
```

**3. Start the frontend** (in a new terminal):

```bash
cd frontend
npm run dev
```

**4. Navigate to projects**: Visit `http://localhost:5173/projects`

**5. Test CRUD operations**:

- Create a new project
- View the projects list
- Edit existing projects
- Delete projects

### Summary

You've successfully added a complete "Projects" feature that demonstrates:

- **Database design**: Migration with proper foreign keys and constraints
- **Backend architecture**: Repository pattern, service layer, API routes
- **Dependency injection**: Proper container configuration
- **API design**: RESTful endpoints with proper HTTP status codes
- **Frontend integration**: API client generation and UI components
- **Security**: Organization-based data isolation and authentication

This pattern can be applied to add any new entity to your FastSvelte application. The key principles are:

1. Start with the database schema
2. Build from the data layer up (repository → service → routes)
3. Configure dependency injection
4. Generate and use the API client on the frontend
5. Build UI components following the existing design patterns
