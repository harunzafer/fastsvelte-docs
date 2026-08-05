---
description: "FastSvelte usage metering and plan limits: define per-feature quotas in plans, meter usage per billing period, and enforce limits from one generic FeatureKey system."
keywords: "fastsvelte usage limits, plan features, quotas, metering, featurekey, usage tracking, per-feature limits, saas quotas"
---

# Plans & Usage

FastSvelte meters usage and enforces per-feature limits through one generic system. The same engine powers the note quota and the AI token allotment, and you can extend it to anything.

## How it works

- **Plans declare features once.** `PlanFeatures` in `backend/app/model/plan_model.py` is the single definition of what a plan's `features` JSON may contain. Each field is one feature and carries a default, a display label, and a kind: `"quota"` for a numeric per-period limit, `"flag"` for a boolean toggle. The kit ships `max_notes` and `token_limit` as quotas and `enable_ai` as a flag. Validation, enforcement and the billing page rows all derive from this model.
- **Usage is metered per billing period.** `OrganizationUsageService` tracks each org's consumption per `FeatureKey` against the plan's limit, scoped to the current subscription period; it resets when the period rolls over.
- **Limits are enforced.** Call `check_quota_for(...)` before an action and `update_usage(...)` after:

```python
if not await usage_service.check_quota_for(org_id, FeatureKey.MAX_NOTES, 1):
    raise QuotaExceeded(FeatureKey.MAX_NOTES)
# ... perform the action ...
await usage_service.update_usage(org_id, FeatureKey.MAX_NOTES, 1)
```

The note demo does exactly this on create and delete.

**Flags are enforced where the feature is served.** A quota is checked by the generic engine; a flag is checked by the endpoint that provides the feature. `enable_ai` gates the copilot endpoints with a `FEATURE_DISABLED` error, distinct from `QUOTA_EXCEEDED` so your frontend can tell "upgrade your plan" apart from "you ran out this period".

## Configuring limits

Set each plan's limits in its `features` map via the admin Plans page or seed data.

A numeric limit of `-1` means **unlimited**. `{"max_notes": -1}` grants notes with no ceiling, and the billing page shows "Unlimited" in place of a usage meter.

Any other negative value is not unlimited. It resolves to a limit of zero and blocks the feature, so a plan you meant to uncap would refuse every request.

See [Admin & User Dashboards](admin-dashboards.md). How plans map to Stripe products is covered in [Billing & Subscriptions](billing.md).

## When the JSON and the model disagree

The features JSON is edited free-form, so it can drift from `PlanFeatures`. Reads never fail because of it:

- A **missing** key falls back to the field default: quotas to 0 (the feature is blocked until you set a value), flags to false.
- A value of the **wrong type** is ignored and the default applies, with a warning in the backend log.
- An **extra** key is stored but ignored until you add a matching `PlanFeatures` field.

The admin Plans form shows the same three cases as live warnings under the features editor, and the save response repeats them. They never block saving: you may edit a plan's JSON before updating `PlanFeatures` or the other way around, and either order works.

## Add your own metered feature

Metering a new feature is one declaration and one enforcement call: add a `FeatureKey` and a `PlanFeatures` field, set limits per plan, and wrap your action with `check_quota_for` / `update_usage`. The admin form validation and the billing page row follow from the declaration, with no frontend change. Full walkthrough: **[Metering a Custom Feature](../guides/metering-a-custom-feature.md)**.

## AI usage

The AI token allotment (`token_limit`) is just one metered feature, but it adds a paid **credit top-up** layer on top. See [AI Usage & Credit Billing](ai-billing.md).
