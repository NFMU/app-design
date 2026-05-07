# Channel Management Spec

## Canonical Tables

- `channels`
- `channel_members`

## Structural Rules

- `channels` belongs to exactly one `tenants` record.
- `channel_members` is the persisted participation record between a user and a channel.
- Channel-local preferences (pinning, notification level, display order) belong to `channel_members`, not `channels`.
- Channel administration is implemented through RBAC assignments; no hard-coded role column exists on `channels`.
- A user must be an active `tenant_members` row before they can hold a `channel_members` row in the same tenant.

---

## Table Definitions

### `channels`

Discussion space inside a tenant. Each row is one channel.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Internal surrogate key. Used for all FK relations. |
| `uuid` | `uuid` | NOT NULL, unique | Public identifier exposed in API responses. Never reused. |
| `tenant_id` | `bigint` | NOT NULL, FK → `tenants.id` CASCADE, index | Owning tenant. Channels cannot exist without a tenant. |
| `created_by` | `bigint` | NOT NULL, FK → `users.id` RESTRICT | The user who created this channel. Must also have a `channel_members` row after creation. |
| `name` | `varchar(255)` | NOT NULL | Display name of the channel, e.g. `"general"`, `"engineering-backend"`. |
| `slug` | `varchar(100)` | NOT NULL | URL-safe channel identifier. Unique within the owning tenant scope. |
| `channel_type` | `varchar(20)` | NOT NULL, default `'public'` | Access model: `public` \| `private` \| `direct`. `direct` channels are 1-to-1 or small-group DMs. |
| `description` | `text` | nullable | Optional longer description shown on the channel detail page. |
| `topic` | `varchar(500)` | nullable | Short, current topic pinned in the channel header. |
| `avatar_url` | `varchar(500)` | nullable | CDN URL for the channel icon. Null uses a generated icon. |
| `status` | `varchar(20)` | NOT NULL, default `'active'` | Lifecycle state: `active` \| `archived` \| `deleted`. |
| `settings_json` | `jsonb` | NOT NULL, default `'{}'` | Channel-level policies merged from requirements: post permissions, threading mode, retention policy, slow-mode interval. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |
| `deleted_at` | `timestamptz` | nullable | Soft-delete marker. Non-null means the channel is deleted. |

**Indexes:** unique on `uuid`, unique on `(tenant_id, slug)`, index on `tenant_id`

**`channel_type` values:** `public` | `private` | `direct`

**Status values:** `active` | `archived` | `deleted`

**`settings_json` shape (example):**
```json
{
  "post_permission": "members",
  "threading_enabled": true,
  "message_retention_days": 365,
  "slow_mode_seconds": 0,
  "allow_reactions": true
}
```

---

### `channel_members`

Persisted participation record for a user inside a channel.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `channel_id` | `bigint` | NOT NULL, FK → `channels.id` CASCADE, index | The channel this row belongs to. |
| `user_id` | `bigint` | NOT NULL, FK → `users.id` RESTRICT, index | The participating user. |
| `added_by` | `bigint` | nullable, FK → `users.id` SET NULL | The user who added this member. Null when the user joined voluntarily or is the creator. |
| `status` | `varchar(20)` | NOT NULL, default `'active'` | Membership state: `active` \| `left` \| `removed`. |
| `notification_level` | `varchar(20)` | NOT NULL, default `'all'` | Per-channel notification preference: `all` \| `mentions` \| `none`. Overrides the global default for this channel. |
| `is_pinned` | `boolean` | NOT NULL, default `false` | Whether the user has pinned this channel to the top of their sidebar. |
| `display_order` | `integer` | NOT NULL, default `0` | User-controlled sort position within the pinned or recent channel list. |
| `joined_at` | `timestamptz` | nullable | Set when the user first becomes an active member of this channel. |
| `left_at` | `timestamptz` | nullable | Set when the user leaves or is removed. Combines the intent of `left_at` and `removed_at` into one timestamp; the `status` field distinguishes voluntary leave from admin removal. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** composite unique on `(channel_id, user_id)` for active rows, index on `channel_id`, index on `user_id`

**Status values:** `active` | `left` | `removed`

**`notification_level` values:** `all` | `mentions` | `none`

---

## Constraint Summary

| Rule | Details |
|---|---|
| `channels.slug` unique per tenant | Enforced by composite unique index `(tenant_id, slug)`. |
| Channel creator also becomes a member | Application service inserts both `channels` and `channel_members` in the same transaction. |
| No duplicate active membership | Composite unique on `(channel_id, user_id)` for active rows. Historical rows are retained. |
| Tenant membership precedes channel membership | Application services must verify `tenant_members` before inserting `channel_members`. |
| `channel_type = 'direct'` restrictions | DM channels typically set `name` to a generated value and enforce a membership ceiling of 2–8 users via `settings_json.max_members`. |

## Modeling Note

Channel rules and local administration are stored in `channels.settings_json` plus RBAC assignments in Phase 1.
A separate `channel_rules` table is not introduced unless query or versioning requirements justify it in a later phase.
