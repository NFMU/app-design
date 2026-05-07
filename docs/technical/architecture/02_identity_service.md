# Identity Service

## Status

**Implemented.** Operational endpoints exist for authentication and signup. Profile, settings, and reference HTTP surfaces are partially implemented (entities and repository helpers exist; controllers in progress).

Source: `d:/Workspace/Github/chat-slack/services/identity/`.

## Responsibility

The identity service is the authority for **who** uses the platform and **what their personal preferences are**. It also owns **shared reference data** (languages, timezones, locations) that other services consume by ID.

It does **not** know about tenants, channels, messages, RBAC, or billing. Other services hold their own user-membership and role records, referencing `users.id` (UUID) only.

## Owned Tables

| Table | PK type | Purpose |
|---|---|---|
| `users` | `uuid` | Account credentials and lifecycle state. |
| `user_profiles` | `uuid` | Public-facing identity (display name, avatar, contact). 1:1 with `users`. |
| `user_settings` | `uuid` | Personal preferences (language, timezone, theme, 2FA). 1:1 with `users`. |
| `user_sessions` | `uuid` | Refresh-token-backed device sessions. N:1 with `users`. |
| `password_resets` | `uuid` | One-time password reset tokens. N:1 with `users`. |
| `email_verifications` | `uuid` | One-time email verification tokens. N:1 with `users`. |
| `languages` | `integer` (PK) + `uuid` unique | **Reference table** — supported UI languages. `id` for same-service FK; `uuid` for cross-service references. |
| `timezones` | `integer` (PK) + `uuid` unique | **Reference table** — IANA timezone catalog. `id` for same-service FK; `uuid` for cross-service references. |
| `locations` | `integer` (PK) + `uuid` unique | **Reference table** — ISO 3166-1 country list. `id` for same-service FK; `uuid` for cross-service references. |

Full column definitions: `docs/analysis/specs/01_identity_and_profile.md`.

## Module Layout

```
services/identity/src/
├── app.module.ts
├── main.ts
├── core/
│   ├── decorators/        # bearer-auth, future service-auth
│   ├── errors/            # AuthError, ReferenceError
│   ├── filters/           # HTTP/business exception filters
│   ├── interceptors/      # response status interceptor
│   └── utils/              # password, token, request metadata, date
├── database/
│   ├── database.module.ts
│   ├── entities/          # TypeORM entities (canonical column source)
│   ├── factories/         # Seed data sources
│   ├── migrations/        # 1776...init.ts, plus per-feature migrations
│   └── seeders/           # Language / Timezone / Location seeders
└── modules/
    ├── auth/              # Login, signup, password reset, email verify, sessions
    ├── profiles/          # Profile read/update (skeleton)
    ├── settings/          # Settings read/update (skeleton)
    ├── references/        # PLANNED — public + internal reference endpoints
    └── mail/              # Outbound email (verification, reset)
```

## Public API Surface

User-facing endpoints. Bearer-token auth unless marked public.

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/auth/signup` | Public | Create account, send verification email. |
| POST | `/auth/login` | Public | Issue access + refresh tokens. |
| POST | `/auth/refresh` | Public (carries refresh token) | Rotate token pair. |
| POST | `/auth/logout` | Bearer | Revoke current session. |
| POST | `/auth/email-verification/verify` | Public | Consume email verification token. |
| POST | `/auth/password/reset/request` | Public | Send password reset email. |
| POST | `/auth/password/reset/confirm` | Public | Consume reset token, set new password. |
| POST | `/auth/password/change` | Bearer | Change password (knows current). |
| GET | `/me` | Bearer | Composite user + profile + settings view. |
| PATCH | `/me/profile` | Bearer | Update profile fields. |
| GET | `/me/settings` | Bearer | Get settings. |
| PATCH | `/me/settings` | Bearer | Update settings. |
| GET | `/me/sessions` | Bearer | List sessions. |
| DELETE | `/me/sessions/{sessionId}` | Bearer | Revoke a session. |
| GET | `/reference/languages` | Bearer | List languages for UI dropdowns. |
| GET | `/reference/timezones` | Bearer | List timezones for UI dropdowns. |
| GET | `/reference/locations` | Bearer | List locations for UI dropdowns. |

Full request/response shapes: `docs/technical/api/01_identity_auth_api.md` and `02_profile_settings_api.md`.

## Internal API Surface (Service-to-Service)

Authenticated by `X-Service-Token`. Not exposed in public OpenAPI. See `01_shared_reference_resources.md` for the full contract.

| Method | Path | Purpose |
|---|---|---|
| GET | `/internal/reference/languages` | Bulk list with ETag support. |
| GET | `/internal/reference/languages/{uuid}` | Single-item validate / fetch by UUID. |
| GET | `/internal/reference/timezones` | Bulk list with ETag support. |
| GET | `/internal/reference/timezones/{uuid}` | Single-item validate / fetch by UUID. |
| GET | `/internal/reference/locations` | Bulk list with ETag support. |
| GET | `/internal/reference/locations/{uuid}` | Single-item validate / fetch by UUID. |
| GET | `/internal/users/{id}` | Validate that a `users.id` exists (for cross-service writes). Returns minimal user summary. |

## Inter-Service Dependencies

**Outbound:** none. Identity does not call any other service.

**Inbound:** every other service calls identity:

| Caller | Endpoint(s) | When |
|---|---|---|
| `tenants` | `/internal/users/{id}` | Validate `owner_user_id` on tenant create, `user_id` on invitation accept. |
| `tenants` | `/internal/reference/{type}/{uuid}` | Validate `timezone_id` and `language_id` (uuid values) on tenant create/update. |
| All services | `/internal/reference/{type}` (bulk) | Cache reference list at startup / periodic refresh. |

## Authentication Flow

```
Client                          Identity                         Other Services
  │                                │                                  │
  │ POST /auth/signup              │                                  │
  ├───────────────────────────────►│ create user, send verify email   │
  │                                │                                  │
  │ POST /auth/login               │                                  │
  ├───────────────────────────────►│ verify password, issue JWT       │
  │ access_token + refresh_token   │                                  │
  │◄───────────────────────────────┤                                  │
  │                                │                                  │
  │ Authorization: Bearer <jwt>    │                                  │
  ├──────────────────────────────────────────────────────────────────►│
  │                                │                                  │ verify JWT
  │                                │                                  │ using shared secret
  │                                │                                  │
  │ <response>                     │                                  │
  │◄──────────────────────────────────────────────────────────────────┤
```

JWT secret is shared across services via configuration (`JWT_SECRET` env). Other services verify tokens locally without calling identity. The token payload includes `userId` and `sessionId`; services use `userId` to attribute writes.

## Migrations

Sequential timestamped migrations under `src/database/migrations/`:

- `1776762269404-init.ts` — `users`, `user_profiles`, `user_settings`, `user_sessions`, `password_resets`.
- `1776849555834-add-language-and-timezone-table.ts` — reference tables.
- `1777856640000-add-locations-table-and-profile-location.ts` — locations and profile FK.
- `1777856641000-add-email-verifications-table.ts` — email verification tokens.

Run via `AUTO_RUN_MIGRATIONS=true` env var or the TypeORM CLI.

## Seeders

Reference tables are populated by `src/database/seeders/{language,timezone,location}.seeder.ts` from the data files in `src/database/factories/datas/`.

Seeders run when `AUTO_RUN_SEEDS=true` and `NODE_ENV !== 'production'` (production seeding is intentionally manual).

## Operational Notes

- **Database:** Postgres. Schema is service-private.
- **Mail:** SMTP via `@nestjs-modules/mailer`, configured in `app.module.ts`.
- **Swagger:** mounted at `/docs`. Internal endpoints should be excluded from the public document — gate with a custom Swagger include filter when `references` module ships.
- **Soft delete:** `users.deleted_at` is the only soft-delete column in this service. All other rows are hard-deleted via cascade.
