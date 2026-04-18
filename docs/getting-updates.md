---
description: "How to get updates from FastSvelte while maintaining your customizations. Learn strategies for pulling upstream changes, managing git remotes, and minimizing merge conflicts."
keywords: "fastsvelte updates, git upstream, merge upstream, starter kit updates, git remote, pull updates, version control, merge conflicts, fork maintenance"
---

# Getting Updates

[FastSvelte](https://fastsvelte.dev) is yours to customize and extend. You can stay connected to receive bug fixes and improvements, or disconnect completely and own your codebase.

## Minimizing Conflicts

**Extend rather than modify** to reduce merge conflicts. When you create new files instead of editing existing FastSvelte code, conflicts are limited to a few predictable integration points.

**Extension Pattern for Fewer Conflicts:**
- New routes, services, repositories in their own files
- New frontend pages and components
- New database migrations
- Conflicts are still expected in files such as `container.py` or `routes.py` where new components are registered. But manageable.

## Update Methods

### 1. Merge (Recommended)

Keep FastSvelte as an upstream remote and merge updates:

```bash
# During initial setup (instead of removing origin)
git clone https://github.com/harunzafer/fastsvelte.git my-project
cd my-project
git remote rename origin upstream
git remote add origin <your-repo-url>
git push -u origin main

# If you already removed origin, add upstream back
git remote add upstream https://github.com/harunzafer/fastsvelte.git

# Later, when you want updates
git fetch upstream

# View available updates
git log --oneline HEAD..upstream/main

# Merge updates
git merge upstream/main

# Resolve conflicts if any (usually in container.py, routes.py)
# After resolving, commit the merge

# Test the changes
cd backend && pytest                                    # For backend changes
cd frontend && npm run build && npm run test            # For frontend changes
cd landing && npm run build && npm run check            # For landing page changes

# Push to your repository
git push origin main
```

### 2. Cherry-pick

For selective updates or when you've modified core files extensively:

```bash
# Cherry-pick specific commits
git cherry-pick <commit-hash>

# Or a range of commits
git cherry-pick <start-commit>..<end-commit>

# Resolve conflicts and continue
git cherry-pick --continue
```


### 3. Manual Copy

Remove the FastSvelte remote and manually copy changes when needed:

```bash
# During initial setup (default approach in Getting Started)
git clone https://github.com/harunzafer/fastsvelte.git my-project
cd my-project
git remote remove origin
git remote add origin <your-repo-url>
git push -u origin main

# Later, check GitHub for updates you want
# Manually copy relevant changes to your project
```

**Best for:** Projects with heavy customization or teams that prefer full control.

## Best Practices

1. **Always test updates in development first** - Never update production directly
2. **Create a backup branch** before major updates
3. **Review release notes** to understand what changed

---

**Next Steps:**
- [Development Guide](development.md) - Build your application
- [Deployment](deployment/index.md) - Deploy to production
