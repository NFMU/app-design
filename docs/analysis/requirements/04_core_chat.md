# Core Chat Requirements

## Business Outcome

Members must be able to participate in channel conversations with real-time delivery, moderation, read tracking, pinning, and attachments.

## Functional Requirements

- A member can open a channel conversation.
- A member can view message lists and load more history through pagination or infinite scroll.
- A member can send a text message.
- A member can receive new messages in real time.
- A member can view message sender, time, content, and edited status.
- A member can edit or soft delete their own message.
- Channel, workspace, or tenant administrators can edit or soft delete messages owned by others when policy allows it.
- A member can mark a channel as read.
- The system can show unread count.
- The system can emit in-session notifications for new activity.
- Authorized users can pin and unpin messages.
- Authorized users can attach files to messages.
- Messages can participate in lightweight threaded replies.

## Business Rules

- Message deletion in Phase 1 is soft delete, not hard delete.
- A reply references one optional parent message.
- Read tracking is persisted and must support unread-count calculations.
- Message pinning is a persisted moderation action.
- Attachments are persisted records linked to a message, not temporary upload state.

## Persisted Concepts Expected By Downstream Design

- message
- message read checkpoint
- message pin
- message attachment
