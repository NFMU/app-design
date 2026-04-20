# Collaboration Foundation Spec

## Canonical Tables

- `plans`
- `tenants`
- `tenant_members`
- `invitations`

## Structural Rules

- `plans` defines commercial or capability limits for a tenant.
- `tenants` is the persisted collaboration boundary for Phase 1.
- `tenant_members` is the authoritative membership record for a user inside `tenants`.
- `invitations` is a persisted record for onboarding and optional scoped channel access.

## Suggested Core Attributes

`plans`
- `id`
- `code`
- `name`
- `description`
- `max_members`
- `max_channels`
- `max_storage_gb`
- `features_json`
- `status`

`tenants`
- `id`
- `uuid`
- `plan_id`
- `owner_user_id`
- `name`
- `slug`
- `domain`
- `status`
- `timezone`
- `locale`
- `branding_json`
- `settings_json`
- `activated_at`
- `suspended_at`

`tenant_members`
- `id`
- `tenant_id`
- `user_id`
- `invited_by`
- `status`
- `joined_at`
- `left_at`
- `removed_at`

`invitations`
- `id`
- `tenant_id`
- `channel_id`
- `invited_by`
- `accepted_by_user_id`
- `email`
- `invite_type`
- `token`
- `role_scope`
- `role_code`
- `status`
- `expires_at`
- `accepted_at`

## Constraint Hints

- every `tenants` row belongs to exactly one `plans` row in Phase 1
- `tenant_members` belongs to exactly one `tenants` row and one `users` row
- `invitations.channel_id` is optional because some invitations target the whole collaboration boundary
- `accepted_by_user_id` stays nullable until the invite is completed
- `tenants.slug` should be unique in the current deployment scope

## Technical Simplification

The collaboration foundation intentionally does not introduce a separate `workspaces` table in Phase 1.
Business flows that refer to workspace creation or workspace administration map onto the `tenants` boundary and its settings payloads.
