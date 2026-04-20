# Identity And Profile Requirements

## Business Outcome

Users must be able to register, authenticate, manage account security, and maintain a personal profile that travels with them across the collaboration platform.

## Functional Requirements

- A guest can register a new account with email and password.
- An invited user can accept an invitation and continue onboarding even if they do not yet have an account.
- A member can log in and log out.
- A member can request a password reset and complete the reset by using a verification token.
- A member can change their password while authenticated.
- A member can view active sessions and revoke the current device session.
- A member can update display name, avatar, job title, internal phone, language, and timezone.
- A member can set a basic online or offline presence state.
- A member can configure basic notification preferences.
- A member can manage basic personal preferences, including pinned-channel behavior and other display preferences.

## Business Rules

- Each user account must have exactly one profile record and exactly one settings or preference record.
- A user can have zero or many active sessions over time.
- Password reset requests are time-bound and must expire.
- Personal preferences belong to the user account, not to a tenant administrator.
- Profile changes must not require tenant-level approval in Phase 1.

## Persisted Concepts Expected By Downstream Design

- user account
- user profile
- user settings or preferences
- user session
- password reset request
