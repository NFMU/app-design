# Profile, Settings, Sessions, And Reference API

These endpoints are planned. They are derived from identity requirements and the current identity service persistence model.

## Resource Shapes

User summary:

```json
{
  "id": "f15e9a5b-57f2-4a0b-85b4-bce74a4c9b01",
  "email": "user@example.com",
  "is_email_verified": false,
  "status": "active"
}
```

Profile:

```json
{
  "display_name": "John Doe",
  "first_name": "John",
  "last_name": "Doe",
  "phone_number": "+1234567890",
  "job_title": "Software Engineer",
  "company": "Acme Inc.",
  "website": "https://www.example.com",
  "location_id": 1,
  "avatar_url": "https://cdn.example.com/avatar.png",
  "date_of_birth": "1990-01-01",
  "bio": "Short profile text"
}
```

Settings:

```json
{
  "language_id": 1,
  "timezone_id": 1,
  "theme": "system",
  "two_factor_enabled": false,
  "marketing_emails_enabled": false
}
```

## Endpoint Summary

| Method | Path | Purpose |
|---|---|---|
| GET | `/me` | Get the authenticated account, profile, and settings. |
| PATCH | `/me/profile` | Update profile fields. |
| GET | `/me/settings` | Get user settings. |
| PATCH | `/me/settings` | Update user settings. |
| GET | `/me/sessions` | List active and recent sessions. |
| DELETE | `/me/sessions/{sessionId}` | Revoke a session. |
| GET | `/reference/languages` | List valid language references. |
| GET | `/reference/timezones` | List valid timezone references. |
| GET | `/reference/locations` | List valid location references. |

## GET `/me`

Success `data`:

```json
{
  "user": {
    "id": "f15e9a5b-57f2-4a0b-85b4-bce74a4c9b01",
    "email": "user@example.com",
    "is_email_verified": false,
    "status": "active"
  },
  "profile": {
    "display_name": "John Doe",
    "first_name": "John",
    "last_name": "Doe"
  },
  "settings": {
    "language_id": 1,
    "timezone_id": 1,
    "theme": "system"
  }
}
```

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 401 | `UNAUTHORIZED` | Missing, expired, or invalid access token. |
| 404 | `USER-NOT_FOUND` | User was not found. |

## PATCH `/me/profile`

Request body accepts any subset of:

| Field | Type | Rules |
|---|---|---|
| `display_name` | string or null | Maximum 120 characters. |
| `first_name` | string or null | Maximum 120 characters. |
| `last_name` | string or null | Maximum 120 characters. |
| `phone_number` | string or null | Maximum 32 characters. |
| `job_title` | string or null | Maximum 120 characters. |
| `company` | string or null | Maximum 120 characters. |
| `website` | string or null | Valid URL, maximum 500 characters. |
| `location_id` | integer or null | Must exist when not null. |
| `avatar_url` | string or null | Valid URL, maximum 500 characters. |
| `date_of_birth` | date or null | ISO date. |
| `bio` | string or null | Free text. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 400 | `AUTH-INVALID_LOCATION` | The selected location does not exist. |
| 401 | `UNAUTHORIZED` | Missing, expired, or invalid access token. |

## PATCH `/me/settings`

Request body accepts any subset of:

| Field | Type | Rules |
|---|---|---|
| `language_id` | integer | Must exist. |
| `timezone_id` | integer | Must exist. |
| `theme` | string | One of `system`, `light`, `dark`. |
| `two_factor_enabled` | boolean | Future-ready setting. |
| `marketing_emails_enabled` | boolean | Email preference. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 400 | `AUTH-INVALID_LANGUAGE` | The selected language does not exist. |
| 400 | `AUTH-INVALID_TIMEZONE` | The selected timezone does not exist. |
| 401 | `UNAUTHORIZED` | Missing, expired, or invalid access token. |

## GET `/me/sessions`

Success `data`:

```json
{
  "items": [
    {
      "id": "57362fa5-91e8-4a91-a3db-945cf498cb75",
      "device_name": "Chrome on Windows",
      "ip_address": "203.0.113.10",
      "last_used_at": "2026-05-05T08:00:00.000Z",
      "expires_at": "2026-06-04T08:00:00.000Z",
      "revoked_at": null
    }
  ]
}
```

## DELETE `/me/sessions/{sessionId}`

Revokes a session belonging to the authenticated user.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 401 | `UNAUTHORIZED` | Missing, expired, or invalid access token. |
| 404 | `AUTH-SESSION_NOT_FOUND` | Session was not found. |
| 409 | `AUTH-SESSION_ALREADY_REVOKED` | Session is already revoked. |

## Reference Endpoints

`GET /reference/languages` returns:

```json
{
  "items": [
    {
      "id": 1,
      "code": "en",
      "locale": "en-US",
      "name": "English"
    }
  ]
}
```

`GET /reference/timezones` returns:

```json
{
  "items": [
    {
      "id": 1,
      "name": "UTC",
      "utc_offset": "+00:00"
    }
  ]
}
```

`GET /reference/locations` returns:

```json
{
  "items": [
    {
      "id": 1,
      "code": "US",
      "name": "United States"
    }
  ]
}
```

