---
description: "FastSvelte type-safe API client — how Orval generates a TypeScript client from the FastAPI OpenAPI spec, and how to regenerate it after backend changes."
keywords: "fastsvelte orval, type-safe api client, openapi, typescript client generation, fastapi openapi, sveltekit api client"
---

# Type-Safe API Client (Orval)

FastSvelte keeps the frontend and backend in sync with a TypeScript API client **generated from the backend's OpenAPI spec** by Orval. Change a Pydantic model or route, regenerate, and the TypeScript compiler catches any breakage at compile time.

## Regenerate after backend changes

```bash
cd frontend
npm run generate   # reads the OpenAPI spec and regenerates the client (backend must be running)
```

Generated code lives in `frontend/src/lib/api/gen/` — **don't edit it by hand**; it's overwritten on every generate.

## How it works

1. Define Pydantic models and routes with `response_model` and `operation_id`:

    ```python
    @router.post("", response_model=NoteResponse, operation_id="createNote")
    async def create_note(data: CreateNoteRequest, ...): ...
    ```

    - `response_model` → the response type in the OpenAPI spec
    - `operation_id` → the generated TypeScript function name

2. FastAPI generates the OpenAPI spec from the decorators and models.
3. Orval reads the spec and emits typed functions and models.
4. Import and use them with full type safety:

    ```typescript
    import { createNote } from "$lib/api/gen/notes";
    import type { CreateNoteRequest, NoteResponse } from "$lib/api/gen/model";

    const note: NoteResponse = await createNote({ title: "My Note", content: "..." });
    ```

Change the backend → run `npm run generate` → the compiler flags any frontend code that no longer matches.
