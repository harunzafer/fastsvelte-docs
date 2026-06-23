---
description: "Add a metered, plan-limited feature to FastSvelte — define a FeatureKey, set per-plan limits, enforce the quota, and optionally let credits top it up."
keywords: "fastsvelte metering, custom feature quota, featurekey, plan limits, usage-based feature, credit top-up"
---

# Metering a Custom Feature

After this guide you can put any action behind a per-plan usage limit — and, optionally, let credit top-ups extend it the way AI tokens do. See [Plans & Usage](../features/plans-and-usage.md) for the engine this builds on.

## 1. Define the feature

Add it to both the `FeatureKey` enum and the `PlanFeatures` model in `backend/app/model/plan_model.py`:

```python
class FeatureKey(str, Enum):
    MAX_NOTES = "max_notes"
    TOKEN_LIMIT = "token_limit"
    ENABLE_AI = "enable_ai"
    MAX_PROJECTS = "max_projects"   # new

class PlanFeatures(BaseModel):
    max_notes: int | None
    token_limit: int | None
    enable_ai: bool | None
    max_projects: int | None        # new
```

## 2. Set limits per plan

Add `max_projects` to each plan's `features` map — via the admin Plans page or seed data (see [Plans & Usage](../features/plans-and-usage.md#configuring-limits)).

## 3. Enforce the limit

Wrap the action with the generic usage service — check before, record after:

```python
if not await usage_service.check_quota_for(org_id, FeatureKey.MAX_PROJECTS, 1):
    raise QuotaExceeded(FeatureKey.MAX_PROJECTS)
project = await project_service.create(...)
await usage_service.update_usage(org_id, FeatureKey.MAX_PROJECTS, 1)
```

That's all it takes to meter a feature against the plan limit, reset each billing period.

## 4. (Optional) Let credits top it up

Today only AI tokens fall back to the [credit balance](../features/ai-billing.md). To let credits extend another feature, mirror what `AiUsageBillingService` does: when the per-period limit is exhausted, check and debit the org's credit balance before blocking the action.

Two caveats, since the credit system is currently AI-token-specific:

- The balance is a single **token-denominated** pool. If you want a separate balance per feature (e.g. "project credits"), add a per-feature balance rather than reusing the token pool.
- Decide the unit. Reusing the token pool only makes sense if your feature is denominated the same way.

So: metering any feature is built in; credit top-ups for non-AI features are a small, deliberate extension.
