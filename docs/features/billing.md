---
description: "Stripe billing in FastSvelte: subscriptions via the Stripe Customer Portal, linking products to plans, webhooks, local development, and going live."
keywords: "fastsvelte stripe, subscription billing, stripe customer portal, stripe webhooks, stripe cli, payment processing"
---

# Billing & Subscriptions

FastSvelte handles **subscription billing** through Stripe's **Customer Portal**, so there is no custom billing UI to build. Subscription state is **webhook-driven**: Stripe is the source of truth, and the backend updates the database only from webhook events sent to `/webhooks/stripe`. Credit-pack purchases have a second path: the billing page verifies the checkout with Stripe after the redirect, so a lost webhook cannot cost a customer their credits. Three columns hold the link: `organization.stripe_customer_id`, `plan.stripe_product_id`, and `organization_plan` (status, period, subscription id).

## Setup

### 1. API key

Create a Stripe account and a [sandbox](https://docs.stripe.com/sandboxes/dashboard/manage#create-a-sandbox) for development, then copy your **Secret key** (Developers → API Keys; `sk_test_...` in sandbox) into `backend/.env`:

```bash
FS_STRIPE_API_KEY=sk_test_YOUR_KEY_HERE
```

Only the secret key is needed; the portal handles all payment UI. Never commit it.

### 2. Create products

In **Products → Add Product**, create your tiers and copy each **Product ID** (`prod_...`):

- **Free tier**: a single **monthly** price; must be **$0** (auto-provisioned on first login).
- **Paid tiers**: **monthly** and **annual** prices in the same currency.

!!! tip "No public free tier?"

    You still need a $0 default product for auto-provisioning. Name it "Default" and leave it out of the portal, so users can't select it.

### 3. Link products to plans

The kit seeds three plans (Free, Professional, Premium). At **`/admin/plans`**, edit each and paste its **Stripe Product ID**. The default plan (Free) must map to your $0 product. It's validated and auto-assigned to new users.

### 4. Configure the Customer Portal

In **Settings → Billing → Customer portal**: enable "customers can switch plans" and add the products/prices you want to offer; enable updating payment methods and viewing invoices, and optionally cancellation. Save.

## How plans are created and assigned

The kit seeds three plans (Free, Professional, Premium); Free is flagged as the default. From there, assignment is automatic:

1. **New organizations get the Free plan on their own.** In B2C this happens at first login, in B2B at organization creation and on org admin logins. The same background step also creates the Stripe customer. It never blocks a login: if it can't finish (for example, Stripe keys are missing), it logs the error and tries again on the next login.
2. **Paid plans arrive by webhook.** When a customer subscribes through Stripe Checkout, the `customer.subscription.created/updated` events update the organization's plan to match Stripe.
3. **Fallback.** An organization with no plan row uses whichever plan is flagged as default. Only when both are missing does the app treat the organization as having no plan: the billing page says so, and AI calls run on purchased credits alone (or are refused when there are none).

**If the billing page shows "No AI plan active":** no plan resolved for that organization. Check the backend log for onboarding errors (usually Stripe configuration) and confirm one plan still has the default flag set (Admin → Plans).

## Local development

Forward Stripe webhooks to your backend with the [Stripe CLI](https://docs.stripe.com/stripe-cli/install):

```bash
stripe login
stripe listen --forward-to localhost:8000/webhooks/stripe
```

Copy the printed `whsec_...` into `backend/.env`, then restart the backend (keep `stripe listen` running in a second terminal):

```bash
FS_STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxx
```

## Testing

Use Stripe [test cards](https://stripe.com/docs/testing) in sandbox (any future expiry, any CVC and postal code):

| Card | Result |
|------|--------|
| `4242 4242 4242 4242` | Success |
| `4000 0000 0000 0002` | Declined |
| `4000 0025 0000 3155` | 3D Secure |

## Going live

In **Developers → Webhooks**, add an endpoint at `https://api.yourdomain.com/webhooks/stripe` subscribed to these events, and copy its signing secret to `FS_STRIPE_WEBHOOK_SECRET`:

- `customer.subscription.created` / `updated` / `deleted`
- `checkout.session.completed`: fulfills [AI credit-pack](ai-billing.md) purchases

Then switch to **Live mode**: recreate products with live pricing (free tier still $0), use live `sk_live_...` keys, update `/admin/plans` with the live Product IDs, configure the portal in live mode, and test a real upgrade.

## Troubleshooting

**Free subscription didn't sync (dev).** If you logged in before `stripe listen` was running, the free subscription exists in Stripe but not your database ("No active plan found"). Fix it from `/billing` → **Manage Subscription** → in the portal click **Cancel subscription**, then **Don't cancel**. That fires `customer.subscription.updated` and syncs. (Or resend the original event from **Stripe → Events**.)

## Related

[Plans & Usage](plans-and-usage.md) · [AI Usage & Credit Billing](ai-billing.md) · [Deployment](../deployment/index.md)
