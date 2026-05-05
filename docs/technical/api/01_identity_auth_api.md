# Identity And Auth API

## Implementation Status

| Method | Path | Status | Notes |
|---|---|---|---|
| POST | `/auth/login` | Implemented | Matches `loginInputSchema` and returns `LoginOutput`. |
| POST | `/auth/signup` | Implemented with response gap | Creates user, profile, and settings, but the service currently returns `null` instead of `LoginOutput`. |
| POST | `/auth/email-verification/verify` | Planned | Verifies the token from the signup verification email and marks the user's email as verified. |
| POST | `/auth/logout` | Planned | Session persistence exists but the endpoint is not implemented. |
| POST | `/auth/password-reset/request` | Planned | `password_resets` persistence exists but the endpoint is not implemented. |
| POST | `/auth/password-reset/confirm` | Planned | `password_resets` persistence exists but the endpoint is not implemented. |
| PATCH | `/auth/password` | Planned | Required by identity requirements, not implemented. |

## POST `/auth/login`

Public endpoint for exchanging email and password for an access token.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `email` | string | yes | Valid email. Lowercased before lookup. |
| `password` | string | yes | Minimum 8 characters. |

Success response:

```json
{
  "success": true,
  "code": "SUCCESS",
  "message": "Request successful",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 401 | `AUTH-INVALID_CREDENTIALS` | The provided credentials are invalid. |
| 500 | `INTERNAL_SERVER_ERROR` | Internal server error. |

## POST `/auth/signup`

Public endpoint for confirming the signup form and creating an account. The implemented service creates rows in `users`, `user_profiles`, and `user_settings` in one transaction, then publishes `UserCreatedEvent`.

After the signup form is confirmed, the backend must send a verification email to the submitted mail address. That email contains a frontend link with a verification token, for example:

```text
https://app.example.com/verify-email?token=<email_verification_token>
```

When the user clicks the link, the frontend reads the token from the URL and calls `POST /auth/email-verification/verify`.

Implementation note: the current source sends a welcome email from `MailHandler.handleUserCreated`, but it does not yet generate or persist an email verification token. The source does already have `users.is_email_verified`, which should be set to `true` after token verification.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `email` | string | yes | Valid email. Lowercased before persistence. |
| `password` | string | yes | Minimum 8 characters. |
| `confirmPassword` | string | yes | Must match `password`. |
| `firstName` | string | yes | 1 to 120 characters. |
| `lastName` | string | yes | 1 to 120 characters. |
| `phoneNumber` | string | no | Maximum 32 characters. |
| `jobTitle` | string | no | Maximum 120 characters. |
| `company` | string | no | Maximum 120 characters. |
| `website` | string | no | Valid URL, maximum 500 characters. |
| `location_id` | integer | no | Positive integer. Must exist when provided. |
| `bio` | string | no | Free text. |
| `language_id` | integer | no | Positive integer. Defaults to `1`. Must exist. |
| `timezone_id` | integer | no | Positive integer. Defaults to `1`. Must exist. |

Intended success response:

```json
{
  "success": true,
  "code": "SUCCESS",
  "message": "Request successful",
  "data": null
}
```

Current source behavior: `AuthService.signup` is typed as `Promise<LoginOutput>` but returns `null`. Until that is fixed, the running service returns a success envelope with `data: null`.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 400 | `AUTH-PASSWORD_MISMATCH` | Password and confirmation password do not match. |
| 400 | `AUTH-INVALID_LANGUAGE` | The selected language does not exist. |
| 400 | `AUTH-INVALID_TIMEZONE` | The selected timezone does not exist. |
| 400 | `AUTH-INVALID_LOCATION` | The selected location does not exist. |
| 409 | `AUTH-EMAIL_ALREADY_EXISTS` | A user with this email already exists. |
| 500 | `INTERNAL_SERVER_ERROR` | Internal server error. |

## POST `/auth/email-verification/verify`

Verifies the token from the signup verification email and marks the user's email address as verified.

Status: planned. This route is not implemented in the current identity controller.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `token` | string | yes | Email verification token from the frontend link. |

Success response:

```json
{
  "success": true,
  "code": "SUCCESS",
  "message": "Request successful",
  "data": {
    "verified": true,
    "user_id": "f15e9a5b-57f2-4a0b-85b4-bce74a4c9b01",
    "email": "user@example.com"
  }
}
```

Persistence note: implementation should store only a hashed verification token with an expiry time. The current source schema does not yet include an email verification token table or fields.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 400 | `AUTH-EMAIL_VERIFICATION_INVALID` | Verification token is invalid. |
| 400 | `AUTH-EMAIL_VERIFICATION_EXPIRED` | Verification token has expired. |
| 404 | `USER-NOT_FOUND` | User was not found. |
| 409 | `AUTH-EMAIL_ALREADY_VERIFIED` | Email address is already verified. |

## Planned Auth Endpoints

### POST `/auth/logout`

Revokes the caller's current session token.

Request body: none.

Success `data`:

```json
{
  "revoked": true
}
```

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 401 | `UNAUTHORIZED` | Missing, expired, or invalid access token. |
| 404 | `AUTH-SESSION_NOT_FOUND` | Active session was not found. |

### POST `/auth/password-reset/request`

Creates a time-bound password reset request for an email address.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `email` | string | yes | Valid email. |

Success `data`:

```json
{
  "accepted": true
}
```

Do not reveal whether the email exists.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 429 | `AUTH-PASSWORD_RESET_RATE_LIMITED` | Too many password reset requests. |

### POST `/auth/password-reset/confirm`

Completes a password reset using a reset token.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `token` | string | yes | Reset token from email. |
| `password` | string | yes | Minimum 8 characters. |
| `confirmPassword` | string | yes | Must match `password`. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `AUTH-PASSWORD_MISMATCH` | Password and confirmation password do not match. |
| 400 | `AUTH-PASSWORD_RESET_INVALID` | Reset token is invalid. |
| 400 | `AUTH-PASSWORD_RESET_EXPIRED` | Reset token has expired. |
| 409 | `AUTH-PASSWORD_RESET_USED` | Reset token has already been used. |

### PATCH `/auth/password`

Changes the authenticated user's password.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `currentPassword` | string | yes | Current password. |
| `newPassword` | string | yes | Minimum 8 characters. |
| `confirmPassword` | string | yes | Must match `newPassword`. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `AUTH-PASSWORD_MISMATCH` | Password and confirmation password do not match. |
| 401 | `AUTH-INVALID_CREDENTIALS` | The provided credentials are invalid. |
