---
description: "Complete guide to Stripe integration in FastSvelte - Setup, webhooks, local development, testing, and production deployment."
keywords: "fastsvelte stripe, stripe integration, stripe webhooks, subscription billing, payment processing, stripe cli, stripe testing"
---

# Stripe Integration Guide

FastSvelte currently supports **subscription-based billing** out of the box. One-time payments are on our roadmap and should be straightforward to implement on your own if needed.

## Architecture Overview

FastSvelte uses **Stripe's Customer Portal** for subscription management instead of building a custom billing UI. This approach offers several benefits:

- **Security**: Stripe handles sensitive payment information and PCI compliance
- **Familiarity**: Users recognize and trust Stripe's interface
- **Less maintenance**: No custom billing UI to build or maintain
- **Feature-complete**: Includes invoice history, payment methods, and subscription management

!!! info "Webhook-Driven Design"

    FastSvelte syncs subscription state exclusively through Stripe webhooks. Stripe is the source of truth - all subscription changes happen in Stripe, then Stripe sends webhook events to your backend (`/webhook/stripe`), which updates your database. FastSvelte never directly modifies subscription data outside of webhook handlers.

## Initial Setup

### 1. Create Stripe Account

1. Sign up at [stripe.com](https://stripe.com)
2. Navigate to your [Dashboard](https://dashboard.stripe.com)
3. Create a [Sandbox environment](https://docs.stripe.com/sandboxes/dashboard/manage#create-a-sandbox) for development

!!! info "Sandbox vs Live Mode"

    Stripe has two isolated environments:

            - **Sandbox** (`sk_test_...`): Development with [test cards](https://stripe.com/docs/testing#cards), no real payments
            - **Live Mode** (`sk_live_...`): Production with real payments and customers

            Data in each mode is completely separate.

### 2. Get API Keys

1. In your Stripe Dashboard, navigate to **Developers** → **API Keys** (or **Workbench** → **API Keys** for newer accounts)
2. Under "Standard keys", copy your **Secret key** (starts with `sk_test_` for test mode)
3. Add to `backend/.env`:

```bash
FS_STRIPE_API_KEY=sk_test_YOUR_KEY_HERE
```

!!! warning "Security"

    **Secret keys** must never be exposed in client-side code or committed to version control.

!!! note "Publishable Key Not Required"

    FastSvelte uses Stripe Customer Portal for all payment UI, so you only need the secret key. The publishable key is not needed unless you add custom checkout flows later.

### 3. Create Products in Stripe

Create your subscription tiers in Stripe:

1. Go to **Products** → **Add Product**
2. Create your subscription tiers (e.g., Free, Pro, Enterprise)
3. **For the Free tier**: Create only a **monthly** price (FastSvelte auto-provisions this on first login)
4. **For paid tiers**: Create both **monthly** and **annual** prices with the same currency
5. Copy each **Product ID** (starts with `prod_`)

!!! info "One Active Subscription Per Customer"

    FastSvelte maintains one active subscription per customer at a time. When a user signs up, they automatically get the Free tier subscription. Users can then upgrade/downgrade to other tiers through the Stripe Customer Portal.

!!! tip "Don't Want to Offer a Free Tier?"

    You still need to create a free tier product for auto-provisioning, but you can name it "Default" or something similar. Later, when configuring the Stripe Customer Portal, you can hide this tier from users so it doesn't appear as an option. This allows you to auto-provision users while only showing paid plans.

### 4. Link Stripe Products to Plans

FastSvelte comes with three plans pre-configured from seed data (Free, Professional, Premium). Configure them through the admin interface:

1. Log in as system admin and navigate to **`/admin/plans`**.
2. You'll see three existing plans from the database seed data
3. For each plan, click **Edit** and add the **Stripe Product ID** from step 3
4. **Important**: The Free plan (marked as default) must get your free tier's Product ID

!!! warning "Free Plan Requirements"

    The plan marked as `is_default = true` (Free tier) **must** have a $0 price in Stripe - FastSvelte validates this during auto-provisioning - This plan is automatically assigned to new users on first login

!!! info "Database Schema"

    Three tables handle Stripe data:

    - `organization.stripe_customer_id` - Links to Stripe Customer
    - `plan.stripe_product_id` - Links to Stripe Product
    - `organization_plan` - Stores subscription status, period dates, and Stripe subscription ID

### 5. Configure Stripe Customer Portal

The Customer Portal is where users manage their subscriptions, payment methods, and billing history.

1. In Stripe Dashboard, go to **Settings** → **Billing** → **Customer portal**
2. **Configure Products**:
   - Under subscriptions enable "customers can switch plans"
   - Then add the products (plans) with prices you want to offer for upgrade/downgrade in the portal
   - If you want to offer monthly and annual billing, make sure to add both prices for each product
3. **Configure Features**:
   - Enable "Customers can update payment methods"
   - Enable "Customers can view invoices"
   - Optionally enable "Customers can cancel subscriptions" (they'll downgrade to free tier)
   - Check out other features and configure as needed
4. **Save changes**

!!! tip "Hiding the Free Tier"

    If you don't want users to downgrade to the free tier on their own, simply don't add the free tier product to the portal configuration. They can still cancel their paid subscription which will auto-downgrade them to the free tier, but it won't be an explicit option in the portal.

## Local Development Setup

### 1. Install Stripe CLI

Follow the [official installation guide](https://docs.stripe.com/stripe-cli/install) for your platform.

### 2. Authenticate Stripe CLI

```bash
stripe login
```

- Go to the dashboard URL provided by the CLI
- Select your sandbox account
- Click "Allow access" to authorize the CLI with your Stripe account

This opens your browser to authorize the CLI.

### 3. Forward Webhooks to Localhost

```bash
stripe listen --forward-to localhost:8000/webhook/stripe
```

You'll see output like:

```
> Ready! Your webhook signing secret is whsec_xxxxxxxxxxxxx
```

### 4. Update Webhook Secret

Copy the `whsec_` secret and add to `backend/.env`:

```bash
FS_STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxx
```

### 5. Restart Backend

```bash
cd backend
# Restart your FastAPI server to pick up the new environment variable
```

!!! tip "Running Development Environment"

    Keep two terminals open:

    ```bash
    # Terminal 1: Run backend
    cd backend && uv run uvicorn app.main:app --reload

    # Terminal 2: Forward webhooks
    stripe listen --forward-to localhost:8000/webhook/stripe
    ```

### 6. Synchronize Free Subscriptions

In development mode, when you log in for the first time, FastSvelte auto-provisions a free subscription in Stripe. If you logged in before running `stripe listen`, the webhook won't be received and your database won't sync. To fix this:

1. Navigate to `/billing` and verify that you see "No subscription found" in the "Current Subscription" section
2. Click "Manage Subscription" button.
3. In the Stripe portal, click "Cancel subscription"
4. Then click "Don't cancel" to undo the cancellation
5. This triggers a `customer.subscription.updated` webhook
6. Go back to `/billing` and refresh. You should now see the free subscription details synced from Stripe.

## Testing

### Test Cards

Use these in Sandbox mode:

| Card Number           | Description        |
| --------------------- | ------------------ |
| `4242 4242 4242 4242` | Successful payment |
| `4000 0000 0000 0002` | Card declined      |
| `4000 0025 0000 3155` | 3D Secure required |

Use any future expiry, any CVC, any postal code.

## Production Webhook Setup

### Configure Webhook Endpoint

1. Go to [Stripe Dashboard → Developers → Webhooks](https://dashboard.stripe.com/webhooks)
2. Click **Add Destination**
3. Select events and continue. FastSvelte listens for three key subscription events:
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
4. On the next page, select "Webhook endpoint" and Enter your Endpoint URL: `https://api.yourdomain.com/webhooks/stripe
5. click "Create endpoint" and you'll see the webhook details page with the signing secret.
6. Copy the **Signing secret** (starts with `whsec_`)
7. Add to production environment:

```bash
FS_STRIPE_WEBHOOK_SECRET=whsec_YOUR_PRODUCTION_SECRET
```

## Production Checklist

Before going live:

- Switch to **Live mode** (not test/sandbox) in Stripe Dashboard
- Create your products in Live mode with correct pricing
- Verify the free tier product has a $0 price
- Get **Live API keys** (`sk_live_...`) and update `FS_STRIPE_API_KEY` in production
- Configure **webhook endpoint** at `https://api.yourdomain.com/webhooks/stripe`
- Copy **live webhook secret** to `FS_STRIPE_WEBHOOK_SECRET` in production
- Update plan records in `/admin/plans` with **live mode Product IDs**
- Configure **Customer Portal** settings in live mode
- Test free tier auto-provisioning with a new signup
- Test complete upgrade flow with a real card
- Set up **email notifications** for failed payments
- Review **tax settings** if applicable
- Set up **monitoring** for webhook failures

## Troubleshooting

### Missed Webhook During First Login

If you logg up before starting `stripe listen`, the free subscription exists in Stripe but not in your database. You'll see "No active plan found" warnings in logs and users can't access the billing page.

**Quick Fix (Development Only):**

1. Navigate to `/billing` and click "Manage Subscription"
2. In the Stripe portal, click "Cancel subscription"
3. Then click "Don't cancel" to undo the cancellation
4. This triggers a `customer.subscription.updated` webhook
5. Your database is now synced

**Alternative:** Find the subscription creation event in your [Stripe Dashboard → Events](https://dashboard.stripe.com/test/events) and click "Resend webhook".

## Related Documentation

- [Development Guide](../development.md) - Local setup
- [Deployment](../deployment/index.md) - Production deployment
- [Integrations](index.md) - Other integrations

## External Resources

- [Stripe Documentation](https://stripe.com/docs)
- [Stripe CLI](https://stripe.com/docs/stripe-cli)
- [Testing Stripe](https://stripe.com/docs/testing)
- [Webhook Best Practices](https://stripe.com/docs/webhooks/best-practices)
