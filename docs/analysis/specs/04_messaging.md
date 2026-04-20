# Messaging Spec

## Canonical Tables

- `messages`
- `message_reads`
- `message_pins`
- `message_attachments`

## Structural Rules

- `messages` belongs to exactly one `channels` record.
- each `messages` row has exactly one sender stored by user reference.
- `messages.parent_message_id` is optional and supports simple reply threading.
- `message_reads` stores the latest read checkpoint per user and channel.
- `message_pins` stores persisted pin or unpin actions for moderation and quick access.
- `message_attachments` stores file metadata linked to one message.

## Suggested Core Attributes

`messages`
- `id`
- `uuid`
- `channel_id`
- `sender_user_id`
- `parent_message_id`
- `message_type`
- `content_text`
- `metadata_json`
- `edited_at`
- `edited_by`
- `deleted_at`
- `deleted_by`
- `created_at`

`message_reads`
- `id`
- `channel_id`
- `user_id`
- `last_read_message_id`
- `last_read_at`

`message_pins`
- `id`
- `channel_id`
- `message_id`
- `pinned_by`
- `pinned_at`
- `unpinned_at`

`message_attachments`
- `id`
- `message_id`
- `file_name`
- `mime_type`
- `file_size`
- `storage_path`
- `url`
- `thumbnail_url`

## Constraint Hints

- soft delete is represented by `deleted_at` and optional `deleted_by`
- a reply message may reference one parent but root messages have no parent
- `message_reads` should support one current checkpoint per user-channel pair
- attachments may be absent for most messages, so they remain a child table rather than inline fields

## Query-Oriented Guidance

- unread count is derived from `message_reads` and channel activity, not stored redundantly on every message
- edited or deleted boolean flags are unnecessary when timestamps already express that lifecycle state
