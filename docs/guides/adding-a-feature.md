---
description: "Add a complete feature to FastSvelte end to end: migration, repository, service, routes, dependency injection, generated API client, and a SvelteKit page built on the kit's load-function pattern."
keywords: "fastsvelte tutorial, fastapi add entity, sveltekit crud, database migration, dependency injection, orval client, add feature fullstack"
---

# Adding a Feature

This guide adds a complete **Projects** feature: a database table, a CRUD API, and a page in the app. Every step mirrors the shipped **Notes** feature, so you can open the real file next to each snippet and see the same pattern with its production trimmings. Work in **DB → backend → frontend** order so the generated API client stays in sync (see [Development Workflow](development-workflow.md)).

!!! note "About the schema name"

    The SQL examples use `myapp` as the schema. Use your own: the value of `FS_DB_SCHEMA` in `backend/.env`, chosen when you ran `init.py`.

## 1. Create the migration

```bash
cd backend/db
./sqitch.sh add add_project -n "Add project table"
```

This creates three files: `deploy/add_project.sql` (how to apply the change), `revert/add_project.sql` (how to undo it), and `verify/add_project.sql` (how to prove it worked).

Fill in `deploy/add_project.sql` below the generated header:

```sql
BEGIN;

CREATE TABLE myapp.project (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    organization_id INTEGER NOT NULL REFERENCES myapp.organization(id) ON DELETE CASCADE,
    user_id INTEGER NOT NULL REFERENCES myapp."user"(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_project_organization ON myapp.project(organization_id);

COMMIT;
```

`organization_id` is what keeps tenants isolated: every query in the repository will filter on it. The index backs those queries.

`revert/add_project.sql`:

```sql
BEGIN;

DROP TABLE IF EXISTS myapp.project;

COMMIT;
```

`verify/add_project.sql` proves the table exists with the expected columns, the same way the kit's own verify scripts do:

```sql
BEGIN;

SELECT id, name, description, organization_id, user_id, created_at, updated_at
FROM myapp.project
WHERE false;

ROLLBACK;
```

Deploy it:

```bash
./sqitch.sh dev deploy
```

You will see the change apply:

```
  + add_project .. ok
```

## 2. Define the models

Create `backend/app/model/project_model.py`. Model files end in `_model.py`, and timestamped entities extend the shared `Timestamped` base instead of redeclaring the columns:

```python
from pydantic import BaseModel

from app.model.common import Timestamped


class Project(Timestamped):
    id: int
    organization_id: int
    user_id: int
    name: str
    description: str | None = None


class CreateProjectRequest(BaseModel):
    name: str
    description: str | None = None


class UpdateProjectRequest(BaseModel):
    name: str | None = None
    description: str | None = None


class ProjectResponse(Timestamped):
    id: int
    name: str
    description: str | None = None
```

`Project` is the internal entity, one to one with a table row. `ProjectResponse` is what the API returns; it leaves out `organization_id` and `user_id` because the caller's session already determines both.

## 3. Add the repository

Create `backend/app/data/repo/project_repo.py`. Repositories are the only layer that touches SQL. `self.schema` comes from `BaseRepo` (it reads `FS_DB_SCHEMA`), so the code itself stays schema-agnostic:

```python
from app.data.repo.base_repo import BaseRepo
from app.model.project_model import Project


class ProjectRepo(BaseRepo):
    async def create_project(
        self, organization_id: int, user_id: int, name: str, description: str | None
    ) -> Project:
        query = f"""
            INSERT INTO {self.schema}.project (organization_id, user_id, name, description)
            VALUES ($1, $2, $3, $4)
            RETURNING id, organization_id, user_id, name, description, created_at, updated_at
        """
        row = await self.fetch_one(query, organization_id, user_id, name, description)
        return Project(**row)

    async def get_project_by_id(self, project_id: int, organization_id: int) -> Project | None:
        query = f"""
            SELECT id, organization_id, user_id, name, description, created_at, updated_at
            FROM {self.schema}.project
            WHERE id = $1 AND organization_id = $2
        """
        row = await self.fetch_one(query, project_id, organization_id)
        return Project(**row) if row else None

    async def list_projects(self, organization_id: int) -> list[Project]:
        query = f"""
            SELECT id, organization_id, user_id, name, description, created_at, updated_at
            FROM {self.schema}.project
            WHERE organization_id = $1
            ORDER BY created_at DESC
        """
        rows = await self.fetch_all(query, organization_id)
        return [Project(**row) for row in rows]

    async def update_project(
        self, project_id: int, organization_id: int, name: str | None, description: str | None
    ) -> Project | None:
        query = f"""
            UPDATE {self.schema}.project
            SET name = COALESCE($3, name),
                description = COALESCE($4, description),
                updated_at = now()
            WHERE id = $1 AND organization_id = $2
            RETURNING id, organization_id, user_id, name, description, created_at, updated_at
        """
        row = await self.fetch_one(query, project_id, organization_id, name, description)
        return Project(**row) if row else None

    async def delete_project(self, project_id: int, organization_id: int) -> None:
        query = f"DELETE FROM {self.schema}.project WHERE id = $1 AND organization_id = $2"
        await self.execute(query, project_id, organization_id)
```

Every read and write carries `organization_id` in the `WHERE` clause. A user can never reach another organization's rows, even with a guessed id, because the filter is part of the query itself rather than a check bolted on afterwards.

## 4. Add the service

Create `backend/app/service/project_service.py`. The service owns business logic; for plain CRUD it stays thin, and that's fine. It exists so that when rules arrive (quotas, notifications, cross-entity checks) they have a home that isn't a route handler:

```python
from app.data.repo.project_repo import ProjectRepo
from app.model.project_model import CreateProjectRequest, Project, UpdateProjectRequest


class ProjectService:
    def __init__(self, project_repo: ProjectRepo):
        self.project_repo = project_repo

    async def create_project(
        self, organization_id: int, user_id: int, data: CreateProjectRequest
    ) -> Project:
        return await self.project_repo.create_project(
            organization_id, user_id, data.name, data.description
        )

    async def list_projects(self, organization_id: int) -> list[Project]:
        return await self.project_repo.list_projects(organization_id)

    async def get_project(self, project_id: int, organization_id: int) -> Project | None:
        return await self.project_repo.get_project_by_id(project_id, organization_id)

    async def update_project(
        self, project_id: int, organization_id: int, data: UpdateProjectRequest
    ) -> Project | None:
        return await self.project_repo.update_project(
            project_id, organization_id, data.name, data.description
        )

    async def delete_project(self, project_id: int, organization_id: int) -> None:
        await self.project_repo.delete_project(project_id, organization_id)
```

## 5. Add the routes

Create `backend/app/api/route/project_route.py`:

```python
from dependency_injector.wiring import Provide, inject
from fastapi import APIRouter, Depends

from app.api.middleware.auth_handler import min_role_required
from app.config.container import Container
from app.exception.common_exception import ResourceNotFound
from app.model.project_model import CreateProjectRequest, ProjectResponse, UpdateProjectRequest
from app.model.role_model import Role
from app.model.user_model import CurrentUser
from app.service.project_service import ProjectService

router = APIRouter()


@router.post("", response_model=ProjectResponse, operation_id="createProject")
@inject
async def create_project(
    data: CreateProjectRequest,
    user: CurrentUser = Depends(min_role_required(Role.MEMBER)),
    project_service: ProjectService = Depends(Provide[Container.project_service]),
):
    project = await project_service.create_project(user.organization_id, user.id, data)
    return ProjectResponse.model_validate(project.model_dump())


@router.get("", response_model=list[ProjectResponse], operation_id="listProjects")
@inject
async def list_projects(
    user: CurrentUser = Depends(min_role_required(Role.READONLY)),
    project_service: ProjectService = Depends(Provide[Container.project_service]),
):
    projects = await project_service.list_projects(user.organization_id)
    return [ProjectResponse.model_validate(p.model_dump()) for p in projects]


@router.get("/{project_id}", response_model=ProjectResponse, operation_id="getProject")
@inject
async def get_project(
    project_id: int,
    user: CurrentUser = Depends(min_role_required(Role.READONLY)),
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
    project = await project_service.update_project(project_id, user.organization_id, data)
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

Three conventions to notice, all copied from the shipped `note_route.py`:

- **Reads take `Role.READONLY`, writes take `Role.MEMBER`.** A read-only user can browse but not change anything.
- **`operation_id` becomes the TypeScript function name** when the API client is generated (`listProjects()` in the frontend). See [Type-Safe API Client](orval.md).
- **Missing rows raise `ResourceNotFound`**, which the exception middleware turns into a clean 404. Services and repositories never touch HTTP status codes.

!!! tip "Want a per-plan limit on projects?"

    The notes feature caps creation with a plan quota (`max_notes`). To do the same for projects, follow [Metering a Custom Feature](metering-a-custom-feature.md).

## 6. Wire it into the container

Register the new classes in `backend/app/config/container.py`. There are **three** additions, and the third is the one that's easy to forget.

Add the imports:

```python
from app.data.repo.project_repo import ProjectRepo
from app.service.project_service import ProjectService
```

Add the providers, next to the existing repository and service registrations. Everything in the container is a `Singleton`: repositories and services hold no per-request state, so one instance serves the whole app:

```python
project_repo = providers.Singleton(ProjectRepo, db_config=db_config)

project_service = providers.Singleton(
    ProjectService,
    project_repo=project_repo,
)
```

Add the route module to the `wiring_config` list in the same file:

```python
    wiring_config = containers.WiringConfiguration(
        modules=[
            # ...existing modules...
            "app.api.route.project_route",
        ]
    )
```

!!! warning "Skipping the wiring entry breaks the routes at runtime"

    `@inject` only works in modules listed in `wiring_config`. Leave `app.api.route.project_route` out and the app still starts, but every `/projects` request fails, because `Provide[Container.project_service]` is never replaced with a real service.

## 7. Register the router

In `backend/app/api/router.py`, import the router and add it inside `include_all_routers()`:

```python
from app.api.route.project_route import router as project_router

app.include_router(project_router, prefix="/projects", tags=["Projects"])
```

The `Projects` tag groups the endpoints in the OpenAPI spec, and the generated client uses it to name the module (`projects.ts`).

## 8. Run it: test the API

Start the backend:

```bash
cd backend
uv run uvicorn app.main:app --reload
```

Open [localhost:8000/docs](http://localhost:8000/docs). You will see a **Projects** section with the five endpoints. If it's missing, revisit step 7; if the endpoints are there but return 500, revisit step 6.

The kit ships `.http` test files under `backend/http/` for the [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) VSCode extension. Add `backend/http/22_projects.http`:

```http
### Projects API
### Log in first via 01_auth.http; the session cookie persists across files.

@baseUrl = {{$dotenv BASE_URL}}

### Create project
POST {{baseUrl}}/projects
Content-Type: application/json

{
  "name": "My first project",
  "description": "Testing the new feature"
}

### List projects
GET {{baseUrl}}/projects

### Get one
# @prompt projectId
GET {{baseUrl}}/projects/{{projectId}}

### Update
# @prompt projectId
PUT {{baseUrl}}/projects/{{projectId}}
Content-Type: application/json

{
  "name": "Renamed project"
}

### Delete
# @prompt projectId
DELETE {{baseUrl}}/projects/{{projectId}}

### Missing project returns 404
GET {{baseUrl}}/projects/99999
```

Open `01_auth.http`, run the login request, then run "Create project". You will see the JSON response:

```json
{
  "id": 1,
  "name": "My first project",
  "description": "Testing the new feature",
  "created_at": "2026-08-01T12:00:00.000000+00:00",
  "updated_at": "2026-08-01T12:00:00.000000+00:00"
}
```

??? note "Testing with curl instead"

    Log in once and reuse the session cookie:

    ```bash
    curl -X POST http://localhost:8000/auth/login \
      -H "Content-Type: application/json" \
      -d '{"email": "you@example.com", "password": "your-password"}' \
      -c cookies.txt

    curl -X POST http://localhost:8000/projects \
      -H "Content-Type: application/json" \
      -d '{"name": "My first project"}' \
      -b cookies.txt

    curl http://localhost:8000/projects -b cookies.txt
    ```

## 9. Regenerate the API client

With the backend still running:

```bash
cd frontend
npm run generate
```

You will see a new `projects.ts` module in `frontend/src/lib/api/gen/` exporting `createProject`, `listProjects`, `getProject`, `updateProject`, and `deleteProject`, fully typed from the OpenAPI spec.

## 10. Add the invalidation key

The frontend refreshes data by re-running load functions, and load functions are matched by named scopes. Add a scope for projects in `frontend/src/lib/invalidation-keys.ts`:

```ts
export const KEYS = {
	// ...existing keys...
	/** The organization's projects. Claimed by the projects list. */
	projects: 'app:projects'
} as const;
```

## 11. Build the page

Data loading follows the kit's load-function pattern: the page's `+page.ts` fetches, the component renders. No `onMount`, no hand-rolled `loading` flags. The shipped notes page is the annotated reference for everything in this step; open `src/routes/(protected)/notes/+page.ts` and `+page.svelte` alongside it.

Create `frontend/src/routes/(protected)/projects/+page.ts`:

```ts
import { listProjects } from '$lib/api/gen/projects';
import { KEYS } from '$lib/invalidation-keys';
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ depends }) => {
	// Claims the scope that invalidate(KEYS.projects) matches.
	depends(KEYS.projects);

	// No await: the page receives a promise and renders a skeleton while it
	// resolves. Errors land in the {:catch} branch of the {#await} block.
	return {
		projects: listProjects().then((projects) => projects ?? [])
	};
};
```

Create `frontend/src/routes/(protected)/projects/+page.svelte`:

```svelte
<script lang="ts">
	import { goto, invalidate } from '$app/navigation';
	import { resolve } from '$app/paths';
	import { deleteProject } from '$lib/api/gen/projects';
	import { KEYS } from '$lib/invalidation-keys';
	import PageHeader from '$lib/components/PageHeader.svelte';
	import { toast } from '$lib/util/toast.svelte';

	let { data } = $props();

	// Nothing copies data.projects into local state, and there is no loading
	// flag: {#await} below renders all three states from the promise itself.
	async function handleDelete(id: number) {
		if (!confirm('Delete this project?')) return;
		try {
			await deleteProject(id);
			// Re-runs +page.ts, which refetches the list. This is how every
			// mutation refreshes its page.
			await invalidate(KEYS.projects);
			toast.success('Project deleted');
		} catch {
			toast.error('Failed to delete project. Please try again.');
		}
	}
</script>

<PageHeader title="Projects" subtitle="Everything your organization is working on" />

<div class="mb-6 flex justify-end">
	<button class="btn btn-primary" onclick={() => goto(resolve('/projects/new'))}>
		New project
	</button>
</div>

{#await data.projects}
	<div class="grid gap-4 md:grid-cols-2">
		{#each Array(4) as _, i (i)}
			<div class="skeleton h-32 w-full"></div>
		{/each}
	</div>
{:then projects}
	{#if projects.length === 0}
		<p class="text-base-content/60">No projects yet. Create the first one.</p>
	{:else}
		<div class="grid gap-4 md:grid-cols-2">
			{#each projects as project (project.id)}
				<div class="card bg-base-100 border-base-200 border shadow-sm">
					<div class="card-body">
						<h2 class="card-title">{project.name}</h2>
						{#if project.description}
							<p>{project.description}</p>
						{/if}
						<div class="card-actions justify-end">
							<button class="btn btn-outline btn-sm" onclick={() => handleDelete(project.id)}>
								Delete
							</button>
						</div>
					</div>
				</div>
			{/each}
		</div>
	{/if}
{:catch}
	<div class="alert alert-error">
		<span>Failed to load projects. Please try again.</span>
	</div>
{/await}
```

The `confirm()` dialog keeps this example short; the notes page shows the same flow with a proper modal.

Create the form at `frontend/src/routes/(protected)/projects/new/+page.svelte`, using the kit's `createFormValidation` helper the same way the notes create modal does:

```svelte
<script lang="ts">
	import { goto } from '$app/navigation';
	import { resolve } from '$app/paths';
	import { z } from 'zod';
	import { createProject } from '$lib/api/gen/projects';
	import { getErrorMessage } from '$lib/api/fetch';
	import { createFormValidation } from '$lib/util/createFormValidation.svelte';
	import PageHeader from '$lib/components/PageHeader.svelte';

	const schema = z.object({
		name: z.string().trim().min(1, 'Project name is required'),
		description: z.string()
	});
	const form = createFormValidation(schema, { name: '', description: '' }, () => (submitError = ''));
	let submitError = $state('');
	let submitting = $state(false);

	async function handleSubmit(e: Event) {
		form.handleSubmit(e, async (values) => {
			submitting = true;
			submitError = '';
			try {
				await createProject({
					name: values.name,
					description: values.description || null
				});
				goto(resolve('/projects'));
			} catch (error) {
				submitError = getErrorMessage(error, 'Failed to create project. Please try again.');
			} finally {
				submitting = false;
			}
		});
	}
</script>

<PageHeader title="New project" />

<form class="max-w-md space-y-4" onsubmit={handleSubmit} onfocusout={form.handleBlur} novalidate>
	{#if submitError}
		<div class="alert alert-error"><span>{submitError}</span></div>
	{/if}

	<div class="form-control">
		<label class="label" for="name"><span class="label-text">Name</span></label>
		<input
			id="name"
			name="name"
			type="text"
			class="input input-bordered"
			class:input-error={form.errors.name}
			bind:value={form.formData.name}
			oninput={form.handleChange}
		/>
		{#if form.errors.name}
			<span class="text-error text-sm">{form.errors.name}</span>
		{/if}
	</div>

	<div class="form-control">
		<label class="label" for="description"><span class="label-text">Description</span></label>
		<textarea
			id="description"
			name="description"
			rows="3"
			class="textarea textarea-bordered"
			bind:value={form.formData.description}
			oninput={form.handleChange}
		></textarea>
	</div>

	<div class="flex gap-2">
		<button type="submit" class="btn btn-primary" disabled={submitting || !form.isValid}>
			{submitting ? 'Creating...' : 'Create project'}
		</button>
		<button type="button" class="btn btn-outline" onclick={() => goto(resolve('/projects'))}>
			Cancel
		</button>
	</div>
</form>
```

There is no `invalidate()` after create: navigating to `/projects` runs its load function anyway.

## 12. Add it to the sidebar

Navigation lives in `frontend/src/routes/(protected)/menu.ts` as a typed array. Add an entry:

```ts
{
	id: 'projects',
	icon: 'lucide--folder',
	label: 'Projects',
	url: resolve('/projects'),
	minRole: 'readonly'
},
```

`minRole: 'readonly'` matches the routes: read-only users can open the list, and the write buttons are the backend's problem to refuse.

## 13. Run it end to end

Start the frontend (with the backend still running):

```bash
cd frontend
npm run dev
```

Open [localhost:5173/projects](http://localhost:5173/projects). You will see the skeleton flash, then the project you created in step 8. Create another through the form, delete one from the list, and watch the list refresh without a page reload.

## Recap

- Write the migration (`deploy`, `revert`, `verify`) and deploy it with `./sqitch.sh dev deploy`.
- Add the four backend pieces: `project_model.py`, `project_repo.py`, `project_service.py`, `project_route.py`.
- Register all of it in `container.py`: two `Singleton` providers **and** the `wiring_config` entry.
- Include the router in `router.py` and confirm it in `/docs`.
- Regenerate the typed client with `npm run generate`.
- Add an invalidation key, a `+page.ts` load that returns an un-awaited promise, a page that renders it with `{#await}`, and a menu entry.

The same thirteen steps add any entity to FastSvelte. When your feature outgrows plain CRUD, the notes feature shows where each addition goes: quotas in the route via the usage service, AI actions via the copilot pattern, business rules in the service.
