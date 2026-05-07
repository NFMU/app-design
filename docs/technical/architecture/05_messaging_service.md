# Messaging Service

## Status

**Planned.** No source code exists yet under `services/`. Spec authority remains `docs/analysis/specs/04_messaging.md`.

## Responsibility

The messaging service is the authority for **content posted in channels**: messages, replies, read checkpoints, pins, and file attachment metadata. It is the highest-write service in the platform.

It does **not** know about channel ownership rules; it asks `channels` to validate that the sender is a permitted poster. It does **not** store binary file content; attachment files live in object storage and the service stores only metadata and URLs.

## Owned Tables

| Table | PK type | Purpose |
|---|---|---|
| `messages` | `integer` | The message itself. Public ID is `messages.uuid`. Belongs to one channel; optionally references a parent for replies. |
| `message_reads` | `integer` | Per-user, per-channel read checkpoint. Drives unread-count computation. |
| `message_pins` | `integer` | Pin and unpin events. Active pins have `unpinned_at IS NULL`. |
| `message_attachments` | `integer` | File metadata for attached files. Binary content lives in object storage. |

Full column definitions: `docs/analysis/specs/04_messaging.md`.

## Module Layout (Target)

```
services/messaging/src/
├── app.module.ts
├── main.ts
├── core/
├── infrastructure/
│   ├── clients/
│   │   ├── identity.client.ts
│   │   ├── channels.client.ts       # channel + membership validation
│   │   └── storage.client.ts        # presigned upload URLs, S3-compatible
│   ├── database/database.module.ts
│   ├── orms/
│   │   ├── message.orm.ts
│   │   ├── message-read.orm.ts
│   │   ├── message-pin.orm.ts
│   │   └── message-attachment.orm.ts
│   └── realtime/
│       └── websocket.gateway.ts     # send / receive events
└── modules/
    ├── messages/                    # Send, edit, soft-delete, list
    ├── reads/                       # Mark read, unread count
    ├── pins/                        # Pin, unpin, list
    └── attachments/                 # Upload init, finalize
```

## Planned Public API Surface

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/channels/{channelUuid}/messages` | Bearer + can post | Send a new message (or reply if `parent_message_uuid` provided). |
| GET | `/channels/{channelUuid}/messages` | Bearer + channel member | Paginated list, supports `before` / `after` cursor. |
| GET | `/messages/{messageUuid}` | Bearer + channel member | Single message detail. |
| PATCH | `/messages/{messageUuid}` | Bearer + sender or `channel.messages.edit_any` | Edit body. Updates `edited_at`, `edited_by`. |
| DELETE | `/messages/{messageUuid}` | Bearer + sender or `channel.messages.delete_any` | Soft delete. Sets `deleted_at`, `deleted_by`. |
| POST | `/channels/{channelUuid}/reads` | Bearer + channel member | Update read checkpoint to `last_read_message_id`. |
| GET | `/channels/{channelUuid}/unread-count` | Bearer + channel member | Compute unread count from checkpoint. |
| POST | `/messages/{messageUuid}/pin` | Bearer + `channel.messages.pin` | Pin a message. |
| DELETE | `/messages/{messageUuid}/pin` | Bearer + `channel.messages.pin` | Unpin (sets `unpinned_at`). |
| GET | `/channels/{channelUuid}/pins` | Bearer + channel member | List active pins. |
| POST | `/messages/{messageUuid}/attachments/upload-url` | Bearer + sender | Request a presigned upload URL. |
| POST | `/messages/{messageUuid}/attachments` | Bearer + sender | Finalize an upload by recording metadata. |

Full request/response shapes: `docs/technical/api/05_messaging_api.md`.

## Internal API Surface

| Method | Path | Purpose |
|---|---|---|
| POST | `/internal/messages/system` | Allow other services to post system messages (e.g. `member_joined` events from `channels`). |
| GET | `/internal/channels/{id}/last-message` | For `channels` sidebar ordering. |

## Real-Time Surface

WebSocket / SSE endpoint at `/ws/channels/{channelUuid}`. Required for live message delivery, typing indicators, and read receipts. Auth uses the same Bearer JWT (passed as a query parameter or first message frame depending on transport).

Events emitted:

| Event | Payload |
|---|---|
| `message.created` | Full message object. |
| `message.edited` | Message with new body and `edited_at`. |
| `message.deleted` | `{ message_uuid, deleted_at }`. |
| `message.pinned` | Pin record. |
| `message.unpinned` | `{ message_uuid }`. |
| `read.updated` | `{ user_id, last_read_message_id }`. |

## Inter-Service Dependencies

**Outbound:**

| Target | Endpoint(s) | When |
|---|---|---|
| `identity` | `GET /internal/users/{id}` | Hydrate sender info in API responses (with cache). |
| `channels` | `GET /internal/channels/{id}` | Verify channel exists and is not archived. |
| `channels` | `GET /internal/channels/{id}/members?user_id=X` | Permission check on message send. |
| `rbac` (planned) | `GET /internal/role-permissions?user_id=X&channel_id=Y` | Resolve fine-grained permissions (`channel.messages.edit_any` etc.). |

**Inbound:**

| Caller | Endpoint(s) | When |
|---|---|---|
| `channels` | `POST /internal/messages/system` | Emit system messages on member add/remove, channel rename. |
| `channels` | `GET /internal/channels/{id}/last-message` | Sidebar ordering. |

## Cross-Service Reference Columns

| Column | Owning service / table | Validation strategy |
|---|---|---|
| `messages.channel_id` | `channels.channels` (integer) | Validate-on-write via `/internal/channels/{id}`. |
| `messages.sender_user_id` | `identity.users` (uuid) | Trusted (from authenticated caller). |
| `messages.edited_by` | `identity.users` (uuid) | Trusted. |
| `messages.deleted_by` | `identity.users` (uuid) | Trusted. |
| `message_reads.channel_id` | `channels.channels` (integer) | Trusted (path parameter). |
| `message_reads.user_id` | `identity.users` (uuid) | Trusted. |
| `message_pins.channel_id` | `channels.channels` (integer) | Trusted (derived from `message.channel_id`). |
| `message_pins.pinned_by`, `unpinned_by` | `identity.users` (uuid) | Trusted. |

## Performance & Scale Notes

- **`messages` is the largest table.** Partition by `channel_id` once per-channel volume exceeds ~10M rows. Until then, a composite index on `(channel_id, id)` serves chronological pagination efficiently.
- **`message_reads` is upsert-heavy.** Use `INSERT ... ON CONFLICT (channel_id, user_id) DO UPDATE` rather than read-modify-write.
- **Unread count is computed, not stored.** A simple `COUNT(*) WHERE id > last_read_message_id` against the partial index is acceptable up to per-channel volumes of ~100k unread; beyond that, denormalize a counter.
- **Attachments use direct-to-storage uploads.** Presigned URLs avoid routing binary data through the messaging service. The service only sees metadata.

## Soft Delete & Edit History

`edited_at` and `deleted_at` are nullable timestamps. There is **no** `is_edited` or `is_deleted` boolean. Clients infer state from null vs non-null.

Edit history (storing previous bodies) is **out of scope for Phase 1**. If needed, add a `message_edits` audit table later.
