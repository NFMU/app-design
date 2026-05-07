# Shared Reference Resources

## Purpose

This document is the authoritative contract for **reference data owned by `identity` and consumed by every other service**. It covers:

- Which tables are reference data.
- Why these are owned by `identity` (and how to challenge that decision).
- The HTTP contract for consumers — both public (user-facing) and internal (service-to-service).
- Caching, validation, and seeding rules.

Read this alongside `00_microservice_overview.md` § *Cross-Service Reference Policy*.

---

## What Counts As Reference Data

A table is reference data if **all** of the following hold:

1. Rows are created by platform operators / migrations, not by end users.
2. Updates are infrequent (typically only via deployments / seed scripts).
3. The table is read by multiple services for validation, display, or filtering.
4. Loss of consistency (e.g. cache lag of seconds to minutes) does not break correctness.

Tables matching these criteria today:

| Table | Owner | Purpose |
|---|---|---|
| `languages` | `identity` | Supported UI languages. Used to validate `user_settings.language_id` and to render language pickers. |
| `timezones` | `identity` | IANA timezones. Used to validate `user_settings.timezone_id` and to display tenant local time. |
| `locations` | `identity` | ISO 3166-1 country list. Used to validate `user_profiles.location_id` and to populate billing-address country dropdowns. |

Future candidates (currency codes, plan codes, RBAC permission codes) follow the same pattern but are owned by their domain services (e.g. `billing` owns currencies; `rbac` owns permission codes).

## Why `identity` Owns These (And When To Move Them)

`identity` is the first service that needed these tables (for `user_profiles` and `user_settings`). The data is generic and not tied to any business domain, so consolidating it here avoids duplication.

Move a reference table out of `identity` if:

- It becomes domain-specific (e.g. a `currency` list that only `billing` updates).
- It needs write access from a non-identity admin flow.
- Its read load on `identity` becomes a hotspot.

Until then, identity is the canonical home.

---

## Schema Recap

The persisted columns are defined in `services/identity/src/database/entities/`. Repeated here for consumer convenience — if this section drifts from the entity files, the entity files win.

### `languages`

| Column | Type | Notes |
|---|---|---|
| `id` | `integer` (auto-increment) | Primary key. **Internal use only.** |
| `uuid` | `uuid` (auto-generated, unique) | **Cross-service identifier.** All non-identity services reference this. |
| `code` | `varchar(10)` unique | BCP 47 language tag, e.g. `en`, `vi`. |
| `locale` | `varchar(100)` unique | Full locale, e.g. `en-US`, `vi-VN`. |
| `name` | `varchar(120)` unique | English display name. |

### `timezones`

| Column | Type | Notes |
|---|---|---|
| `id` | `integer` (auto-increment) | Primary key. **Internal use only.** |
| `uuid` | `uuid` (auto-generated, unique) | **Cross-service identifier.** |
| `name` | `varchar(100)` unique | IANA identifier, e.g. `Asia/Ho_Chi_Minh`. |
| `utc_offset` | `varchar(10)` | Display-only offset, e.g. `+07:00`. Do not use for arithmetic. |

### `locations`

| Column | Type | Notes |
|---|---|---|
| `id` | `integer` (auto-increment) | Primary key. **Internal use only.** |
| `uuid` | `uuid` (auto-generated, unique) | **Cross-service identifier.** |
| `code` | `varchar(2)` unique | ISO 3166-1 alpha-2, e.g. `VN`, `US`. |
| `name` | `varchar(120)` unique | English country name. |

### Identifier Usage Rule

- **Same-service code (within `identity`)** uses the `id` column. `user_settings.language_id` (integer FK) is the canonical example.
- **Cross-service code (any non-identity service)** uses the `uuid` column. `tenants.language_id` stores a `uuid`, not an integer.
- **API responses** include both. `id` is documented as internal; consumers outside identity should bind to `uuid`.

Seed data lives at `services/identity/src/database/factories/datas/{language,timezone,location}.data.ts`. Update seed files via PR; production data is loaded by the seeder during deployment.

---

## API Contract

There are two consumer audiences with different endpoints, auth, and SLAs.

### Public (User-Facing)

Used by the web/mobile UI to populate dropdowns. Already documented in `docs/technical/api/02_profile_settings_api.md` § *Reference Endpoints*.

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/reference/languages` | User JWT | List all languages. |
| `GET` | `/reference/timezones` | User JWT | List all timezones. |
| `GET` | `/reference/locations` | User JWT | List all locations. |

Response uses the standard public envelope:

```json
{
  "success": true,
  "code": "SUCCESS",
  "message": "Request successful",
  "data": {
    "items": [
      { "id": 1, "code": "en", "locale": "en-US", "name": "English" }
    ]
  }
}
```

These endpoints do not paginate today (lists are short). Add pagination only when a single response exceeds ~500 rows.

### Internal (Service-to-Service)

Used by other services to validate cross-service references and to fetch reference rows for display in their own API responses.

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/internal/reference/languages` | `X-Service-Token` | Bulk list with optional `If-None-Match` for caching. |
| `GET` | `/internal/reference/languages/{uuid}` | `X-Service-Token` | Validate / fetch a single language by UUID. |
| `GET` | `/internal/reference/timezones` | `X-Service-Token` | Bulk list. |
| `GET` | `/internal/reference/timezones/{uuid}` | `X-Service-Token` | Validate / fetch a single timezone by UUID. |
| `GET` | `/internal/reference/locations` | `X-Service-Token` | Bulk list. |
| `GET` | `/internal/reference/locations/{uuid}` | `X-Service-Token` | Validate / fetch a single location by UUID. |

**Authentication header:**

```http
X-Service-Token: <static-token-from-config>
```

A missing or invalid header returns:

```http
HTTP/1.1 401 Unauthorized
{
  "success": false,
  "code": "UNAUTHORIZED",
  "message": "Missing or invalid service token."
}
```

**Single-item lookup response (existing):**

```http
GET /internal/reference/languages/f15e9a5b-57f2-4a0b-85b4-bce74a4c9b01
HTTP/1.1 200 OK
{
  "success": true,
  "code": "SUCCESS",
  "data": {
    "uuid": "f15e9a5b-57f2-4a0b-85b4-bce74a4c9b01",
    "id": 1,
    "code": "en",
    "locale": "en-US",
    "name": "English"
  }
}
```

**Single-item lookup response (not found):**

```http
HTTP/1.1 404 Not Found
{
  "success": false,
  "code": "REFERENCE-NOT_FOUND",
  "message": "The requested reference row does not exist."
}
```

**Bulk list response with caching:**

```http
GET /internal/reference/languages
Accept: application/json
If-None-Match: "v3"

HTTP/1.1 200 OK
ETag: "v3"
Cache-Control: max-age=300
{
  "success": true,
  "code": "SUCCESS",
  "data": {
    "version": "v3",
    "items": [
      {
        "uuid": "f15e9a5b-57f2-4a0b-85b4-bce74a4c9b01",
        "id": 1,
        "code": "en",
        "locale": "en-US",
        "name": "English"
      }
    ]
  }
}
```

If the consumer's `If-None-Match` matches the current version, the server returns `304 Not Modified` with no body.

`version` is a monotonic string updated whenever the underlying table is modified (deployment-driven; bumped by the seeder).

---

## Consumer Patterns

### Pattern A: Validate-On-Write (Default)

The consuming service calls `GET /internal/reference/{type}/{uuid}` before persisting a row that references reference data. On `404`, return a domain-specific validation error to the original caller.

Example — `tenants` validating a user-supplied `language_id` (uuid) on tenant create:

```ts
const lang = await identityClient.getLanguage(input.languageId); // input.languageId is a uuid string
if (!lang) {
  throw new BusinessError('AUTH-INVALID_LANGUAGE');
}
// proceed with persistence; tenants.language_id stores the uuid
```

Use this pattern when:

- Writes are infrequent (acceptance of an invitation, profile update).
- Strong validation is needed before touching the consumer's database.

### Pattern B: Snapshot Cache

The consuming service loads the full reference list at startup and on a periodic refresh (default: every 5 minutes), keyed by `version`. Validation is then a local lookup.

Use this pattern when:

- Writes are hot-path (e.g. message send tagged with sender language).
- The reference set is small (under ~10k rows).

Cache invalidation is best-effort. Acceptable lag: minutes.

### Pattern C: Display Hydration

Consumer-side responses that include a reference UUID (e.g. `tenants.timezone_id`) hydrate the full reference object only at the API boundary, not in the database. Typically combined with Pattern B.

```json
// stored in tenants DB
{ "tenant_id": 42, "timezone_id": "57362fa5-91e8-4a91-a3db-945cf498cb75" }

// exposed in tenant API response, hydrated from cache
{
  "tenant_id": 42,
  "timezone": {
    "uuid": "57362fa5-91e8-4a91-a3db-945cf498cb75",
    "name": "Asia/Ho_Chi_Minh",
    "utc_offset": "+07:00"
  }
}
```

---

## Error Codes (Internal API)

| HTTP | Code | Used When |
|---|---|---|
| 401 | `UNAUTHORIZED` | Missing or invalid `X-Service-Token`. |
| 404 | `REFERENCE-NOT_FOUND` | Single-item lookup did not match. |
| 304 | _(no body)_ | `If-None-Match` matched current `version`. |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected failure. |

---

## Open Questions / Trade-offs

These are flagged for the team to decide before the next service is wired up.

1. **Versioning beyond `If-None-Match`.** A simple `version` string works for tables seeded once per deployment. If reference data ever needs hot reload, switch to row-level `updated_at` and let consumers track the latest seen value.

2. **Move to event-driven cache invalidation later.** When the event bus exists (post-Phase 1), publish `LanguageUpdated` etc. so consumers can invalidate without polling.

3. **Auth model migration.** `X-Service-Token` is a Phase 1 expedient. Plan to replace with mTLS or signed service JWTs once the deployment platform supports it.

## Resolved Decisions

- **2026-05-07** — `tenants.timezone` and `tenants.locale` were originally stored as strings to avoid an inter-service hop. Resolved: changed to `tenants.timezone_id` and `tenants.language_id` so all cross-service consumers use the same ID-based pattern. Display strings are hydrated at the API boundary using Pattern C.
- **2026-05-07** — Reference tables (`languages`, `timezones`, `locations`) initially used integer PK only, which is fragile across environments because seed order varies. Resolved: added a `uuid` column (auto-generated, unique) to each. Cross-service columns (`tenants.timezone_id`, `tenants.language_id`) now store UUIDs; same-service FKs (`user_settings.language_id`, `user_settings.timezone_id`, `user_profiles.location_id`) keep the integer `id` for efficiency.
