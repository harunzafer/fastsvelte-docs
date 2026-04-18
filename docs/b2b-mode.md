---
description: "Configure FastSvelte for B2B team collaboration - Setup closed registration, user invitations, organization management, and role-based access control for multi-tenant SaaS applications."
keywords: "b2b saas, team collaboration, organization management, user invitations, multi-tenant, role-based access, fastsvelte b2b mode, closed registration, saas permissions"
---

# B2B Mode

[FastSvelte](https://fastsvelte.dev) supports two operational modes: **B2C** (Business-to-Consumer) and **B2B** (Business-to-Business). This guide explains how B2B mode works.

## What is B2B Mode?

In B2B mode, FastSvelte operates as a closed registration system where:

- Public signup is disabled
- Users can only join via invitation
- Organizations are managed by system administrators
- Each organization has its own admins who manage members

## Initial Setup

When you run `uv run init.py` and select **B2B mode**, the script creates:

1. A system administrator account (sys_admin)
2. Sets `FS_APP_MODE=b2b` in your environment

## The B2B Flow

### 1. System Administrator Login

After running `init.py`, log in with the sys_admin credentials created during setup.

### 2. Create an Organization

1. Navigate to **System > Organizations** in the sidebar
2. Click **Create Organization**
3. Fill in the organization details:
   - **Organization Name**: The company/organization name
   - **Admin Email**: Email address for the organization administrator
   - **Admin Full Name**: Full name for billing purposes

4. Click **Create Organization**

**What happens next:**
- A new organization is created in the database
- A Stripe customer is automatically created for billing
- An invitation email is sent to the admin email address
- The organization admin can accept the invitation to create their account

### 3. Organization Admin Accepts Invitation

The organization admin receives an invitation email with a unique link. When they click it:

1. They're directed to the invitation acceptance page
2. They set their password and complete their profile
3. Their account is created with the `org_admin` role
4. They can now log in and access their organization

!!! note "Email Service in Development"

    In development mode, the email service defaults to `stub` (no actual emails sent). To get invitation links, check your backend terminal for `[STUB EMAIL]` blocks containing the invitation URL. Copy and paste this URL in your browser to accept the invitation.

### 4. Organization Admin Manages Members

Once logged in, the org_admin can:

- **View Members**: See all users in their organization
- **Invite Users**: Send invitations to new members
- **Change Roles**: Update member roles (readonly, member, org_admin)
- **Remove Members**: Remove users from the organization

Access these features via **Organization > Users** in the sidebar.

### 5. Member Invitation Flow

When an org_admin invites a new member:

1. Navigate to **Organization > Invitations**
2. Create a new invitation with email and role
3. The invited user receives an email with an invitation link
4. They accept the invitation and create their account
5. They automatically join the organization with the assigned role

!!! note "Finding Invitation Links in Development"

    Since the email service defaults to `stub` in development, check your backend logs for `[STUB EMAIL]` blocks that contain the invitation URL.

## User Roles in B2B Mode

| Role | Description | Permissions |
|------|-------------|-------------|
| `sys_admin` | System Administrator | Full system access, can manage all organizations |
| `org_admin` | Organization Administrator | Can manage their organization's members and settings |
| `member` | Regular Member | Can use application features |
| `readonly` | Read-only User | View-only access |

## What's Disabled in B2B Mode

When running in B2B mode:

- ❌ Public signup page (`/signup`) returns 404
- ❌ OAuth signup creates accounts only for existing invited users
- ❌ Individual billing is hidden (billing managed at organization level)
- ✅ Only invitation-based registration is allowed

## Development vs Production

The flow is the same in both development and production environments. The key differences:

- **Development**: Uses local email preview (no actual emails sent by default)
- **Production**: Sends real invitation emails via your configured email service

## Configuration

B2B mode is controlled by the `FS_APP_MODE` environment variable:

```bash
# Backend (.env)
FS_APP_MODE=b2b

# Frontend (.env)
PUBLIC_APP_MODE=b2b
```

To switch modes, update these variables and restart your services.

## Common Workflows

### Adding a New Company

1. sys_admin creates organization
2. Organization admin accepts invitation
3. org_admin invites team members
4. Members accept invitations and join

### Managing Organization Members

1. org_admin views members at `/organization/users`
2. Change member roles with the "Change Role" button
3. Remove members with the "Remove" button
4. Cannot remove yourself or the last org_admin

### Suspending an Organization

1. sys_admin navigates to organization details
2. Clicks "Suspend Organization"
3. All users in the organization are blocked from logging in
4. Data is preserved and can be reactivated later
