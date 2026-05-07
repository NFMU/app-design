# Collaboration Foundation Spec

## Canonical Tables

- `plans`
- `tenants`
- `tenant_members`
- `invitations`

## Structural Rules

- `plans` defines commercial or capability limits for a tenant.
- `plans` remains the product and entitlement catalog; subscription billing details are modeled separately in the billing spec.
- `tenants` is the persisted collaboration boundary for Phase 1.
- `tenant_members` is the authoritative membership record for a user inside `tenants`.
- `invitations` is a persisted record for onboarding and optional scoped channel access.

---

## Table Definitions

### `plans`

System-managed catalog of product tiers. Seeded by platform administrators.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `code` | `varchar(50)` | NOT NULL, unique | Stable machine identifier, e.g. `free`, `starter`, `pro`, `enterprise`. Used in code and API references. |
| `name` | `varchar(120)` | NOT NULL | Human-readable plan display name, e.g. `"Free"`, `"Pro"`. |
| `description` | `text` | nullable | Marketing description of the plan shown in UI. |
| `max_members` | `integer` | nullable | Maximum number of active `tenant_members` rows allowed. `NULL` means unlimited. |
| `max_channels` | `integer` | nullable | Maximum number of active `channels` rows allowed per tenant. `NULL` means unlimited. |
| `max_storage_gb` | `numeric(10,2)` | nullable | Total file storage quota in GB. `NULL` means unlimited. |
| `features_json` | `jsonb` | NOT NULL, default `'{}'` | Feature flags and capability overrides as key-value pairs, e.g. `{"guest_access": true, "sso": false}`. |
| `status` | `varchar(20)` | NOT NULL, default `'active'` | Lifecycle state: `active` \| `deprecated` \| `hidden`. Deprecated plans can be retained by existing tenants but not selected by new ones. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** unique on `code`

**Valid status values:** `active` | `deprecated` | `hidden`

---

### `tenants`

Core collaboration boundary. Each row represents one organization workspace in Phase 1.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Internal surrogate key. Used for all FK relations within the system. |
| `uuid` | `uuid` | NOT NULL, unique | Stable public identifier exposed in APIs and URLs. Never reused after deletion. |
| `plan_id` | `bigint` | NOT NULL, FK → `plans.id` RESTRICT | Current entitlement plan. May be synchronized from the active `subscriptions` row when billing is enabled. |
| `owner_user_id` | `bigint` | NOT NULL, FK → `users.id` RESTRICT | The user who created the tenant and holds the initial owner role. Must also have an active `tenant_members` row. |
| `name` | `varchar(255)` | NOT NULL | Display name of the tenant organization, e.g. `"Acme Corp"`. |
| `slug` | `varchar(100)` | NOT NULL, unique within deployment | URL-safe identifier used in routing, e.g. `acme-corp`. Unique constraint scoped to the deployment domain. |
| `domain` | `varchar(255)` | nullable, unique | Custom domain for SSO auto-enrollment, e.g. `acme.com`. Null means no domain restriction. |
| `status` | `varchar(20)` | NOT NULL, default `'active'` | Tenant lifecycle state: `active` \| `suspended` \| `deleted`. |
| `timezone` | `varchar(100)` | NOT NULL, default `'UTC'` | IANA timezone for the organization (used for display and scheduled notifications). |
| `locale` | `varchar(20)` | NOT NULL, default `'en'` | Default BCP 47 locale for members who have not set a personal preference. |
| `branding_json` | `jsonb` | NOT NULL, default `'{}'` | Visual identity overrides: `logo_url`, primary `color`, and `theme`. |
| `settings_json` | `jsonb` | NOT NULL, default `'{}'` | Operational policies: message retention days, guest access toggle, file-sharing rules, SSO configuration. |
| `activated_at` | `timestamptz` | nullable | Set when the tenant first becomes usable (e.g. after email verification of the owner). |
| `suspended_at` | `timestamptz` | nullable | Set when the tenant is suspended due to non-payment or policy violation. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |
| `deleted_at` | `timestamptz` | nullable | Soft-delete marker. Non-null means the tenant is deleted. |

**Indexes:** unique on `uuid`, unique on `slug`, unique on `domain` (sparse — null excluded)

**Status values:** `active` | `suspended` | `deleted`

**`branding_json` shape (example):**
```json
{
  "logo_url": "https://cdn.example.com/logos/acme.png",
  "color": "#1a73e8",
  "theme": "light"
}
```

**`settings_json` shape (example):**
```json
{
  "message_retention_days": 90,
  "guest_access": false,
  "file_sharing_enabled": true,
  "sso_provider": null
}
```

---

### `tenant_members`

Authoritative membership record between a `users` row and a `tenants` row.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `tenant_id` | `bigint` | NOT NULL, FK → `tenants.id` CASCADE, index | Owning tenant. |
| `user_id` | `bigint` | NOT NULL, FK → `users.id` RESTRICT, index | Member user. Composite unique with `tenant_id` ensures no duplicate active membership. |
| `invited_by` | `bigint` | nullable, FK → `users.id` SET NULL | The user who sent the invitation that resulted in this membership. Null for the tenant owner. |
| `status` | `varchar(20)` | NOT NULL, default `'active'` | Membership state: `active` \| `left` \| `removed` \| `suspended`. |
| `joined_at` | `timestamptz` | nullable | Set when the user first accepts the invitation and becomes an active member. |
| `left_at` | `timestamptz` | nullable | Set when the member voluntarily leaves the tenant. |
| `removed_at` | `timestamptz` | nullable | Set when an administrator removes the member. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** composite unique on `(tenant_id, user_id)` for active rows, index on `tenant_id`, index on `user_id`

**Status values:** `active` | `left` | `removed` | `suspended`

**Note:** A user may have at most one active membership per tenant. Historical rows (left/removed) are retained for audit purposes.

---

### `invitations`

Persisted record of an invitation to join a tenant, optionally pre-scoped to a specific channel.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `tenant_id` | `bigint` | NOT NULL, FK → `tenants.id` CASCADE | The tenant the invitation grants access to. |
| `channel_id` | `bigint` | nullable, FK → `channels.id` SET NULL | If set, the invitee is also enrolled in this specific channel upon acceptance. |
| `invited_by` | `bigint` | NOT NULL, FK → `users.id` RESTRICT | The user who created the invitation. |
| `accepted_by_user_id` | `bigint` | nullable, FK → `users.id` SET NULL | The user who accepted the invitation. Null until the invite is completed. |
| `email` | `varchar(255)` | NOT NULL | Target email address. The system matches against `users.email` on acceptance. |
| `invite_type` | `varchar(20)` | NOT NULL, default `'email'` | Delivery method: `email` \| `link`. A `link` invite can be accepted by any user who has the token. |
| `token` | `varchar(255)` | NOT NULL, unique | Opaque one-time token embedded in the invitation URL. |
| `role_scope` | `varchar(20)` | NOT NULL, default `'tenant'` | The RBAC scope for the default role assigned on acceptance: `tenant` \| `channel`. |
| `role_code` | `varchar(50)` | NOT NULL | The role code to assign on acceptance, e.g. `member`, `admin`. Must exist in `roles`. |
| `status` | `varchar(20)` | NOT NULL, default `'pending'` | Lifecycle state: `pending` \| `accepted` \| `expired` \| `revoked`. |
| `expires_at` | `timestamptz` | NOT NULL | After this time the token is no longer valid even if unused. |
| `accepted_at` | `timestamptz` | nullable | Set when the invitation is successfully accepted. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** unique on `token`, index on `(tenant_id, email)`, index on `tenant_id`

**Status values:** `pending` | `accepted` | `expired` | `revoked`

**`invite_type` behavior:**
- `email`: Only the specific email address may accept.
- `link`: Any authenticated user with the token URL may accept (useful for open enrollment periods).

---

## Constraint Summary

| Rule | Details |
|---|---|
| Every `tenants` row belongs to exactly one `plans` row | `tenants.plan_id` is NOT NULL. |
| `tenants.slug` is unique | Scoped to the current deployment namespace. |
| `tenants.domain` is unique when set | Prevents two tenants claiming the same domain. |
| No duplicate active membership | Composite unique on `(tenant_id, user_id)` for active rows. |
| `invitations.token` is unique | Prevents token collision and replay attacks. |
| `invitations.channel_id` is optional | Tenant-only invitations leave `channel_id` null. |
| `accepted_by_user_id` is null until accepted | Updated atomically when the invitation is consumed. |

## Technical Simplification

The collaboration foundation intentionally does not introduce a separate `workspaces` table in Phase 1.
Business flows that refer to workspace creation or workspace administration map onto the `tenants` boundary and its settings payloads.
Payment and subscription lifecycle records must remain outside `tenants`; billing state can restrict or suspend access without changing the tenant's collaboration identity.

Tenant API responses may expose a read-only entitlement summary derived from billing records, but payment state is not owned by the `tenants` table.
