---
description: "Add a metered, plan-limited feature to FastSvelte: define a FeatureKey, set per-plan limits, enforce the quota, and optionally let credits top it up."
keywords: "fastsvelte metering, custom feature quota, featurekey, plan limits, usage-based feature, credit top-up"
---

# Metering a Custom Feature

After this guide you can put any action behind a per-plan usage limit and, optionally, let credit top-ups extend it the way AI tokens do. See [Plans & Usage](../features/plans-and-usage.md) for the engine this builds on.

## 1. Declare the feature

Add a `FeatureKey` member and a `PlanFeatures` field in `backend/app/model/plan_model.py`:

```python
class FeatureKey(str, Enum):
    MAX_NOTES = "max_notes"
    TOKEN_LIMIT = "token_limit"
    ENABLE_AI = "enable_ai"
    MAX_PROJECTS = "max_projects"   # new

class PlanFeatures(BaseModel):
    ...
    max_projects: int = Field(
        default=0,
        title="Max Projects",
        json_schema_extra={"kind": "quota", "allows_unlimited": True},
    )
```

The field is the whole declaration. The **default** is what an organization gets when a plan's JSON omits the key, so 0 keeps the feature blocked until you set a limit. `title` is the label the admin form and the billing page display, and `kind: "quota"` gives the feature a used/limit row with a meter on the billing page. `test_plan_features.py` pins `FeatureKey` to the model's fields, so forgetting one half fails the test run.

!!! info
    The frontends read this declaration from `GET /plan/feature-schema` and the effective values with usage from `GET /usage/features`. That is why the billing page row and the admin form warnings appear with no frontend change.

!!! note
    For a boolean feature, declare a `bool` field with `kind: "flag"` and enforce it in the endpoint that serves the feature. `enable_ai`, checked by the copilot endpoints via `ensure_ai_in_plan`, is the shipped example.

## 2. Set limits per plan

Add `max_projects` to each plan's `features` map, either from the admin Plans page or in seed data (see [Plans & Usage](../features/plans-and-usage.md#configuring-limits)). Plans that do not have the key yet keep working: the admin form warns that `max_projects` is not set and will default to 0.

## 3. Enforce the limit

Wrap the action with the generic usage service. Check the quota before, record usage after:

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
