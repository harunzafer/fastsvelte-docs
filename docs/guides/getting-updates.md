---
description: "How to get updates from FastSvelte: why projects diverge over time, how to cherry-pick security and bug fixes, and when merging still makes sense."
keywords: "fastsvelte updates, git upstream, cherry-pick, security fixes, starter kit updates, merge conflicts, fork maintenance"
---

# Getting Updates

[FastSvelte](https://fastsvelte.dev) is yours to customize and extend. From the first week you will add your own migrations, routes, and pages, and your project starts to diverge from the kit. That is by design: FastSvelte is a starting point you own, not a framework you track.

Be realistic about what that means for updates:

- **Early on**, merging upstream works well. Your diff is small and conflicts are few.
- **Over time, merging stops being sustainable.** Your schema, routes, and components drift away from the kit's, and a full merge brings more conflict than value. This is normal and expected, not a failure.
- **What always works: taking specific fixes.** FastSvelte ships security and critical fixes as standalone commits precisely so you can cherry-pick them into any project, no matter how far it has diverged.

Starting a new project? Clone the latest FastSvelte. Everything below is for projects already in flight.

## Set Up the Upstream Remote

Every method below needs FastSvelte available as a remote:

```bash
# During initial setup (instead of removing origin)
git clone https://github.com/harunzafer/fastsvelte.git my-project
cd my-project
git remote rename origin upstream
git remote add origin <your-repo-url>
git push -u origin main

# If you already removed origin, add upstream back
git remote add upstream https://github.com/harunzafer/fastsvelte.git
```

## Security and Critical Fixes

Read this section even if you never merge. We publish security and critical fixes as **standalone commits**, prefixed `[security]` or `[fix]`, so they can be taken in isolation:

```bash
git fetch upstream

# List the fixes you don't have yet
git log --oneline --grep='^\[security\]' --grep='^\[fix\]' HEAD..upstream/main

# Take one
git cherry-pick <sha>
```

If a fix requires anything beyond the code change (rotating a secret, invalidating sessions), the commit message says so. Read it before picking.

If the cherry-pick conflicts with your customizations, the fix is usually small: inspect it with `git show <sha>` and apply the change manually.

!!! tip "Get notified of security fixes"

    On GitHub, Watch the FastSvelte repository with Custom → Releases. Every `[security]` fix is also published as a GitHub Release, and critical ones are announced to customers by email.

## After Any Update

Whatever method you used, finish with these steps:

```bash
# If the update touches backend/db/, deploy the new migrations
cd backend/db && ./sqitch.sh dev deploy      # repeat against prod when you release

# If backend API routes or models changed, regenerate the API client
cd frontend && npm install && npm run generate

# Test before pushing
cd backend && pytest
cd frontend && npm run build && npm run test
cd landing && npm run build && npm run check
```

## Update Methods Over a Project's Life

### Early project: merge

While your project is young and close to the kit, merging takes everything at once:

```bash
git fetch upstream
git log --oneline HEAD..upstream/main   # review what's coming
git merge upstream/main
# Resolve conflicts (usually container.py, routes.py), commit, test, push
```

### Established project: cherry-pick what you need

Once merging hurts more than it helps, switch to taking only what matters, using the same commands as the security section above. Cherry-picking works for any upstream commit, not only fixes. One caution: release commits build on each other, so picking a large feature release into a heavily diverged project can conflict extensively. Fixes are kept small for exactly this reason; features are take-at-your-own-risk.

### Fallback: manual copy

For maximum control, skip git entirely: review the change on GitHub and copy what you need into your project. This always works, and for a heavily customized file it is often faster than resolving a conflict.

## Minimizing Conflicts

**Extend rather than modify** to reduce merge conflicts. When you create new files instead of editing existing FastSvelte code, conflicts are limited to a few predictable integration points:

- New routes, services, repositories in their own files
- New frontend pages and components
- New database migrations

Conflicts still happen in the registration points (`container.py`, `routes.py`) where new components hook in, but they stay small and predictable.

## Best Practices

1. **Test updates in development first.** Never update production directly.
2. **Create a backup branch** before major updates.
3. **Review the commit message** of anything you pick: fixes that need manual steps say so there.

## Next steps

- **[Development Workflow](development-workflow.md)**: build your application
- **[Deployment](../deployment/index.md)**: deploy to production
