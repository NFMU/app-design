# REST API Documentation Overview

## Scope

These documents define the Phase 1 REST API surface for the collaboration platform.

Source priority:

1. Implemented identity service code in `d:/Workspace/Github/chat-slack/identity`
2. Analysis requirements and specs in `docs/analysis`
3. Existing ERD and RMD technical diagrams

The currently implemented REST endpoints are:

| Status | Method | Path | Source |
|---|---:|---|---|
| Implemented | POST | `/auth/login` | `identity/src/modules/auth/auth.controller.ts` |
| Implemented with response gap | POST | `/auth/signup` | `identity/src/modules/auth/auth.controller.ts` |
| Planned | POST | `/auth/email-verification/verify` | Verifies the token from the signup verification email. |

All other endpoints in this folder are clean target contracts for APIs that are not implemented yet.

## Planned Document Set

| Document | Domain |
|---|---|
| `01_identity_auth_api.md` | Identity and authentication |
| `02_profile_settings_api.md` | Profile and personal settings |
| `03_tenant_membership_api.md` | Tenant, membership, and invitation |
| `04_channel_api.md` | Channel management |
| `05_messaging_api.md` | Messaging, reads, pins, and attachments |
| `06_rbac_api.md` | RBAC enforcement |
| `07_billing_api.md` | Payment and subscription billing |

## Base URL

The identity service currently has no global API prefix. Paths are documented relative to the service root.

Example:

```http
POST /auth/login
```

## Authentication

Public endpoints are explicitly marked as public. Every other endpoint requires:

```http
Authorization: Bearer <access_token>
```

The login and signup endpoints return a raw JWT token value. Clients must add the `Bearer` scheme when sending it back.

## Success Envelope

Controllers should return the shared success envelope:

```json
{
  "success": true,
  "code": "SUCCESS",
  "message": "Request successful",
  "data": {}
}
```

Use `201 CREATED` for resource creation once the controller supports custom success statuses. The implemented auth controller currently returns `200 OK` through `buildSuccessResponseProxy`.

## Error Envelope

Business errors should return:

```json
{
  "success": false,
  "code": "AUTH-INVALID_CREDENTIALS",
  "message": "The provided credentials are invalid.",
  "errors": {
    "email": {
      "code": "invalid_format",
      "message": "Invalid email address"
    }
  }
}
```

`errors` is optional and is used for field-level validation detail.

Implementation note: the current validation pipe throws a Nest `BadRequestException`. Until the global exception filter preserves the response body from that exception, validation errors may serialize differently in the running service. The contract above is the target API shape.

## Common Error Codes

| HTTP | Code | Meaning |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Request body, query, or route parameter failed schema validation. |
| 400 | `BAD_REQUEST` | Request is syntactically valid but cannot be processed. |
| 401 | `UNAUTHORIZED` | Missing, expired, or invalid access token. |
| 402 | `PAYMENT_REQUIRED` | Required subscription or payment state is missing, past due, or blocked by billing policy. |
| 403 | `FORBIDDEN` | Authenticated user lacks permission for the action. |
| 404 | `NOT_FOUND` | Requested resource does not exist or is not visible to the caller. |
| 409 | `CONFLICT` | Request conflicts with an existing resource or current lifecycle state. |
| 429 | `RATE_LIMITED` | Request was rejected by a rate limit or abuse-control rule. |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected server failure. |

## ID And Date Conventions

| Concept | Type |
|---|---|
| `users.id`, `user_profiles.id`, `user_settings.id`, `user_sessions.id`, `password_resets.id` | UUID string in the current identity source code |
| `languages.id`, `timezones.id`, `locations.id` | Integer |
| Planned collaboration IDs | Opaque ID in API contracts; current RMD uses integer unless later source code chooses UUID |
| Planned billing IDs | Opaque ID in API contracts; billing RMD uses integer unless later source code chooses UUID |
| Date and time values | ISO 8601 string |
| Soft delete fields | `deleted_at` or equivalent lifecycle timestamp in persistence; APIs expose lifecycle status unless the timestamp is needed |

## Payment Data Boundary

Billing APIs must not accept or return raw card, bank account, or sensitive payment credential values. Payment setup should use a provider-hosted or provider-tokenized flow, and API responses should expose only display-safe references such as brand, last four digits, expiry, and provider reference IDs where the caller is authorized.

Billing access state is evaluated separately from tenant lifecycle state. Tenant, channel, messaging, and membership endpoints may return `402 PAYMENT_REQUIRED` when an authenticated caller has permission but the tenant's entitlement summary is `restricted`, `suspended`, or `blocked` by billing policy. Payment recovery or an active billing override can restore access without recreating tenant, workspace, membership, channel, or message records.

## Pagination

List endpoints should support:

| Query | Type | Default | Notes |
|---|---|---:|---|
| `limit` | integer | 25 | Minimum 1, maximum 100. |
| `cursor` | string | null | Opaque cursor returned by the previous page. |
| `sort` | string | context-specific | Use stable sort fields only. |

Paginated response data:

```json
{
  "items": [],
  "next_cursor": "opaque-next-cursor",
  "has_more": true
}
```
