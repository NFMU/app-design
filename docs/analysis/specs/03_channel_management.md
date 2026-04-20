# Channel Management Spec

## Canonical Tables

- `channels`
- `channel_members`

## Structural Rules

- `channels` belongs to exactly one `tenants` record.
- `channel_members` is the persisted participation record between a user and a channel.
- channel-local preferences such as pinning, notification level, and display order belong to `channel_members`.
- channel administration is implemented through RBAC assignments, not by hard-coding a separate role column on `channels`.

## Suggested Core Attributes

`channels`
- `id`
- `uuid`
- `tenant_id`
- `created_by`
- `name`
- `slug`
- `channel_type`
- `description`
- `topic`
- `avatar_url`
- `status`
- `settings_json`

`channel_members`
- `id`
- `channel_id`
- `user_id`
- `added_by`
- `status`
- `notification_level`
- `is_pinned`
- `display_order`
- `joined_at`
- `left_at`

## Constraint Hints

- channel creator should also appear in `channel_members`
- `channels.slug` should be unique within the owning tenant scope
- a user should not have duplicate active `channel_members` rows for the same channel
- application services must ensure the user already belongs to the owning tenant before granting channel membership

## Modeling Note

The requirements mention channel rules and local administration.
In Phase 1 those rules live in `channels.settings_json` plus RBAC assignments, rather than a separate `channel_rules` table.
