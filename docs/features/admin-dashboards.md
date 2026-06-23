---
description: "FastSvelte admin and user dashboards — system-admin analytics, user/plan/organization management, health, and AI usage; org-admin member and invitation management; and the user dashboard."
keywords: "fastsvelte admin dashboard, saas admin panel, user dashboard, role-based dashboard, analytics, user management, organization management"
---

# Admin & User Dashboards

FastSvelte ships role-aware dashboards so you don't build admin tooling from scratch. What each user sees is gated by their [role](authentication.md#roles).

## System Admin

Platform-wide tools (`sys_admin`), under `/admin`:

- **Analytics** — usage and signup metrics
- **Users** — manage all users across every organization
- **Plans** — define subscription tiers and link Stripe products (see [Billing & Subscriptions](billing.md))
- **Organizations** — create and manage tenants (the entry point for [B2B onboarding](multi-tenancy.md))
- **AI Usage** — fleet-wide token and credit usage (see [AI Usage & Credit Billing](ai-billing.md))
- **Health** — system health checks
- **Settings** — system-level settings

## Organization Admin

For `org_admin` (B2B), under `/organization`:

- **Members** — view and manage organization users, change roles
- **Invitations** — invite teammates by email (see [Multi-Tenancy](multi-tenancy.md))

## User

For every authenticated user:

- **Dashboard** — their workspace overview
- **Billing** — subscription plus AI credits and usage (see [AI Usage & Credit Billing](ai-billing.md))
- **Profile / Settings** — account details and preferences

Every view is built on the same [role-based access](authentication.md#roles) and the type-safe [API client](../guides/orval.md).
