---
description: "Replace the FastSvelte note copilot demo with your own application — plan your domain, build your features, and remove the demo code."
keywords: "fastsvelte remove demo, replace demo, note copilot, remove sample app, customize fastsvelte"
---

# Replacing the Demo

FastSvelte ships an AI copilot demo (note Improve / Summarize) to show the patterns. After this guide you can strip it and start clean on your own product.

## 1. Plan your application

- What is your main domain model? (Projects, Tasks, Documents, …)
- What actions can users perform? (create, edit, share, export, …)
- What billing model will you use? (per-seat, usage-based, tiered, …)

## 2. Build your features

Follow **[Adding a Feature](adding-a-feature.md#adding-a-new-entity-end-to-end)** to add an entity end to end (schema → repository → service → route → UI), then apply the same pattern to your own domain.

## 3. Remove the demo code

The AI billing/usage infrastructure (`llm_client.py`, `openai_client.py`, and the usage/credit services + repos) is reusable — keep it if you want AI features of your own (see [AI Copilot](../features/ai.md)).

**Backend:**

```bash
cd backend
rm app/model/note_model.py app/model/copilot_model.py
rm app/service/note_service.py app/service/copilot_service.py
rm app/api/route/note_route.py
rm app/data/repo/note_repo.py
```

**Frontend:**

```bash
cd frontend
rm -rf src/routes/(protected)/notes
```

**Database** — drop the note table:

```bash
cd backend/db
sqitch add remove_note_table -n "Remove note demo table"
```

Edit `deploy/remove_note_table.sql`:

```sql
-- Deploy fastsvelte:remove_note_table to pg

BEGIN;

DROP TABLE IF EXISTS fastsvelte.note CASCADE;

COMMIT;
```

Fill `revert/remove_note_table.sql` with the original `CREATE TABLE` from `deploy/001_schema.sql`, then `./sqitch.sh dev deploy`.

**Optional** — if you're not using AI at all:

```bash
cd backend
uv remove openai
```
