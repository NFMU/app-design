# Identity And Profile Spec

## Canonical Tables

- `users`
- `user_profiles`
- `user_settings`
- `user_sessions`
- `password_resets`

## Structural Rules

- `users` is the aggregate root for account identity.
- `user_profiles` is a required one-to-one extension of `users`.
- `user_settings` is a required one-to-one extension of `users`.
- `user_sessions` is a one-to-many child of `users`.
- `password_resets` is a one-to-many child of `users`.

## Suggested Core Attributes

`users`
- `id`
- `uuid`
- `email`
- `password_hash`
- `status`
- `auth_provider`
- `email_verified_at`
- `last_login_at`

`user_profiles`
- `user_id`
- `display_name`
- `avatar_url`
- `job_title`
- `phone`
- `language_code`
- `timezone`
- `status`
- `bio`

`user_settings`
- `user_id`
- `notifications_json`
- `preferences_json`

`user_sessions`
- `id`
- `user_id`
- `session_token`
- `refresh_token_hash`
- `ip_address`
- `user_agent`
- `device_name`
- `last_seen_at`
- `expires_at`
- `revoked_at`

`password_resets`
- `id`
- `user_id`
- `token_hash`
- `expires_at`
- `used_at`

## Constraint Hints

- `users.email` should be unique.
- `users.uuid` should be unique.
- `user_profiles.user_id` and `user_settings.user_id` are both primary-key foreign keys in the simplest design.
- password reset tokens expire and may be marked used.
- session revocation is represented by persisted state rather than only in-memory cache.
