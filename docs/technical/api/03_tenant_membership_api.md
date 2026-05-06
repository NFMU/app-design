# Tenant, Membership, And Invitation API

These endpoints are planned. They use the Phase 1 simplification from the analysis specs: the conceptual workspace boundary is persisted as `tenants`, so no separate `workspaces` REST surface is introduced.

## Resource Shapes

Tenant:

```json
{
  "id": 1001,
  "plan_id": 1,
  "owner_user_id": "f15e9a5b-57f2-4a0b-85b4-bce74a4c9b01",
  "name": "Acme",
  "slug": "acme",
  "domain": "acme.example.com",
  "status": "active",
  "timezone": "UTC",
  "locale": "en-US",
  "branding": {},
  "settings": {},
  "entitlement_summary": {
    "plan_id": 1,
    "source": "subscription",
    "subscription_status": "active",
    "billing_access_state": "enabled",
    "active_billing_override_id": null,
    "current_period_end": "2026-06-01T00:00:00.000Z"
  }
}
```

Tenant member:

```json
{
  "id": 3001,
  "tenant_id": 1001,
  "user_id": "f15e9a5b-57f2-4a0b-85b4-bce74a4c9b01",
  "invited_by": "29c68a53-6632-41b6-aabd-0678b7115c5d",
  "status": "active",
  "joined_at": "2026-05-05T08:00:00.000Z",
  "left_at": null,
  "removed_at": null
}
```

Invitation:

```json
{
  "id": 4001,
  "tenant_id": 1001,
  "channel_id": null,
  "email": "invitee@example.com",
  "invite_type": "tenant",
  "role_scope": "tenant",
  "role_code": "member",
  "status": "pending",
  "expires_at": "2026-05-12T08:00:00.000Z"
}
```

## Endpoint Summary

| Method | Path | Purpose |
|---|---|---|
| POST | `/tenants` | Create tenant and default collaboration boundary. |
| GET | `/tenants` | List tenants visible to the caller. |
| GET | `/tenants/{tenantId}` | Get tenant detail. |
| GET | `/tenants/{tenantId}/entitlement-summary` | Get read-only plan, subscription, override, and billing access state. |
| PATCH | `/tenants/{tenantId}` | Update tenant metadata and settings. |
| PATCH | `/tenants/{tenantId}/status` | Activate, suspend, lock, or unlock tenant. |
| GET | `/tenants/{tenantId}/members` | List tenant members. |
| PATCH | `/tenant-members/{memberId}/status` | Suspend, reactivate, remove, or mark member as left. |
| POST | `/tenants/{tenantId}/invitations` | Invite a member to a tenant or channel. |
| GET | `/tenants/{tenantId}/invitations` | List invitations for a tenant. |
| POST | `/invitations/{token}/accept` | Accept an invitation. |
| POST | `/invitations/{invitationId}/revoke` | Revoke a pending invitation. |

## POST `/tenants`

Requires platform administration permission.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `plan_id` | integer | yes | Must reference an active plan. |
| `owner_user_id` | string | yes | Existing user ID. |
| `name` | string | yes | 1 to 120 characters. |
| `slug` | string | yes | Unique URL-safe slug. |
| `domain` | string | no | Optional tenant domain. |
| `timezone` | string | no | Defaults to platform default. |
| `locale` | string | no | Defaults to platform default. |
| `branding` | object | no | Tenant branding payload. |
| `settings` | object | no | Tenant rules and policy payload. |

Success `data`: tenant resource.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 400 | `TENANT-INVALID_PLAN` | The selected plan does not exist or is inactive. |
| 400 | `TENANT-INVALID_OWNER` | Owner user does not exist. |
| 403 | `FORBIDDEN` | Caller cannot create tenants. |
| 409 | `TENANT-SLUG_ALREADY_EXISTS` | Tenant slug already exists. |

## PATCH `/tenants/{tenantId}`

Request body accepts any subset of `name`, `slug`, `domain`, `timezone`, `locale`, `branding`, and `settings`.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 403 | `FORBIDDEN` | Caller cannot update this tenant. |
| 404 | `TENANT-NOT_FOUND` | Tenant was not found. |
| 409 | `TENANT-SLUG_ALREADY_EXISTS` | Tenant slug already exists. |

## GET `/tenants/{tenantId}/entitlement-summary`

Returns a read-only summary derived from tenant plan and billing records. It must not include payment method details, invoice delivery data, provider payloads, or raw payment credentials.

Success `data`:

```json
{
  "tenant_id": 1001,
  "plan_id": 1,
  "source": "subscription",
  "subscription_id": 7101,
  "subscription_status": "active",
  "billing_policy_id": 8001,
  "billing_access_state": "enabled",
  "active_billing_override_id": null,
  "current_period_start": "2026-05-01T00:00:00.000Z",
  "current_period_end": "2026-06-01T00:00:00.000Z"
}
```

Allowed `source` values are `tenant_plan`, `subscription`, `trial`, `manual_override`, and `free`.

Allowed `billing_access_state` values are `enabled`, `grace_period`, `restricted`, `suspended`, and `blocked`.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot view tenant entitlements. |
| 404 | `TENANT-NOT_FOUND` | Tenant was not found. |
| 402 | `PAYMENT_REQUIRED` | Tenant access is blocked by billing policy. |

## PATCH `/tenants/{tenantId}/status`

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `status` | string | yes | One of `active`, `suspended`, `locked`. |
| `reason` | string | no | Audit reason. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `TENANT-INVALID_STATUS_TRANSITION` | Requested status transition is not allowed. |
| 403 | `FORBIDDEN` | Caller cannot change tenant status. |
| 404 | `TENANT-NOT_FOUND` | Tenant was not found. |

## GET `/tenants/{tenantId}/members`

Query:

| Field | Type | Notes |
|---|---|---|
| `status` | string | Optional member status filter. |
| `limit` | integer | Pagination limit. |
| `cursor` | string | Pagination cursor. |

Success `data` is paginated `tenant_members` with user summary.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot view tenant members. |
| 404 | `TENANT-NOT_FOUND` | Tenant was not found. |

## PATCH `/tenant-members/{memberId}/status`

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `status` | string | yes | One of `active`, `left`, `removed`, `suspended`. |
| `reason` | string | no | Audit reason. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `MEMBER-INVALID_STATUS_TRANSITION` | Requested member transition is not allowed. |
| 403 | `FORBIDDEN` | Caller cannot manage this member. |
| 404 | `MEMBER-NOT_FOUND` | Member was not found. |

## POST `/tenants/{tenantId}/invitations`

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `email` | string | yes | Valid email. |
| `channel_id` | integer | no | Optional channel-scoped invitation. |
| `invite_type` | string | yes | One of `tenant`, `channel`. |
| `role_scope` | string | no | `tenant` or `channel`. |
| `role_code` | string | no | Role to assign after acceptance. |
| `expires_at` | string | no | ISO timestamp. Defaults by policy. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 400 | `INVITATION-INVALID_CHANNEL` | Channel does not belong to this tenant. |
| 400 | `INVITATION-INVALID_ROLE` | Role cannot be assigned for this invitation. |
| 403 | `FORBIDDEN` | Caller cannot invite members. |
| 404 | `TENANT-NOT_FOUND` | Tenant was not found. |
| 409 | `INVITATION-ALREADY_PENDING` | A pending invitation already exists for this target. |

## POST `/invitations/{token}/accept`

Accepts an invitation. If the invitee is not authenticated, this can create or continue onboarding.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `user_id` | string | no | Existing authenticated user. |
| `signup` | object | no | Signup payload when accepting without an account. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `INVITATION-INVALID` | Invitation token is invalid. |
| 400 | `INVITATION-EXPIRED` | Invitation has expired. |
| 409 | `INVITATION-ALREADY_ACCEPTED` | Invitation was already accepted. |
| 409 | `INVITATION-REVOKED` | Invitation was revoked. |
| 409 | `MEMBER-ALREADY_EXISTS` | User is already a member of this tenant. |

## POST `/invitations/{invitationId}/revoke`

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot revoke this invitation. |
| 404 | `INVITATION-NOT_FOUND` | Invitation was not found. |
| 409 | `INVITATION-NOT_PENDING` | Only pending invitations can be revoked. |
