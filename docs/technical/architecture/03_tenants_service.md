# Tenants Service

## Status

**Skeleton.** ORM entities are defined for `plans`, `tenants`, `tenant_members`, `invitations`. No controllers, services, or migrations are wired up yet. The `app.module.ts` is empty.

Source: `d:/Workspace/Github/chat-slack/services/tenants/`.

## Responsibility

The tenants service is the authority for the **collaboration boundary**: which organizations exist, who belongs to them, what their entitlement plan is, and how new members are invited.

It does **not** know about channels, messages, or RBAC role definitions. It does record which **role code** is assigned to a member (e.g. `tenant_admin`, `member`) but the role catalog itself lives in the future `rbac` service. Plan **prices and billing lifecycle** (subscriptions, invoices, payments) live in the future `billing` service; tenants only references `plans` for entitlement.

## Owned Tables

| Table | PK type | Purpose |
|---|---|---|
| `plans` | `integer` | Entitlement catalog (limits, feature flags). One plan applies to many tenants. |
| `tenants` | `integer` | The organization workspace. Public ID is `tenants.uuid`. |
| `tenant_members` | `integer` | Membership of a user in a tenant. Tracks lifecycle (`active` / `left` / `removed` / `suspended`). |
| `invitations` | `integer` | Pending and historical invitations (email or link). May target a specific channel. |

Full column definitions: `docs/analysis/specs/02_collaboration_foundation.md`.
ORM entities: `services/tenants/src/infrastructure/orms/{plan,tenant,tenant-member,invitation}.orm.ts`.

## Module Layout

Current (skeleton):

```
services/tenants/src/
├── app.module.ts                    # currently empty
├── main.ts
├── applications/                    # CQRS handlers (skeleton)
│   └── tenants/
│       ├── commands/create-tenant.command.ts
│       └── handlers/create-tenant.handler.ts
└── infrastructure/
    ├── database/
    │   └── database.module.ts       # path mismatch; see Open Issues
    └── orms/
        ├── plan.orm.ts
        ├── tenant.orm.ts
        ├── tenant-member.orm.ts
        └── invitation.orm.ts
```

Target layout once the service is wired up — mirrors the identity service for consistency:

```
services/tenants/src/
├── app.module.ts
├── main.ts
├── core/                            # decorators, errors, filters, interceptors
├── infrastructure/
│   ├── clients/                     # IdentityClient (HTTP), other future clients
│   ├── database/database.module.ts
│   └── orms/                        # existing
└── modules/
    ├── plans/                       # Read endpoints (admin-managed catalog)
    ├── tenants/                     # Tenant CRUD, entitlement view
    ├── memberships/                 # Tenant member CRUD, leave/remove
    └── invitations/                 # Create, accept, revoke, list
```

## Planned Public API Surface

User-facing endpoints. Bearer-token auth from identity-issued JWT.

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/plans` | Bearer | List available plans. |
| GET | `/plans/{code}` | Bearer | Get plan detail. |
| POST | `/tenants` | Bearer | Create a tenant; caller becomes owner. |
| GET | `/tenants/{uuid}` | Bearer + tenant member | Get tenant detail (with hydrated timezone/language and entitlement summary). |
| PATCH | `/tenants/{uuid}` | Bearer + `tenant.manage` | Update name, slug, branding, settings, timezone, language. |
| DELETE | `/tenants/{uuid}` | Bearer + tenant owner | Soft-delete tenant. |
| GET | `/tenants/{uuid}/members` | Bearer + tenant member | List members. |
| DELETE | `/tenants/{uuid}/members/{memberId}` | Bearer + `tenant.members.remove` | Remove a member. |
| POST | `/tenants/{uuid}/members/leave` | Bearer | Self-leave the tenant. |
| POST | `/tenants/{uuid}/invitations` | Bearer + `tenant.members.invite` | Create invitation (email or link). |
| GET | `/tenants/{uuid}/invitations` | Bearer + `tenant.members.invite` | List active and historical invitations. |
| DELETE | `/tenants/{uuid}/invitations/{id}` | Bearer + `tenant.members.invite` | Revoke an invitation. |
| POST | `/invitations/accept` | Bearer | Accept an invitation by token. |

Full request/response shapes: `docs/technical/api/03_tenant_membership_api.md`.

## Inter-Service Dependencies

**Outbound:**

| Target service | Endpoint(s) | When |
|---|---|---|
| `identity` | `GET /internal/users/{id}` | Validate `owner_user_id`, `accepted_by_user_id` exists before persisting. |
| `identity` | `GET /internal/reference/timezones/{uuid}` | Validate `timezone_id` (uuid value) on tenant create/update. |
| `identity` | `GET /internal/reference/languages/{uuid}` | Validate `language_id` (uuid value) on tenant create/update. |
| `identity` | `GET /internal/reference/{type}` (bulk) | Snapshot cache for hydration of tenant responses. |
| `rbac` (planned) | `GET /internal/roles/lookup?code=X&scope=tenant` | Resolve role code at invitation accept; insert tenant_member_role. |
| `billing` (planned) | `GET /internal/tenants/{uuid}/entitlement-summary` | Compose entitlement view returned from `GET /tenants/{uuid}`. |

**Inbound:**

| Caller | Endpoint(s) | When |
|---|---|---|
| `channels` (planned) | `GET /internal/tenants/{id}/exists` | Verify a tenant before creating a channel. |
| `channels` (planned) | `GET /internal/tenant-members?user_id=X&tenant_id=Y` | Verify the user is an active member before creating a channel membership. |
| `billing` (planned) | `GET /internal/tenants/{id}` | Fetch tenant metadata when generating invoices. |

## Cross-Service Reference Columns

| Column | Owning service / table | Validation strategy |
|---|---|---|
| `tenants.owner_user_id` | `identity.users` (uuid) | Validate-on-write via `/internal/users/{id}`. |
| `tenants.timezone_id` | `identity.timezones.uuid` (uuid) | Validate-on-write via `/internal/reference/timezones/{uuid}`. |
| `tenants.language_id` | `identity.languages.uuid` (uuid) | Validate-on-write via `/internal/reference/languages/{uuid}`. |
| `tenant_members.user_id` | `identity.users` (uuid) | Validate-on-write at invitation accept. |
| `tenant_members.invited_by` | `identity.users` (uuid) | Trusted (set from authenticated caller). |
| `invitations.invited_by` | `identity.users` (uuid) | Trusted. |
| `invitations.accepted_by_user_id` | `identity.users` (uuid) | Trusted. |
| `invitations.channel_id` | `channels.channels` (integer, planned) | Validate-on-write once channels service exists. Today: nullable, no validation. |

## Authentication Model

Tenants accepts user-issued JWTs from `identity` and verifies them locally using the shared `JWT_SECRET`. The token payload yields the calling `userId` (uuid). For tenant-scoped permission checks, the service either:

1. Calls `rbac` to resolve the user's roles for the target tenant, or
2. Caches a per-request lookup of `tenant_member_roles` after the rbac service is online.

Until `rbac` exists, treat `tenants.owner_user_id == auth.userId` as `tenant_admin`.

## Migrations

None yet. The first migration will need to create `plans`, `tenants`, `tenant_members`, `invitations` plus enum types (`tenant_status`, `tenant_member_status`, `invitation_status`, `invitation_type`, `invitation_role_scope`, `plan_status`). Generate via:

```bash
pnpm --filter tenants typeorm migration:generate ./src/infrastructure/database/migrations/init
```

(after fixing the database module path — see Open Issues).

## Open Issues

1. **Database module entity path mismatch.** `src/infrastructure/database/database.module.ts` configures `entities: [path.join(__dirname, "entities/**/*.entity.{ts,js}")]` but ORMs live at `../orms/*.orm.ts`. Fix: change the glob to `path.join(__dirname, "../orms/**/*.orm.{ts,js}")` before running migrations.

2. **`app.module.ts` is empty.** Needs to import `ConfigModule`, `JwtModule` (for verifying identity-issued tokens), `DatabaseModule`, and the planned feature modules.

3. **No `core/` folder yet.** Decorators, error catalog, filters, and the JWT auth guard need to be ported from identity (or moved to a shared `@xlr8-nest/*` package).

4. **No HTTP client to identity.** Wrap `axios` or `@nestjs/axios` in an `IdentityClient` injectable that handles `X-Service-Token` headers and reference caching.

## Operational Notes

- **Database:** Postgres. Schema is service-private; no shared tables with identity.
- **Cross-service caching:** reference data (languages, timezones) should be loaded into memory at startup with a 5-minute TTL refresh. Validation of unknown IDs falls back to `/internal/reference/{type}/{id}`.
