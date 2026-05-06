# Channel API

These endpoints are planned. Channel APIs operate inside a tenant because Phase 1 uses `tenants` as the persisted collaboration boundary.

## Resource Shapes

Channel:

```json
{
  "id": 5001,
  "tenant_id": 1001,
  "created_by": "f15e9a5b-57f2-4a0b-85b4-bce74a4c9b01",
  "name": "general",
  "slug": "general",
  "channel_type": "public",
  "description": "Company-wide discussion",
  "topic": "Announcements and general chat",
  "avatar_url": null,
  "status": "active",
  "settings": {}
}
```

Channel member:

```json
{
  "id": 6001,
  "channel_id": 5001,
  "user_id": "f15e9a5b-57f2-4a0b-85b4-bce74a4c9b01",
  "added_by": "29c68a53-6632-41b6-aabd-0678b7115c5d",
  "status": "active",
  "notification_level": "default",
  "is_pinned": false,
  "display_order": 0,
  "joined_at": "2026-05-05T08:00:00.000Z",
  "left_at": null
}
```

## Endpoint Summary

| Method | Path | Purpose |
|---|---|---|
| POST | `/tenants/{tenantId}/channels` | Create public or private channel. |
| GET | `/tenants/{tenantId}/channels` | List visible channels for caller. |
| GET | `/channels/{channelId}` | Get channel detail. |
| PATCH | `/channels/{channelId}` | Update channel metadata or settings. |
| DELETE | `/channels/{channelId}` | Soft delete or disable a channel. |
| GET | `/channels/{channelId}/members` | List channel members. |
| POST | `/channels/{channelId}/members` | Add a member to a channel. |
| DELETE | `/channels/{channelId}/members/{userId}` | Remove a member from a channel. |
| POST | `/channels/{channelId}/leave` | Leave a channel as the authenticated member. |
| PATCH | `/channels/{channelId}/members/me/preferences` | Update personal channel preferences. |

## POST `/tenants/{tenantId}/channels`

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `name` | string | yes | 1 to 120 characters. |
| `slug` | string | yes | Unique within tenant. |
| `channel_type` | string | yes | One of `public`, `private`. |
| `description` | string | no | Free text. |
| `topic` | string | no | Free text. |
| `avatar_url` | string | no | Valid URL. |
| `settings` | object | no | Channel rules and preferences payload. |

Behavior:

- The channel belongs to the tenant.
- The creator is immediately added as a channel member.
- Elevated channel privileges are assigned through RBAC role assignment APIs.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 403 | `FORBIDDEN` | Caller cannot create channels in this tenant. |
| 404 | `TENANT-NOT_FOUND` | Tenant was not found. |
| 409 | `CHANNEL-SLUG_ALREADY_EXISTS` | Channel slug already exists in this tenant. |

## GET `/tenants/{tenantId}/channels`

Query:

| Field | Type | Notes |
|---|---|---|
| `channel_type` | string | Optional `public` or `private` filter. |
| `status` | string | Optional status filter. |
| `joined` | boolean | When true, return only channels the caller has joined. |
| `limit` | integer | Pagination limit. |
| `cursor` | string | Pagination cursor. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot list channels in this tenant. |
| 404 | `TENANT-NOT_FOUND` | Tenant was not found. |

## PATCH `/channels/{channelId}`

Request body accepts any subset of `name`, `slug`, `description`, `topic`, `avatar_url`, `status`, and `settings`.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 403 | `FORBIDDEN` | Caller cannot update this channel. |
| 404 | `CHANNEL-NOT_FOUND` | Channel was not found. |
| 409 | `CHANNEL-SLUG_ALREADY_EXISTS` | Channel slug already exists in this tenant. |

## POST `/channels/{channelId}/members`

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `user_id` | string | yes | User to add. |
| `role_codes` | string array | no | Optional channel roles assigned after join. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `CHANNEL-USER_NOT_TENANT_MEMBER` | User must be a tenant member first. |
| 400 | `CHANNEL-INVALID_ROLE` | One or more channel roles are invalid. |
| 403 | `FORBIDDEN` | Caller cannot add channel members. |
| 404 | `CHANNEL-NOT_FOUND` | Channel was not found. |
| 404 | `USER-NOT_FOUND` | User was not found. |
| 409 | `CHANNEL-MEMBER_ALREADY_EXISTS` | User is already an active channel member. |

## DELETE `/channels/{channelId}/members/{userId}`

Removes a user's channel participation without removing tenant membership.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot remove channel members. |
| 404 | `CHANNEL-MEMBER_NOT_FOUND` | Channel member was not found. |
| 409 | `CHANNEL-CREATOR_REMOVE_BLOCKED` | Last owner or protected creator cannot be removed. |

## POST `/channels/{channelId}/leave`

Authenticated member leaves a channel.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot leave this channel. |
| 404 | `CHANNEL-MEMBER_NOT_FOUND` | Caller is not a channel member. |
| 409 | `CHANNEL-LAST_OWNER_CANNOT_LEAVE` | Last channel owner cannot leave until another owner is assigned. |

## PATCH `/channels/{channelId}/members/me/preferences`

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `notification_level` | string | no | One of `default`, `all`, `mentions`, `none`. |
| `is_pinned` | boolean | no | Personal pinned channel state. |
| `display_order` | integer | no | Personal display ordering. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 404 | `CHANNEL-MEMBER_NOT_FOUND` | Caller is not a channel member. |
