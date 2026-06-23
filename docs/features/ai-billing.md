---
description: "FastSvelte AI usage and credit billing — token metering, the monthly plan allotment, non-expiring credit packs, model pricing, usage reporting, and overage settings."
keywords: "fastsvelte ai billing, token metering, usage-based billing, credit packs, ai credits, llm cost, usage tracking, ai saas"
---

# AI Usage & Credit Billing

Every AI call is metered and billed against the **organization** (this kit is B2B multi-tenant; the org is the billing unit, which degrades naturally to "per user" for single-member orgs). Billing layers onto the existing [Stripe subscription](billing.md) model — there is no parallel per-user credit system.

A call's tokens are charged in a fixed order:

1. **Monthly allotment** — the plan's included token budget, resets each billing period.
2. **Credit-pack balance** — purchased, non-expiring tokens.
3. **Overage** — usage beyond both. **Blocked by default** (see [Overage settings](#overage-settings)).

## How a call is billed

`backend/app/service/ai_usage_billing_service.py` owns this. Two entry points:

- **`has_ai_capacity(org_id, estimated_tokens)`** — a pre-flight check the copilot routes call *before* streaming, so an org that's out of capacity gets a clean `QuotaExceeded` instead of a half-streamed answer. It estimates from input length (~4 chars/token).
- **`stream_and_record(...)`** / **`record_llm_usage(...)`** — after the call, the actual `TokenUsage` is split across allotment → credits → overage, the consumed buckets are debited, the USD cost is computed (see [Model pricing](#model-pricing)), and a row is written to `llm_usage_log` tagged with the `bucket` it drew from.

For streaming calls, `stream_and_record` forwards text chunks to the client and bills the final `TokenUsage` once the stream ends — identical accounting to a non-streamed call.

## The monthly allotment

The allotment is just one feature in the generic [Plans & Usage](plans-and-usage.md) system — the `token_limit` `FeatureKey`, metered per billing period and resetting each period. Set each plan's token budget alongside its other limits; it isn't a separate AI-only mechanism. An org with no `token_limit` has a zero allotment and must rely on credit packs.

## Credit packs

Packs are configured in **settings** — `settings.credit_packs`, with a default in `backend/app/config/settings.py` and overridable via the `FS_CREDIT_PACKS` env var (JSON). The defaults:

| Pack | Tokens | Price |
|------|--------|-------|
| `pack_5m` | 5,000,000 | $39 |
| `pack_10m` | 10,000,000 | $69 |
| `pack_25m` | 25,000,000 | $149 |

Change tokens or pricing in settings — no code edit, and **no Stripe products to create** (checkout builds the price inline via Stripe `price_data`). Purchased tokens are **non-expiring** and live in `organization_credit_balance`, separate from the per-period allotment.

**Purchase flow** (`backend/app/api/route/credit_pack_route.py`, all `ORG_ADMIN`):

| Endpoint | Purpose |
|----------|---------|
| `GET /billing/credit-packs` | List available packs |
| `POST /billing/credit-packs/checkout` | `{ pack_id, return_base_url }` → `{ url }` — redirect the user to this Stripe Checkout URL |

The org must already have a Stripe customer (complete subscription billing setup first). Fulfillment is handled on the Stripe **`checkout.session.completed`** webhook (`backend/app/api/route/stripe_webhook_route.py`): `CreditPackService.fulfill_purchase` checks `metadata.type == "credit_pack"`, credits the balance, and appends an `organization_credit_transaction` audit row.

## Model pricing

USD cost per call is computed from the `model_price` table (seeded by migrations `006_model_price.sql` + `008_more_model_prices.sql`), which ships current OpenAI GPT-4.1 and GPT-5 series pricing (standard tier, per 1M tokens). A sample:

| Provider | Model | Input / 1M | Output / 1M |
|----------|-------|-----------|-------------|
| openai | gpt-5-mini *(default)* | $0.25 | $2.00 |
| openai | gpt-5 | $1.25 | $10.00 |
| openai | gpt-5-nano | $0.05 | $0.40 |
| openai | gpt-4.1-mini | $0.40 | $1.60 |
| openai | gpt-4o-mini | $0.15 | $0.60 |

…plus `gpt-5.1`, `gpt-5.2`, `gpt-4.1`, `gpt-4o`, and the `-pro` variants. `LlmPricingService.compute_cost_usd` looks up the price (cached) and multiplies by input/output tokens. **If you point the copilot at a model that isn't in this table, calls will fail** — add a row (or a migration) for any model you enable.

## Usage reporting

`backend/app/api/route/usage_route.py` exposes:

| Endpoint | Role | Returns |
|----------|------|---------|
| `GET /usage/summary` | `MEMBER` | used / limit / credit tokens for the period, plus the caller's own usage |
| `GET /usage/history` | `MEMBER` (own rows) · `ORG_ADMIN`+ (whole org) | paginated per-call log (`limit`, `offset`) |
| `GET /usage/top-users` | `ORG_ADMIN` | top users by tokens this period |
| `GET /usage/fleet` | `SYSTEM_ADMIN` | fleet-wide rollup over `days` — backs the admin AI-usage page |

The user-facing usage page reads `/summary` + `/history`; the system-admin AI-usage page reads `/fleet`.

## Overage settings

Overage is gated by two settings, **both `false` by default**, so usage beyond allotment + credits is blocked (a hard cap) rather than silently charged:

- **`overage_enabled`** (org setting) — an org opts into overage.
- **`ai_overage_enabled`** (system setting, `SYSTEM_ADMIN`) — an operator-level kill switch; an org's `overage_enabled` only takes effect if this is also on.

Both use the existing generic setting mechanism (org settings via `/organization/{id}/{key}`, system settings via `/system/{key}`). Actually **charging** for overage (Stripe metered billing / invoice items) is intentionally not implemented yet — leave overage off until you wire it up.

## Schema

Migration `007_ai_credit_billing.sql` adds:

- `llm_usage_log` — append-only per-request log (tokens, cost, `bucket` = `allotment` \| `credit` \| `overage`).
- `organization_credit_balance` — running non-expiring credit-pack balance.
- `organization_credit_transaction` — audit log of credit purchases/debits.

## Next steps

- **[AI Integration](ai.md)** — the LLM client, the sample copilot, and streaming.
- **[Stripe Integration](billing.md)** — the subscription billing this builds on.
