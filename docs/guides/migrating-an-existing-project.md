---
description: "Migrate an existing app onto FastSvelte with your AI agent: plan the migration, break it into small reviewable tasks, and port feature by feature."
keywords: "fastsvelte migration, migrate existing project, port app to fastsvelte, ai agent migration, rebuild app on starter kit"
---

# Migrating an Existing Project

You have an app that works. Maybe you built it with AI, maybe by hand, and now you want it standing on FastSvelte's foundation instead of hardening your own. This guide is the process we used to migrate a production app onto FastSvelte ourselves, driven by an AI agent the whole way.

The direction matters: you do not port FastSvelte into your project. You move your project onto a fresh FastSvelte clone, where auth, billing, organizations, and email already work, so the only things your agent rebuilds are your own features.

The process has three phases, and each one ends with the agent stopping so you can review. That stop-and-review rhythm is the whole trick. Agents are good at executing a small, well-defined step and bad at knowing when they have drifted; the phase boundaries are where you catch the drift.

## Prerequisites

- A FastSvelte clone, set up and running locally (see [Project Setup](project-setup.md))
- Your existing project available on the same machine
- An AI agent (Claude Code, Cursor, or similar) running **in the FastSvelte clone**, not in your old project
- 15 minutes to read the plan your agent writes. Do not skip that part.

!!! note "The old project is read-only"

    The agent reads your existing code to understand it and never modifies it. If anything goes wrong, your old app is untouched and still deployable.

!!! warning "Commit after `init.py`"

    Run the initial setup, then commit before the migration starts. Phase 3 uses "revert to the last good commit" as its safety net, and that only protects you if your configured, working baseline is actually committed rather than sitting in the working tree.

## Phase 1: Plan the migration

Paste this prompt into your agent, with the path filled in:

```text
I want to migrate an existing project onto this FastSvelte codebase.

The existing project lives at: /full/path/to/my-old-project
Treat it as read-only: read anything, modify nothing there.

Create a folder named migration/ at the root of this project and write
migration/plan.md containing:

1. Feature inventory: every user-facing feature of the existing project,
   with the routes, data models, and third-party integrations behind it.
2. Already covered: which of those features FastSvelte already provides
   (auth, organizations, billing, email, AI metering), and what
   configuration they need instead of code.
3. To port: the features that must be rebuilt here, each mapped to the
   FastSvelte layers it will touch (schema, repository, service, route,
   frontend).
4. Data migration: what existing data must move, and a first idea of how.
5. Open questions: anything you could not determine from the code alone.

Never assume. When a feature's purpose, a business rule, or an intended
behavior is not clear from the code, stop and ask me instead of guessing.
A wrong assumption in this plan changes every task that follows.

Do not write or change any other file yet. When the plan is ready, stop
so I can review it.
```

Expect the agent to ask questions while it plans. That is the behavior you want: an agent that asks ten questions writes a dramatically better plan than one that quietly guesses ten answers, and the plan is where guesses are cheapest to prevent.

Then read `migration/plan.md` carefully. This is the highest-leverage review of the whole migration: a wrong assumption caught here costs one sentence to fix; caught in phase 3 it costs an afternoon.

Check three things in particular:

- **Is the feature inventory complete?** You know your product; the agent only knows the code. Add what it missed.
- **Did it map features to FastSvelte's built-ins correctly?** A common miss is rebuilding something FastSvelte already ships, like password reset or invitations.
- **Are the open questions answered?** Answer them in a follow-up message and have the agent update the plan.

Iterate until the plan describes the migration you actually want. Then move on.

## Phase 2: Break the plan into tasks

Once the plan is right, paste this:

```text
The migration plan in migration/plan.md is approved. Now break it into
tasks.

Create migration/tasks/, migration/tasks/done/, and
migration/tasks/index.md.

Split the plan into tasks that each take roughly 30 to 45 minutes. One
task is one .md file in migration/tasks/, structured like this:

---
title: <what this task delivers>
effort: 30-45m
depends-on: <other task files, if any>
---

# <title>

## Scope
- [ ] concrete steps, checkable one by one

## Pointers
- Old project: <files to read>
- This project: <files to create or change>

## Done when
- <a verifiable outcome, including the tests to run>

## Manual check
- <only when needed: what I should click through in the UI before we
  continue>

Rules:
- Follow the porting order from the plan; respect dependencies.
- Every task that changes the schema, a visible page, or anything in
  auth or billing must include a Manual check section.
- migration/tasks/index.md lists every task in execution order with a
  one-line description and a status.

Do not implement anything yet. Stop when the tasks are ready for my
review.
```

Skim the tasks. The thing to verify is size: if a task says "port the entire dashboard," send it back to be split. Small tasks are what keep every diff reviewable and every failure cheap.

!!! tip "Why 30 to 45 minutes"

    That is the window where an agent stays reliable and a human still actually reads the diff. Both fall off fast beyond it.

## Phase 3: Implement, one task at a time

Now the loop. Each iteration is one prompt:

```text
Work on the next task in migration/tasks/index.md.

Implement exactly that one task and nothing beyond it. Run the tests it
names. When it is done, move its file to migration/tasks/done/, update
the status in index.md, and stop. If the task has a Manual check
section, tell me what to verify before we continue.
```

Between iterations, you do three small things:

1. **Review the diff.** One task's worth of changes, a few minutes.
2. **Commit.** One task, one commit. If a task turns out wrong later, it reverts cleanly.
3. **Run the manual check when the task asks for one.** Click through the flow in the running app. The agent cannot see your UI; this is the part only you can do.

Repeat until `migration/tasks/` is empty and `done/` is full.

!!! warning "Do not let the agent run ahead"

    "Implement exactly that one task and stop" is in the prompt for a reason. An agent that batches five tasks produces one unreviewable diff, and you lose the ability to catch drift at the boundaries. If it runs ahead anyway, revert to the last good commit and re-run the single task.

## Phase 4: Cutover

When every task is done:

1. **Move the data.** Write the import against your new schema (a script or a one-off Sqitch migration), run it against a staging database first, and verify counts and a few known records by hand.
2. **Deploy** the FastSvelte app alongside the old one (see [Deployment](../deployment/index.md)) and use it with real accounts for a few days.
3. **Switch DNS** when you trust it. Keep the old app runnable until you have been on the new one comfortably for a while.

The `migration/` folder has served its purpose at this point. Keep it in git history and delete it from the working tree, the same way you would any finished planning document.
