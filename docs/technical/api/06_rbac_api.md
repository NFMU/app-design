# RBAC API

These endpoints are planned. RBAC applies to platform, tenant, channel, and self scopes. Users do not receive raw permissions directly; roles bundle permissions, and role assignments are contextual.

## Resource Shapes

Role:

```json
{
  "id": 9001,
  "tenant_id": 1001,
  "scope_type": "tenant",
  "code": "billing_admin",
  "name": "Billing Admin",
  "description": "Can manage tenant subscription, invoices, and payment settings",
  "is_system": true
}
```

Permission:

```json
{
  "id": 9101,
  "scope_type": "tenant",
  "code": "tenant.billing.subscription.manage",
  "name": "Manage tenant subscription",
  "description": "Allows starting, changing, canceling, or recovering a tenant subscription"
}
```

## Seeded Billing Permissions

| Scope | Code | Intended roles |
|---|---|---|
| platform | `platform.billing.prices.manage` | `super_admin` |
| tenant | `tenant.billing.profile.manage` | `tenant_admin`, `billing_admin` |
| tenant | `tenant.billing.subscription.manage` | `tenant_admin`, `billing_admin` |
| tenant | `tenant.billing.payment_methods.manage` | `tenant_admin`, `billing_admin` |
| tenant | `tenant.billing.invoices.read` | `tenant_admin`, `billing_admin` |
| tenant | `tenant.billing.overrides.apply` | `super_admin` or explicitly delegated billing operator |
| tenant | `tenant.entitlements.read` | `tenant_admin`, `billing_admin`, selected support roles |
Members without these permissions must not receive invoice-delivery fields, payment method display references, provider references, billing override details, or billing provider event metadata.

## Endpoint Summary

| Method | Path | Purpose |
|---|---|---|
| GET | `/permissions` | List permission definitions. |
| GET | `/roles` | List roles visible to the caller. |
| POST | `/roles` | Create a custom role. |
| GET | `/roles/{roleId}` | Get role detail with permissions. |
| PATCH | `/roles/{roleId}` | Update custom role metadata. |
| DELETE | `/roles/{roleId}` | Disable or delete a custom role. |
| PUT | `/roles/{roleId}/permissions/{permissionId}` | Grant a permission to a role. |
| DELETE | `/roles/{roleId}/permissions/{permissionId}` | Revoke a permission from a role. |
| PUT | `/tenant-members/{memberId}/roles/{roleId}` | Assign tenant-scoped role. |
| DELETE | `/tenant-members/{memberId}/roles/{roleId}` | Revoke tenant-scoped role. |
| PUT | `/channel-members/{memberId}/roles/{roleId}` | Assign channel-scoped role. |
| DELETE | `/channel-members/{memberId}/roles/{roleId}` | Revoke channel-scoped role. |
| GET | `/me/permissions` | Get effective permissions for caller. |

## GET `/permissions`

Query:

| Field | Type | Notes |
|---|---|---|
| `scope_type` | string | Optional `platform`, `tenant`, `channel`, or `self`. |
| `code_prefix` | string | Optional prefix such as `tenant.billing.`. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 401 | `UNAUTHORIZED` | Missing, expired, or invalid access token. |

## POST `/roles`

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `tenant_id` | integer | no | Required for tenant-scoped custom roles. Null for platform system roles. |
| `scope_type` | string | yes | One of `platform`, `tenant`, `channel`, `self`. |
| `code` | string | yes | Unique within scope. |
| `name` | string | yes | Display name. |
| `description` | string | no | Free text. |
| `permission_ids` | integer array | no | Permissions granted at creation. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 400 | `RBAC-INVALID_SCOPE` | Scope type is not supported. |
| 400 | `RBAC-INVALID_PERMISSION` | One or more permissions cannot be granted to this role. |
| 403 | `FORBIDDEN` | Caller cannot create roles in this scope. |
| 409 | `RBAC-ROLE_CODE_ALREADY_EXISTS` | Role code already exists in this scope. |

## PATCH `/roles/{roleId}`

Request body accepts `name`, `description`, and, for custom roles only, `code`.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot update this role. |
| 404 | `RBAC-ROLE_NOT_FOUND` | Role was not found. |
| 409 | `RBAC-SYSTEM_ROLE_IMMUTABLE` | System role cannot be modified. |
| 409 | `RBAC-ROLE_CODE_ALREADY_EXISTS` | Role code already exists in this scope. |

## DELETE `/roles/{roleId}`

Disables or deletes a custom role. System roles are immutable.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot delete this role. |
| 404 | `RBAC-ROLE_NOT_FOUND` | Role was not found. |
| 409 | `RBAC-SYSTEM_ROLE_IMMUTABLE` | System role cannot be deleted. |
| 409 | `RBAC-ROLE_IN_USE` | Role is still assigned to members. |

## PUT `/roles/{roleId}/permissions/{permissionId}`

Grants a permission to a role.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `RBAC-PERMISSION_SCOPE_MISMATCH` | Permission scope does not match role scope. |
| 403 | `FORBIDDEN` | Caller cannot manage role permissions. |
| 404 | `RBAC-ROLE_NOT_FOUND` | Role was not found. |
| 404 | `RBAC-PERMISSION_NOT_FOUND` | Permission was not found. |
| 409 | `RBAC-SYSTEM_ROLE_IMMUTABLE` | System role cannot be modified. |
| 409 | `RBAC-PERMISSION_ALREADY_GRANTED` | Role already has this permission. |

## DELETE `/roles/{roleId}/permissions/{permissionId}`

Revokes a permission from a role.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot manage role permissions. |
| 404 | `RBAC-ROLE_PERMISSION_NOT_FOUND` | Role permission link was not found. |
| 409 | `RBAC-SYSTEM_ROLE_IMMUTABLE` | System role cannot be modified. |

## PUT `/tenant-members/{memberId}/roles/{roleId}`

Assigns a tenant-scoped role to a tenant member.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `reason` | string | no | Optional audit reason. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `RBAC-ROLE_SCOPE_MISMATCH` | Role is not assignable to tenant members. |
| 403 | `FORBIDDEN` | Caller cannot assign tenant roles. |
| 404 | `MEMBER-NOT_FOUND` | Tenant member was not found. |
| 404 | `RBAC-ROLE_NOT_FOUND` | Role was not found. |
| 409 | `RBAC-ROLE_ALREADY_ASSIGNED` | Member already has this role. |

## DELETE `/tenant-members/{memberId}/roles/{roleId}`

Revokes a tenant-scoped role.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot revoke tenant roles. |
| 404 | `RBAC-ROLE_ASSIGNMENT_NOT_FOUND` | Role assignment was not found. |
| 409 | `RBAC-LAST_ADMIN_ROLE_BLOCKED` | Cannot remove the last administrative role for the tenant. |

## PUT `/channel-members/{memberId}/roles/{roleId}`

Assigns a channel-scoped role to a channel member.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `RBAC-ROLE_SCOPE_MISMATCH` | Role is not assignable to channel members. |
| 403 | `FORBIDDEN` | Caller cannot assign channel roles. |
| 404 | `CHANNEL-MEMBER_NOT_FOUND` | Channel member was not found. |
| 404 | `RBAC-ROLE_NOT_FOUND` | Role was not found. |
| 409 | `RBAC-ROLE_ALREADY_ASSIGNED` | Member already has this role. |

## DELETE `/channel-members/{memberId}/roles/{roleId}`

Revokes a channel-scoped role.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot revoke channel roles. |
| 404 | `RBAC-ROLE_ASSIGNMENT_NOT_FOUND` | Role assignment was not found. |
| 409 | `RBAC-LAST_CHANNEL_OWNER_BLOCKED` | Cannot remove the last channel owner role. |

## GET `/me/permissions`

Returns effective permission codes for the caller in an optional context.

Query:

| Field | Type | Notes |
|---|---|---|
| `tenant_id` | integer | Optional tenant context. |
| `channel_id` | integer | Optional channel context. |

Success `data`:

```json
{
  "scope": {
    "tenant_id": 1001,
    "channel_id": 5001
  },
  "permissions": [
    "tenant.entitlements.read",
    "tenant.billing.invoices.read",
    "channel.messages.read",
    "channel.messages.send"
  ]
}
```

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 401 | `UNAUTHORIZED` | Missing, expired, or invalid access token. |
| 403 | `FORBIDDEN` | Caller cannot inspect this context. |
| 404 | `TENANT-NOT_FOUND` | Tenant was not found. |
| 404 | `CHANNEL-NOT_FOUND` | Channel was not found. |
