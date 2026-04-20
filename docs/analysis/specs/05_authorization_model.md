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

## Constraint Hints

- `permissions.code` should be unique
- one role can grant many permissions
- one tenant member can hold multiple tenant-scoped roles over time
- one channel member can hold multiple channel-scoped roles over time
- role assignments must remain auditable through `assigned_by` and `assigned_at`
