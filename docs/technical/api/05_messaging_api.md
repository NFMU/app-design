# Messaging API

These endpoints are planned. Real-time delivery is outside this REST document, but REST endpoints must still support history, mutation, read tracking, pinning, and attachment workflows.

## Resource Shapes

Message:

```json
{
  "id": 7001,
  "channel_id": 5001,
  "sender_user_id": "f15e9a5b-57f2-4a0b-85b4-bce74a4c9b01",
  "parent_message_id": null,
  "message_type": "text",
  "content_text": "Hello team",
  "metadata": {},
  "edited_at": null,
  "edited_by": null,
  "deleted_at": null,
  "deleted_by": null,
  "created_at": "2026-05-05T08:00:00.000Z"
}
```

Attachment:

```json
{
  "id": 8001,
  "message_id": 7001,
  "file_name": "spec.pdf",
  "mime_type": "application/pdf",
  "file_size": 123456,
  "storage_path": "tenants/1001/channels/5001/spec.pdf",
  "url": "https://cdn.example.com/spec.pdf",
  "thumbnail_url": null
}
```

## Endpoint Summary

| Method | Path | Purpose |
|---|---|---|
| GET | `/channels/{channelId}/messages` | Load message history. |
| POST | `/channels/{channelId}/messages` | Send a message. |
| PATCH | `/messages/{messageId}` | Edit a message. |
| DELETE | `/messages/{messageId}` | Soft delete a message. |
| POST | `/channels/{channelId}/read` | Mark channel as read. |
| GET | `/channels/{channelId}/unread-count` | Get unread count for caller. |
| POST | `/messages/{messageId}/pins` | Pin a message. |
| DELETE | `/messages/{messageId}/pins` | Unpin a message. |
| POST | `/messages/{messageId}/attachments` | Attach file metadata to a message. |
| GET | `/messages/{messageId}/attachments` | List message attachments. |

## GET `/channels/{channelId}/messages`

Query:

| Field | Type | Notes |
|---|---|---|
| `limit` | integer | Pagination limit. |
| `cursor` | string | Pagination cursor. |
| `before` | string | Optional ISO timestamp or message ID. |
| `after` | string | Optional ISO timestamp or message ID. |
| `parent_message_id` | integer | Optional thread filter. |

Success `data`:

```json
{
  "items": [
    {
      "id": 7001,
      "channel_id": 5001,
      "sender_user_id": "f15e9a5b-57f2-4a0b-85b4-bce74a4c9b01",
      "content_text": "Hello team",
      "created_at": "2026-05-05T08:00:00.000Z"
    }
  ],
  "next_cursor": null,
  "has_more": false
}
```

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot read this channel. |
| 404 | `CHANNEL-NOT_FOUND` | Channel was not found. |

## POST `/channels/{channelId}/messages`

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `content_text` | string | yes | Required for text messages. |
| `message_type` | string | no | Defaults to `text`. |
| `parent_message_id` | integer | no | Must reference a message in the same channel. |
| `metadata` | object | no | Optional structured metadata. |
| `attachments` | object array | no | Optional attachment metadata created with the message. |

Attachment object:

| Field | Type | Required |
|---|---|---:|
| `file_name` | string | yes |
| `mime_type` | string | yes |
| `file_size` | integer | yes |
| `storage_path` | string | yes |
| `url` | string | yes |
| `thumbnail_url` | string | no |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 400 | `MESSAGE-PARENT_NOT_IN_CHANNEL` | Parent message does not belong to the channel. |
| 400 | `ATTACHMENT-INVALID_METADATA` | Attachment metadata is invalid. |
| 403 | `FORBIDDEN` | Caller cannot send messages to this channel. |
| 404 | `CHANNEL-NOT_FOUND` | Channel was not found. |

## PATCH `/messages/{messageId}`

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `content_text` | string | yes | Updated message content. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 403 | `FORBIDDEN` | Caller cannot edit this message. |
| 404 | `MESSAGE-NOT_FOUND` | Message was not found. |
| 409 | `MESSAGE-DELETED` | Deleted messages cannot be edited. |

## DELETE `/messages/{messageId}`

Soft deletes a message.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot delete this message. |
| 404 | `MESSAGE-NOT_FOUND` | Message was not found. |
| 409 | `MESSAGE-ALREADY_DELETED` | Message is already deleted. |

## POST `/channels/{channelId}/read`

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `last_read_message_id` | integer | yes | Latest message read by the caller in this channel. |
| `last_read_at` | string | no | ISO timestamp. Defaults to server time. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `MESSAGE-READ_CHECKPOINT_INVALID` | Message does not belong to this channel. |
| 403 | `FORBIDDEN` | Caller cannot read this channel. |
| 404 | `CHANNEL-NOT_FOUND` | Channel was not found. |

## GET `/channels/{channelId}/unread-count`

Success `data`:

```json
{
  "channel_id": 5001,
  "unread_count": 12,
  "last_read_message_id": 6990
}
```

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot read this channel. |
| 404 | `CHANNEL-NOT_FOUND` | Channel was not found. |

## POST `/messages/{messageId}/pins`

Pins a message for the channel.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot pin messages in this channel. |
| 404 | `MESSAGE-NOT_FOUND` | Message was not found. |
| 409 | `MESSAGE-PIN_ALREADY_EXISTS` | Message is already pinned. |

## DELETE `/messages/{messageId}/pins`

Unpins a message.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot unpin messages in this channel. |
| 404 | `MESSAGE-PIN_NOT_FOUND` | Pin was not found. |

## POST `/messages/{messageId}/attachments`

Adds persisted attachment metadata to an existing message.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `file_name` | string | yes | Display file name. |
| `mime_type` | string | yes | MIME type. |
| `file_size` | integer | yes | Positive byte count. |
| `storage_path` | string | yes | Internal storage path. |
| `url` | string | yes | Download URL. |
| `thumbnail_url` | string | no | Preview URL. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `ATTACHMENT-INVALID_METADATA` | Attachment metadata is invalid. |
| 400 | `ATTACHMENT-TOO_LARGE` | Attachment exceeds channel or tenant limits. |
| 403 | `FORBIDDEN` | Caller cannot attach files to this message. |
| 404 | `MESSAGE-NOT_FOUND` | Message was not found. |
| 409 | `MESSAGE-DELETED` | Cannot attach files to a deleted message. |

