# Channels Service

## Status

**Planned.** No source code exists yet under `services/`. This document is the target architecture; spec authority remains `docs/analysis/specs/03_channel_management.md`.

## Responsibility

The channels service is the authority for **discussion spaces inside a tenant**: creation, membership, channel-local user preferences (pinning, notification level), and channel settings.

It does **not** own messages. Messages live in the `messaging` service and reference `channel_id`. It does **not** own role definitions; channel-level role assignments are stored here in `channel_member_roles` once `rbac` is online, but the role catalog lives in `rbac`.

## Owned Tables

| Table | PK type | Purpose |
|---|---|---|
| `channels` | `integer` | The channel itself. Public ID is `channels.uuid`. Belongs to one tenant. |
| `channel_members` | `integer` | Membership of a user in a channel. Stores per-user preferences (`is_pinned`, `notification_level`, `display_order`). |

Full column definitions: `docs/analysis/specs/03_channel_management.md`.

## Module Layout (Target)

```
services/channels/src/
├── app.module.ts
├── main.ts
├── core/                            # decorators, errors, filters, interceptors
├── infrastructure/
│   ├── clients/
│   │   ├── identity.client.ts       # validate user IDs
│   │   ├── tenants.client.ts        # validate tenant + tenant membership
│   │   └── rbac.client.ts           # role assignment on channel create/join
│   ├── database/database.module.ts
│   └── orms/
│       ├── channel.orm.ts
│       └── channel-member.orm.ts
└── modules/
    ├── channels/                    # CRUD, archive
    └── memberships/                 # Add/remove member, leave, preferences
```

## Planned Public API Surface

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/tenants/{tenantUuid}/channels` | Bearer + tenant member | Create a channel; caller becomes creator + first member. |
| GET | `/tenants/{tenantUuid}/channels` | Bearer + tenant member | List channels visible to caller. |
| GET | `/channels/{channelUuid}` | Bearer + channel member or tenant admin (for public channels) | Channel detail. |
| PATCH | `/channels/{channelUuid}` | Bearer + `channel.manage` | Update name, slug, topic, settings. |
| POST | `/channels/{channelUuid}/archive` | Bearer + `channel.manage` | Archive (soft) the channel. |
| POST | `/channels/{channelUuid}/restore` | Bearer + `channel.manage` | Restore an archived channel. |
| GET | `/channels/{channelUuid}/members` | Bearer + channel member | List members. |
| POST | `/channels/{channelUuid}/members` | Bearer + `channel.members.add` | Add a tenant member to this channel. |
| DELETE | `/channels/{channelUuid}/members/{memberId}` | Bearer + `channel.members.remove` | Remove a member. |
| POST | `/channels/{channelUuid}/members/leave` | Bearer | Self-leave. |
| PATCH | `/channels/{channelUuid}/me/preferences` | Bearer + channel member | Update `is_pinned`, `notification_level`, `display_order`. |

Full request/response shapes: `docs/technical/api/04_channel_api.md`.

## Internal API Surface (Service-to-Service)

| Method | Path | Purpose |
|---|---|---|
| GET | `/internal/channels/{id}` | Validate channel exists (used by `messaging` and `tenants`). |
| GET | `/internal/channels/{id}/members?user_id=X` | Verify a user is an active channel member (used by `messaging` to enforce posting permissions). |
| GET | `/internal/channels?ids=1,2,3` | Bulk lookup for hydrating channel summaries in other services. |

## Inter-Service Dependencies

**Outbound:**

| Target | Endpoint(s) | When |
|---|---|---|
| `identity` | `GET /internal/users/{id}` | Validate `created_by`, `user_id`, `added_by` exist. |
| `tenants` | `GET /internal/tenants/{id}/exists` | Confirm the tenant exists before creating a channel. |
| `tenants` | `GET /internal/tenant-members?user_id=X&tenant_id=Y` | Confirm user is an active tenant member before creating channel membership. |
| `rbac` (planned) | `POST /internal/role-assignments` | Assign default channel role on channel create or member add. |

**Inbound:**

| Caller | Endpoint(s) | When |
|---|---|---|
| `messaging` | `GET /internal/channels/{id}` | Resolve channel context when a message is posted. |
| `messaging` | `GET /internal/channels/{id}/members?user_id=X` | Permission check on message send. |
| `tenants` | `GET /internal/channels?ids=...` | Hydrate invitation responses that target a specific channel. |

## Cross-Service Reference Columns

| Column | Owning service / table | Validation strategy |
|---|---|---|
| `channels.tenant_id` | `tenants.tenants` (integer) | Validate via `/internal/tenants/{id}/exists` on create. |
| `channels.created_by` | `identity.users` (uuid) | Trusted (set from authenticated caller). |
| `channel_members.user_id` | `identity.users` (uuid) | Validate user is also active in `tenant_members` before insert. |
| `channel_members.added_by` | `identity.users` (uuid) | Trusted. |

## Authentication Model

Same as `tenants`: verify user JWT locally using shared `JWT_SECRET`. For permission checks the service should:

1. Resolve `channel_member` row for the calling user.
2. Resolve assigned `channel_member_roles` (via local cache or `rbac` lookup).
3. Compare against the required permission code (e.g. `channel.messages.delete`).

## Channel Type Rules

`channels.channel_type` distinguishes how membership is enforced:

| Type | Membership rules |
|---|---|
| `public` | Visible to all tenant members. Joining does not require an invitation. |
| `private` | Visible only to current `channel_members`. Membership requires explicit add. |
| `direct` | 1:1 or small-group DM. Generated `name` and `slug`; member ceiling lives in `settings_json.max_members`. |

## Performance & Scale Notes

- `channels` is read-heavy (channel list on every page load). Add Redis cache keyed by `tenant_id` with 60-second TTL.
- `channel_members` queries by `(channel_id, user_id)` are common; the composite unique index serves both reads and write conflicts.
- Sidebar ordering uses `(is_pinned DESC, display_order ASC, last_message_at DESC)`. The last column requires a denormalized `last_message_at` updated by `messaging` via webhook or batch — defer until volume justifies it.
