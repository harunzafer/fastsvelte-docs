---
description: "FastSvelte usage metering and plan limits — define per-feature quotas in plans, meter usage per billing period, and enforce limits from one generic FeatureKey system."
keywords: "fastsvelte usage limits, plan features, quotas, metering, featurekey, usage tracking, per-feature limits, saas quotas"
---

# Plans & Usage

FastSvelte meters usage and enforces per-feature limits through one generic system. The same engine powers the note quota and the AI token allotment — and you can extend it to anything.

## How it works

- **Plans define limits.** Each plan carries a `features` map (typed by `PlanFeatures` + the `FeatureKey` enum in `backend/app/model/plan_model.py`). The kit ships `max_notes`, `token_limit`, and `enable_ai`.
- **Usage is metered per billing period.** `OrganizationUsageService` tracks each org's consumption per `FeatureKey` against the plan's limit, scoped to the current subscription period — it resets when the period rolls over.
- **Limits are enforced.** Call `check_quota_for(...)` before an action and `update_usage(...)` after:

```python
if not await usage_service.check_quota_for(org_id, FeatureKey.MAX_NOTES, 1):
    raise QuotaExceeded(FeatureKey.MAX_NOTES)
# ... perform the action ...
await usage_service.update_usage(org_id, FeatureKey.MAX_NOTES, 1)
```

The note demo does exactly this on create and delete.

## Configuring limits

Set each plan's limits in its `features` map via the admin Plans page or seed data — see [Admin & User Dashboards](admin-dashboards.md). How plans map to Stripe products is covered in [Billing & Subscriptions](billing.md).

## Add your own metered feature

Metering a new feature is a small change: add a `FeatureKey` + `PlanFeatures` field, set limits per plan, and wrap your action with `check_quota_for` / `update_usage`. Full walkthrough: **[Metering a Custom Feature](../guides/metering-a-custom-feature.md)**.

## AI usage

The AI token allotment (`token_limit`) is just one metered feature — but it adds a paid **credit top-up** layer on top. See [AI Usage & Credit Billing](ai-billing.md).
