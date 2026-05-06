# Authorization Model Spec

## Canonical Tables

- `roles`
- `permissions`
- `role_permissions`
- `tenant_member_roles`
- `channel_member_roles`

## Structural Rules

- `roles` is the assignable bundle of permissions.
- `permissions` defines reusable permission codes grouped by scope.
- `role_permissions` is the linking table between roles and permissions.
- `tenant_member_roles` assigns roles at the collaboration-boundary level.
- `channel_member_roles` assigns roles at the channel level.
- Billing administration is modeled as tenant-scoped permissions and roles; no separate billing role-assignment table is introduced in Phase 1.

## Suggested Core Attributes

`roles`
- `id`
- `tenant_id`
- `scope_type`
- `code`
- `name`
- `description`
- `is_system`

`permissions`
- `id`
- `scope_type`
- `code`
- `name`
- `description`

`role_permissions`
- `id`
- `role_id`
- `permission_id`

`tenant_member_roles`
- `id`
- `tenant_member_id`
- `role_id`
- `assigned_by`
- `assigned_at`

`channel_member_roles`
- `id`
- `channel_member_id`
- `role_id`
- `assigned_by`
- `assigned_at`

## Scope Rules

- `scope_type` should distinguish at least `platform`, `tenant`, `channel`, and `self`
- system roles can be represented with `is_system = true`
- `roles.tenant_id` may be nullable for platform-level system roles in the simplified Phase 1 model
- self-scope permissions are still modeled as permission codes even when they do not require a dedicated assignment table
- billing permissions use tenant scope because billing records belong to a tenant account and must be evaluated with the caller's tenant membership

## Seeded Role Guidance

- `super_admin` is a platform system role and can manage global billing configuration, prices, policies, and approved exceptions.
- `tenant_admin` is a tenant system role and can manage tenant settings, members, and billing only when it is granted billing permission codes.
- `billing_admin` is a tenant system role intended for finance or operations users who can manage billing profile, subscription, invoices, payment method references, and tenant billing overrides.
- `member` is a tenant system role and must not receive invoice, payment method, provider reference, or billing override permissions by default.

## Baseline Billing Permissions

- `platform.billing.prices.manage`
- `tenant.billing.profile.manage`
- `tenant.billing.subscription.manage`
- `tenant.billing.payment_methods.manage`
- `tenant.billing.invoices.read`
- `tenant.billing.overrides.apply`
- `tenant.entitlements.read`

## Constraint Hints

- `permissions.code` should be unique
- one role can grant many permissions
- one tenant member can hold multiple tenant-scoped roles over time
- one channel member can hold multiple channel-scoped roles over time
- role assignments must remain auditable through `assigned_by` and `assigned_at`
