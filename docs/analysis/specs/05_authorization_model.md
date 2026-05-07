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
- `role_permissions` is the M-to-M linking table between roles and permissions.
- `tenant_member_roles` assigns roles at the collaboration-boundary level.
- `channel_member_roles` assigns roles at the channel level.
- Billing administration is modeled as tenant-scoped permissions and roles; no separate billing role-assignment table is introduced in Phase 1.
- System roles (`is_system = true`) are seeded at deployment and must not be deleted by tenant administrators.

---

## Table Definitions

### `roles`

Assignable role definition. A role bundles one or more permissions under a named identity.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `tenant_id` | `bigint` | nullable, FK → `tenants.id` CASCADE, index | Owning tenant for tenant-scoped roles. `NULL` for platform-level system roles (e.g. `super_admin`) that apply globally. |
| `scope_type` | `varchar(20)` | NOT NULL | The RBAC layer this role applies to: `platform` \| `tenant` \| `channel` \| `self`. |
| `code` | `varchar(50)` | NOT NULL | Stable machine identifier for the role, e.g. `super_admin`, `tenant_admin`, `billing_admin`, `member`, `channel_moderator`. Unique within `(tenant_id, scope_type)`. |
| `name` | `varchar(120)` | NOT NULL | Human-readable label, e.g. `"Tenant Administrator"`. |
| `description` | `text` | nullable | Explanation of what this role grants, shown in the admin UI. |
| `is_system` | `boolean` | NOT NULL, default `false` | `true` for platform-seeded roles that cannot be deleted or fully edited by tenant admins. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** composite unique on `(tenant_id, scope_type, code)`, index on `tenant_id`

**`scope_type` values:** `platform` | `tenant` | `channel` | `self`

**Seeded system roles:**
| `code` | `scope_type` | `tenant_id` | Purpose |
|---|---|---|---|
| `super_admin` | `platform` | NULL | Global billing config, prices, policies, approved exceptions. |
| `tenant_admin` | `tenant` | seeded per tenant | Tenant settings, members, and billing (when granted billing codes). |
| `billing_admin` | `tenant` | seeded per tenant | Finance/ops: billing profile, subscription, invoices, payment methods, overrides. |
| `member` | `tenant` | seeded per tenant | Default tenant membership with no elevated permissions. |
| `channel_moderator` | `channel` | seeded per tenant | Channel message moderation, pin, and member management. |

---

### `permissions`

Individual capability definition. Permissions are platform-seeded and not editable by tenants.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `scope_type` | `varchar(20)` | NOT NULL | The RBAC layer this permission applies to: `platform` \| `tenant` \| `channel` \| `self`. |
| `code` | `varchar(100)` | NOT NULL, unique | Fully qualified dot-separated permission code, e.g. `tenant.members.invite`, `channel.messages.delete`, `platform.billing.prices.manage`. |
| `name` | `varchar(120)` | NOT NULL | Human-readable label, e.g. `"Invite Members"`. |
| `description` | `text` | nullable | Explanation of what this permission allows. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |

**Indexes:** unique on `code`

**Baseline billing permission codes:**

| Code | Scope | Description |
|---|---|---|
| `platform.billing.prices.manage` | platform | Create and deprecate plan prices. |
| `tenant.billing.profile.manage` | tenant | Update billing contact, tax ID, and address. |
| `tenant.billing.subscription.manage` | tenant | Start, change, and cancel subscriptions. |
| `tenant.billing.payment_methods.manage` | tenant | Add and remove payment method references. |
| `tenant.billing.invoices.read` | tenant | View invoice history and download PDFs. |
| `tenant.billing.overrides.apply` | tenant | Apply approved billing overrides (finance/ops). |
| `tenant.entitlements.read` | tenant | View current plan limits and usage. |

---

### `role_permissions`

M-to-M linking table connecting roles to the permissions they grant.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `role_id` | `bigint` | NOT NULL, FK → `roles.id` CASCADE, index | The role being granted this permission. |
| `permission_id` | `bigint` | NOT NULL, FK → `permissions.id` CASCADE, index | The permission granted by this role. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |

**Indexes:** composite unique on `(role_id, permission_id)`

**Note:** Deleting a `roles` row cascades to all `role_permissions` rows. Deleting a `permissions` row also cascades — exercise caution in production.

---

### `tenant_member_roles`

Role assignment at the tenant (collaboration-boundary) scope.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `tenant_member_id` | `bigint` | NOT NULL, FK → `tenant_members.id` CASCADE, index | The membership record this role is assigned to. |
| `role_id` | `bigint` | NOT NULL, FK → `roles.id` RESTRICT, index | The tenant-scoped role being granted. Must have `scope_type = 'tenant'`. |
| `assigned_by` | `bigint` | NOT NULL, FK → `users.id` RESTRICT | The user (admin) who made the assignment. Preserved for audit. |
| `assigned_at` | `timestamptz` | NOT NULL, default `NOW()` | Timestamp of the assignment. |
| `revoked_at` | `timestamptz` | nullable | Set when the role is explicitly removed. Null means currently active. |
| `revoked_by` | `bigint` | nullable, FK → `users.id` SET NULL | The user who revoked the assignment. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |

**Indexes:** composite index on `(tenant_member_id, role_id)`, index on `role_id`

**Active assignment check:** `revoked_at IS NULL`

**Note:** A tenant member may hold multiple tenant-scoped roles simultaneously (e.g. `tenant_admin` + `billing_admin`).

---

### `channel_member_roles`

Role assignment at the channel scope.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `channel_member_id` | `bigint` | NOT NULL, FK → `channel_members.id` CASCADE, index | The channel membership record this role is assigned to. |
| `role_id` | `bigint` | NOT NULL, FK → `roles.id` RESTRICT, index | The channel-scoped role being granted. Must have `scope_type = 'channel'`. |
| `assigned_by` | `bigint` | NOT NULL, FK → `users.id` RESTRICT | The user (channel admin or tenant admin) who made the assignment. |
| `assigned_at` | `timestamptz` | NOT NULL, default `NOW()` | Timestamp of the assignment. |
| `revoked_at` | `timestamptz` | nullable | Set when the role is explicitly removed. Null means currently active. |
| `revoked_by` | `bigint` | nullable, FK → `users.id` SET NULL | The user who revoked the assignment. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |

**Indexes:** composite index on `(channel_member_id, role_id)`, index on `role_id`

**Active assignment check:** `revoked_at IS NULL`

---

## Scope Rules

| `scope_type` | Applies to | Assignment table |
|---|---|---|
| `platform` | All tenants globally | Seeded; no assignment table in Phase 1 |
| `tenant` | One tenant | `tenant_member_roles` |
| `channel` | One channel | `channel_member_roles` |
| `self` | Own resources only | Permission code only; no dedicated assignment table |

- `roles.tenant_id` is nullable only for platform-level system roles.
- Self-scope permissions are still modeled as permission codes even though they do not require a dedicated assignment table.
- Billing permissions use tenant scope because billing records belong to a tenant account.

## Constraint Summary

| Rule | Details |
|---|---|
| `permissions.code` is unique | Prevents permission collision across scopes. |
| `role_permissions` has no duplicate grants | Composite unique on `(role_id, permission_id)`. |
| Role assignments are auditable | `assigned_by` and `assigned_at` are always required; `revoked_by` and `revoked_at` are set on removal. |
| Tenant member may hold multiple roles | No unique constraint on `tenant_member_id` in `tenant_member_roles`. |
| Channel member may hold multiple roles | No unique constraint on `channel_member_id` in `channel_member_roles`. |
| `member` role excludes billing by default | Seeded `role_permissions` for `member` must not include invoice, payment method, or billing override codes. |
