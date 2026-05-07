# Identity And Profile Spec

## Canonical Tables

- `users`
- `user_profiles`
- `user_settings`
- `user_sessions`
- `password_resets`
- `email_verifications`
- `languages` _(reference)_
- `locations` _(reference)_
- `timezones` _(reference)_

## Structural Rules

- `users` is the aggregate root for account identity.
- `user_profiles` is a required one-to-one extension of `users`.
- `user_settings` is a required one-to-one extension of `users`.
- `user_sessions` is a one-to-many child of `users`.
- `password_resets` is a one-to-many child of `users`.
- `email_verifications` is a one-to-many child of `users`.
- `languages`, `locations`, and `timezones` are system-seeded reference tables; their primary keys are integer auto-increments.
- `user_settings.language_id` references `languages.id` with RESTRICT on delete.
- `user_settings.timezone_id` references `timezones.id` with RESTRICT on delete.
- `user_profiles.location_id` references `locations.id` with SET NULL on delete.

## Source Alignment

The column definitions below are aligned with the implemented TypeORM entities in
`d:/Workspace/Github/chat-slack/services/identity/src/database/entities/`.

---

## Table Definitions

### `users`

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `uuid` | PK, auto-generated | Surrogate primary key used in all relations. |
| `email` | `varchar(255)` | NOT NULL, unique (`uq_users_email`) | Login credential. Normalized to lowercase before storage. |
| `password_hash` | `varchar(255)` | NOT NULL | bcrypt or Argon2 hash of the user's password. Never stored in plaintext. |
| `is_email_verified` | `boolean` | NOT NULL, default `false` | Set to `true` after an `email_verifications` token is consumed. |
| `status` | `enum('active','inactive','blocked')` | NOT NULL, default `'active'` | Lifecycle state. `blocked` prevents all login and token refresh. |
| `last_login_at` | `timestamptz` | nullable | Updated on every successful authentication. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |
| `deleted_at` | `timestamptz` | nullable | Soft-delete marker. Non-null means the account is deleted. |

**Indexes:** `uq_users_email` (unique on `email`)

**Enum `UserStatus`:** `active` | `inactive` | `blocked`

---

### `user_profiles`

One-to-one with `users`. Stores public-facing identity and contact details.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `uuid` | PK, auto-generated | Surrogate key. |
| `user_id` | `uuid` | NOT NULL, FK → `users.id` CASCADE, unique (`uq_user_profiles_user_id`) | Owner reference. Enforces 1-to-1 with `users`. |
| `display_name` | `varchar(120)` | nullable | The name shown to other users in the collaboration UI. |
| `first_name` | `varchar(120)` | nullable | Legal or preferred first name. |
| `last_name` | `varchar(120)` | nullable | Legal or preferred last name. |
| `phone_number` | `varchar(32)` | nullable | Contact phone including country code. |
| `job_title` | `varchar(120)` | nullable | User-supplied role or title at their organization. |
| `company` | `varchar(120)` | nullable | User-supplied employer or company name. |
| `website` | `varchar(500)` | nullable | Personal or professional URL. |
| `location_id` | `integer` | nullable, FK → `locations.id` SET NULL, index (`idx_user_profiles_location_id`) | Reference to the `locations` table. Allows regional filtering without free-text strings. |
| `avatar_url` | `varchar(500)` | nullable | CDN URL for profile photo. Null means the UI uses an auto-generated avatar. |
| `date_of_birth` | `date` | nullable | Stored as a date-only value. Used for age verification where required. |
| `bio` | `text` | nullable | Free-form self-introduction. No length limit enforced at DB level. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** `uq_user_profiles_user_id` (unique on `user_id`), `idx_user_profiles_location_id`

---

### `user_settings`

One-to-one with `users`. Stores per-user application preferences.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `uuid` | PK, auto-generated | Surrogate key. |
| `user_id` | `uuid` | NOT NULL, FK → `users.id` CASCADE, unique (`uq_user_settings_user_id`) | Owner reference. Enforces 1-to-1 with `users`. |
| `language_id` | `integer` | NOT NULL, FK → `languages.id` RESTRICT, default `1` | UI display language. References the `languages` reference table. |
| `timezone_id` | `integer` | NOT NULL, FK → `timezones.id` RESTRICT, default `1` | User's local timezone for date/time display. References the `timezones` reference table. |
| `theme` | `enum('system','light','dark')` | NOT NULL, default `'system'` | UI color theme preference. `system` follows the OS preference. |
| `two_factor_enabled` | `boolean` | NOT NULL, default `false` | Whether TOTP or SMS 2FA is active for this account. |
| `marketing_emails_enabled` | `boolean` | NOT NULL, default `false` | Opt-in for marketing and promotional emails. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** `uq_user_settings_user_id` (unique on `user_id`)

**Enum `UserTheme`:** `system` | `light` | `dark`

---

### `user_sessions`

One-to-many child of `users`. Each row represents one authenticated device session.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `uuid` | PK, auto-generated | Surrogate key. |
| `user_id` | `uuid` | NOT NULL, FK → `users.id` CASCADE, index (`idx_user_sessions_user_id`) | Owner reference. |
| `refresh_token_hash` | `varchar(255)` | NOT NULL, unique (`uq_user_sessions_refresh_token_hash`) | Hashed refresh token. The raw token is issued to the client only at creation time. |
| `device_name` | `varchar(120)` | nullable | Human-readable device label, e.g. `"iPhone 15"` or `"Chrome on macOS"`. |
| `ip_address` | `varchar(45)` | nullable | Client IP at session creation. Supports IPv4 and IPv6 (max 45 chars). |
| `user_agent` | `text` | nullable | Full `User-Agent` header string from the originating request. |
| `last_used_at` | `timestamptz` | nullable | Updated on every token refresh. Used for idle-session detection. |
| `expires_at` | `timestamptz` | NOT NULL | Absolute expiry of the refresh token. Sessions past this time are invalid even if not revoked. |
| `revoked_at` | `timestamptz` | nullable | Explicit revocation timestamp. Set on logout or admin-forced sign-out. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** `idx_user_sessions_user_id`, `uq_user_sessions_refresh_token_hash` (unique)

**Active session check:** `revoked_at IS NULL AND expires_at > NOW()`

---

### `password_resets`

One-to-many child of `users`. Each row represents one password-reset request.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `uuid` | PK, auto-generated | Surrogate key. |
| `user_id` | `uuid` | NOT NULL, FK → `users.id` CASCADE, index (`idx_password_resets_user_id`) | Owner reference. |
| `token_hash` | `varchar(255)` | NOT NULL, unique (`uq_password_resets_token_hash`) | Hash of the one-time reset token sent to the user's email. |
| `requested_ip` | `varchar(45)` | nullable | Client IP that submitted the reset request. Used for fraud audit. |
| `requested_user_agent` | `text` | nullable | `User-Agent` header from the reset request. |
| `expires_at` | `timestamptz` | NOT NULL | Token expiry. Typically 1 hour from creation. |
| `used_at` | `timestamptz` | nullable | Set when the token is consumed. A non-null value means the token cannot be reused. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |

**Indexes:** `idx_password_resets_user_id`, `uq_password_resets_token_hash` (unique)

**Valid token check:** `used_at IS NULL AND expires_at > NOW()`

---

### `email_verifications`

One-to-many child of `users`. Each row represents one email verification challenge.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `uuid` | PK, auto-generated | Surrogate key. |
| `user_id` | `uuid` | NOT NULL, FK → `users.id` CASCADE, index (`idx_email_verifications_user_id`) | Owner reference. |
| `token_hash` | `varchar(255)` | NOT NULL, unique (`uq_email_verifications_token_hash`) | Hash of the one-time verification token sent to the user's email. |
| `expires_at` | `timestamptz` | NOT NULL | Token expiry. |
| `used_at` | `timestamptz` | nullable | Set when verification succeeds. Also triggers `users.is_email_verified = true`. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |

**Indexes:** `idx_email_verifications_user_id`, `uq_email_verifications_token_hash` (unique)

---

### `languages` _(reference)_

System-seeded lookup table. Not user-editable.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `integer` | PK, auto-increment | Integer surrogate key used by `user_settings.language_id`. |
| `code` | `varchar(10)` | NOT NULL, unique | BCP 47 language tag, e.g. `en`, `vi`, `ja`. |
| `locale` | `varchar(100)` | NOT NULL, unique | Full locale string, e.g. `en-US`, `vi-VN`. |
| `name` | `varchar(120)` | NOT NULL, unique | Human-readable name in English, e.g. `"English"`, `"Vietnamese"`. |

---

### `locations` _(reference)_

System-seeded country/region lookup table.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `integer` | PK, auto-increment | Integer surrogate key used by `user_profiles.location_id`. |
| `code` | `varchar(2)` | NOT NULL, unique | ISO 3166-1 alpha-2 country code, e.g. `VN`, `US`, `JP`. |
| `name` | `varchar(120)` | NOT NULL, unique | English country name, e.g. `"Vietnam"`, `"United States"`. |

---

### `timezones` _(reference)_

System-seeded timezone lookup table.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `integer` | PK, auto-increment | Integer surrogate key used by `user_settings.timezone_id`. |
| `name` | `varchar(100)` | NOT NULL, unique | IANA timezone identifier, e.g. `Asia/Ho_Chi_Minh`, `UTC`, `America/New_York`. |
| `utc_offset` | `varchar(10)` | NOT NULL | Offset string for display, e.g. `+07:00`, `-05:00`. Not used for calculation — always use the IANA name. |

---

## Constraint Summary

| Rule | Details |
|---|---|
| `users.email` is unique | Enforced by index `uq_users_email`. |
| `user_profiles.user_id` is unique | Enforced by index `uq_user_profiles_user_id`. |
| `user_settings.user_id` is unique | Enforced by index `uq_user_settings_user_id`. |
| Refresh tokens are unique | Enforced by index `uq_user_sessions_refresh_token_hash`. |
| Reset tokens are unique | Enforced by index `uq_password_resets_token_hash`. |
| Verification tokens are unique | Enforced by index `uq_email_verifications_token_hash`. |
| Reference table FKs use RESTRICT | Prevents deleting a `language`, `timezone`, or `location` that is still referenced. |
| User child records use CASCADE | Deleting a `users` row hard-deletes its sessions, resets, verifications, and settings. |
| Soft delete on `users` | `deleted_at IS NOT NULL` marks the account as deleted without removing the row. |

## Enum Definitions

```
UserStatus:  active | inactive | blocked
UserTheme:   system | light | dark
```
