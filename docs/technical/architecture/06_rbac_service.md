# RBAC Service

## Status

**Planned.** No source code exists yet under `services/`. Spec authority remains `docs/analysis/specs/05_authorization_model.md`.

## Responsibility

The RBAC service is the authority for **permission catalog, role definitions, and role assignments** across the platform. Other services delegate "can this user do X" decisions to RBAC, either via inline lookup or by caching a per-user permission set.

It does **not** own membership rows. `tenant_members` and `channel_members` are owned by their respective services. RBAC stores **role assignments** that reference those membership rows by ID.

It does **not** enforce permissions itself — enforcement happens at the calling service's controller / guard. RBAC just answers questions.

## Owned Tables

| Table | PK type | Purpose |
|---|---|---|
| `roles` | `integer` | Assignable bundle of permissions. Scoped to platform / tenant / channel / self. System roles are seeded; tenants may define custom roles. |
| `permissions` | `integer` | Individual permission codes (e.g. `channel.messages.delete`). Platform-seeded; tenants do not edit. |
| `role_permissions` | `integer` | M:N link between roles and permissions. |
| `tenant_member_roles` | `integer` | Role assignments at tenant scope. References `tenant_members.id`. |
| `channel_member_roles` | `integer` | Role assignments at channel scope. References `channel_members.id`. |

Full column definitions: `docs/analysis/specs/05_authorization_model.md`.

## Module Layout (Target)

```
services/rbac/src/
├── app.module.ts
├── main.ts
├── core/
├── infrastructure/
│   ├── clients/
│   │   ├── identity.client.ts
│   │   ├── tenants.client.ts
│   │   └── channels.client.ts
│   ├── database/database.module.ts
│   └── orms/
│       ├── role.orm.ts
│       ├── permission.orm.ts
│       ├── role-permission.orm.ts
│       ├── tenant-member-role.orm.ts
│       └── channel-member-role.orm.ts
└── modules/
    ├── permissions/                 # Read-only catalog
    ├── roles/                       # CRUD for tenant-scope custom roles
    ├── assignments/                 # Assign / revoke roles
    └── checks/                      # Permission-check API for other services
```

## Planned Public API Surface

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/permissions` | Bearer | List the permission catalog. |
| GET | `/tenants/{tenantUuid}/roles` | Bearer + `tenant.roles.read` | List system + custom tenant roles. |
| POST | `/tenants/{tenantUuid}/roles` | Bearer + `tenant.roles.manage` | Create a custom tenant role. |
| PATCH | `/roles/{id}` | Bearer + `tenant.roles.manage` | Update name, description, attached permissions. System roles are read-only. |
| DELETE | `/roles/{id}` | Bearer + `tenant.roles.manage` | Delete a custom role. |
| POST | `/tenant-members/{memberId}/roles` | Bearer + `tenant.members.manage_roles` | Assign a role to a tenant member. |
| DELETE | `/tenant-member-roles/{id}` | Bearer + `tenant.members.manage_roles` | Revoke (sets `revoked_at`). |
| POST | `/channel-members/{memberId}/roles` | Bearer + `channel.members.manage_roles` | Assign a channel-scope role. |
| DELETE | `/channel-member-roles/{id}` | Bearer + `channel.members.manage_roles` | Revoke. |

Full request/response shapes: `docs/technical/api/06_rbac_api.md`.

## Internal API Surface

This is the **most-called** internal API in the platform. Latency and caching are critical.

| Method | Path | Purpose |
|---|---|---|
| GET | `/internal/roles/lookup?code=X&scope=tenant` | Resolve a role by code at the given scope (used during invitation accept). |
| GET | `/internal/permissions/check` | Single permission check: `?user_id=X&permission=Y&tenant_id=Z` or `&channel_id=Z`. Returns `{ allowed: bool }`. |
| POST | `/internal/permissions/bulk-check` | Batch of permission checks for a single user, returns a `{ permission: bool }` map. |
| GET | `/internal/users/{id}/permissions?tenant_id=X[&channel_id=Y]` | Full permission set for a user in a context. Suitable for caching on the calling service. |
| POST | `/internal/role-assignments` | Programmatic role assignment used by `tenants` (default tenant role on invitation accept) and `channels` (default channel role on member add). |
| GET | `/internal/seeds/initialize?tenant_id=X` | One-time tenant role seeding (run after `POST /tenants`). Inserts `tenant_admin`, `member`, etc. |

Internal endpoints support a short-lived cache header (`Cache-Control: max-age=60`). Callers should cache user permission sets in memory for the duration of a request, never longer.

## Inter-Service Dependencies

**Outbound:**

| Target | Endpoint(s) | When |
|---|---|---|
| `identity` | `GET /internal/users/{id}` | Validate `assigned_by`, `revoked_by` exist. |
| `tenants` | `GET /internal/tenant-members/{id}` | Validate `tenant_member_id` before role assignment. |
| `channels` | `GET /internal/channel-members/{id}` | Validate `channel_member_id` before role assignment. |

**Inbound:**

| Caller | Endpoint(s) | When |
|---|---|---|
| All services | `GET /internal/permissions/check` | Per-request permission gate. |
| All services | `GET /internal/users/{id}/permissions` | Bulk load on context entry. |
| `tenants` | `POST /internal/role-assignments` | Auto-assign `member` on invitation accept. |
| `tenants` | `GET /internal/seeds/initialize` | Seed default roles after `POST /tenants`. |
| `channels` | `POST /internal/role-assignments` | Auto-assign default channel role on creator + member add. |

## Cross-Service Reference Columns

| Column | Owning service / table | Validation strategy |
|---|---|---|
| `roles.tenant_id` | `tenants.tenants` (integer, nullable for platform roles) | Validate via `/internal/tenants/{id}/exists` on custom role create. |
| `tenant_member_roles.tenant_member_id` | `tenants.tenant_members` (integer) | Validate via `/internal/tenant-members/{id}`. |
| `tenant_member_roles.assigned_by`, `revoked_by` | `identity.users` (uuid) | Trusted (from authenticated caller). |
| `channel_member_roles.channel_member_id` | `channels.channel_members` (integer) | Validate via `/internal/channel-members/{id}`. |
| `channel_member_roles.assigned_by`, `revoked_by` | `identity.users` (uuid) | Trusted. |

## Permission Code Conventions

Dotted, scope-prefixed strings:

- `platform.*` — global, e.g. `platform.billing.prices.manage`.
- `tenant.*` — tenant-scoped, e.g. `tenant.members.invite`, `tenant.billing.profile.manage`.
- `channel.*` — channel-scoped, e.g. `channel.messages.delete`, `channel.members.add`.
- `self.*` — applies to the caller's own resources, e.g. `self.profile.update`.

A check at narrower scope automatically passes if the caller has the broader scope's permission. The check service implements this hierarchy so callers do not have to.

## Seeded Catalog

The migration that creates `roles` and `permissions` should also seed the canonical catalog:

| Role | Scope | `tenant_id` | Auto-seeded permissions |
|---|---|---|---|
| `super_admin` | `platform` | NULL | All `platform.*` codes. |
| `tenant_admin` | `tenant` | per tenant | All `tenant.*` codes except `tenant.billing.*` (granted separately). |
| `billing_admin` | `tenant` | per tenant | All `tenant.billing.*` codes. |
| `member` | `tenant` | per tenant | `tenant.entitlements.read`, `self.*`. |
| `channel_moderator` | `channel` | per tenant (template) | `channel.messages.delete`, `channel.messages.pin`, `channel.members.remove`. |

Tenant-scoped roles (`tenant_admin`, `member`, `billing_admin`) are seeded by `tenants` calling `GET /internal/seeds/initialize?tenant_id=X` after a tenant is created.

## Performance & Scale Notes

- **Hot path is permission checks.** Cache the full `(user_id, tenant_id) → permissions[]` map in Redis with a 60-second TTL, invalidated on role assignment changes.
- **The `permissions` catalog is small (<200 rows) and immutable per deploy.** Load entirely into memory at startup.
- **`role_permissions` joins are common.** Materialize a flat `(role_id, permission_code)` view for read-time lookups.
