# Microservice Architecture Overview

## Purpose

This document describes the runtime topology of the collaboration platform: which services exist, what data each service owns, and how services collaborate across process boundaries. It is the source of truth for cross-service contracts and is referenced by every other technical document under `docs/technical/`.

## Source Of Truth

The architecture below reflects the **current implementation** in `d:/Workspace/Github/chat-slack/services/`, not aspirational planning.

Last verified against the codebase: 2026-05-07.

---

## Service Inventory

### Currently Implemented

| Service | Path | Status | Primary Responsibility |
|---|---|---|---|
| `identity` | `services/identity` | Operational (auth + profile flows in progress) | Account identity, authentication, sessions, password recovery, email verification, personal profile and settings, **shared reference data** (languages, timezones, locations). |
| `tenants` | `services/tenants` | Skeleton (ORMs only) | Plans entitlement catalog, tenant collaboration boundary, tenant memberships, invitations. |

### Planned (Spec'd but Not Implemented)

| Service | Planned Responsibility | Source Spec |
|---|---|---|
| `channels` | Channels and channel memberships, channel-local preferences. | `docs/analysis/specs/03_channel_management.md` |
| `messaging` | Messages, replies, reads, pins, attachments. | `docs/analysis/specs/04_messaging.md` |
| `rbac` | Roles, permissions, role assignments at platform/tenant/channel scope. | `docs/analysis/specs/05_authorization_model.md` |
| `billing` | Plan prices, subscriptions, invoices, payment transactions, billing policies, overrides, provider events. | `docs/analysis/specs/06_payment_and_subscription_billing.md` |

> The `channels`, `messaging`, `rbac`, and `billing` rows above are **target architecture**, included so that contracts written today (e.g. cross-service IDs in `tenants`) match the eventual layout.

---

## Data Ownership Map

Each table is owned by exactly one service. No service may write to another service's tables. Cross-service reads happen via REST APIs.

| Table | Owning Service | Notes |
|---|---|---|
| `users`, `user_profiles`, `user_settings`, `user_sessions`, `password_resets`, `email_verifications` | `identity` | UUID primary keys. |
| `languages`, `timezones`, `locations` | `identity` | **Shared reference data** — read by every other service via `/reference/*` APIs. Integer primary keys. |
| `plans`, `tenants`, `tenant_members`, `invitations` | `tenants` | Integer primary keys. |
| `channels`, `channel_members` | `channels` (planned) | Integer primary keys per spec. |
| `messages`, `message_reads`, `message_pins`, `message_attachments` | `messaging` (planned) | Integer primary keys per spec. |
| `roles`, `permissions`, `role_permissions`, `tenant_member_roles`, `channel_member_roles` | `rbac` (planned) | Integer primary keys per spec. |
| `plan_prices`, `billing_profiles`, `subscriptions`, `payment_method_refs`, `invoices`, `payment_transactions`, `billing_adjustments`, `billing_overrides`, `billing_policies`, `billing_provider_events` | `billing` (planned) | Integer primary keys per spec. |

---

## Cross-Service Reference Policy

A **cross-service reference** is a column in service A whose value identifies a row owned by service B.

### Rules

1. **No database-level foreign keys across services.** Service A's database does not have an FK constraint pointing into service B's database. The column is a plain typed column (e.g. `integer`, `uuid`).
2. **ID-only references.** The referencing column stores only the primary key value of the referenced row. Snapshots of the referenced row's other fields (display names, codes, etc.) are *not* persisted in the referencing service.
3. **Validation at the application layer.** Before persisting a cross-service reference, the referencing service must call the owning service's API to validate that the ID exists. Validation failures map to HTTP `400` with a domain-specific error code (e.g. `AUTH-INVALID_LANGUAGE`).
4. **No write-through.** The referencing service must never modify the referenced row. Reference data is read-only from outside the owning service.
5. **Caching is allowed.** Reference data with low write frequency (languages, timezones, locations) may be cached in memory or Redis by consumers, with a documented TTL. Cache invalidation on reference change is best-effort; eventual consistency is acceptable for these domains.
6. **Type compatibility.** The cross-service reference column must use a type compatible with the owning side's primary key (e.g. `integer` for `languages.id`, `uuid` for `users.id`).

### Cross-Service Reference Inventory

| Referencing column | Owning table | Type | Validation strategy |
|---|---|---|---|
| `user_profiles.location_id` | `identity.locations` | `integer` | Same-service FK (both in identity) — direct DB constraint applies. |
| `user_settings.language_id` | `identity.languages` | `integer` | Same-service FK. |
| `user_settings.timezone_id` | `identity.timezones` | `integer` | Same-service FK. |
| `tenants.owner_user_id` | `identity.users` | `uuid` | Cross-service. Validated via `GET /internal/users/{id}` before tenant creation. |
| `tenant_members.user_id` | `identity.users` | `uuid` | Cross-service. Validated on invitation acceptance. |
| `tenant_members.invited_by` | `identity.users` | `uuid` | Cross-service. Trusted (set from authenticated caller). |
| `invitations.invited_by` | `identity.users` | `uuid` | Cross-service. Trusted (set from authenticated caller). |
| `invitations.accepted_by_user_id` | `identity.users` | `uuid` | Cross-service. Trusted (set from authenticated caller on accept). |
| `invitations.channel_id` | `channels.channels` | `integer` (planned) | Cross-service when channels service exists. |
| `tenants.timezone_id` | `identity.timezones.uuid` | `uuid` | Cross-service. Validated via `GET /internal/reference/timezones/{uuid}` before tenant create/update. |
| `tenants.language_id` | `identity.languages.uuid` | `uuid` | Cross-service. Validated via `GET /internal/reference/languages/{uuid}` before tenant create/update. |

---

## Inter-Service Communication

### Synchronous (REST)

Services call each other over HTTP using JSON. The request format and error envelope match the public API conventions in `docs/technical/api/00_api_overview.md`.

**Authentication for inter-service calls** is service-to-service rather than user-to-service:

- **Trusted-network model (Phase 1):** services run on a private network and identify themselves via a static service token in the `X-Service-Token` header. Tokens are issued per-service and rotated via configuration.
- **Future:** mTLS or signed JWT with a `svc` claim. Out of scope for Phase 1.

User-context calls (where the inter-service action is performed on behalf of an authenticated end user) propagate the original `Authorization: Bearer <user-jwt>` header. The receiving service validates the token using the same JWT secret as the user-facing endpoints.

### Asynchronous (Events)

Out of scope for Phase 1. Domain events are documented in `docs/strategic/event_storming/` for future phase planning, but no event bus is currently wired up.

### Internal API Path Convention

Endpoints intended for inter-service consumption (not for browsers / mobile clients) are prefixed with `/internal/`. They:

- Require `X-Service-Token` or are restricted by network policy.
- Are not included in the public OpenAPI document served at `/docs`.
- Are documented in service-specific specs under `docs/technical/api/`.

Public-facing reference endpoints (e.g. `GET /reference/languages` for browser consumption) remain at their existing paths and require user JWT.

---

## Service Boundary Diagram

```
                  ┌─────────────────────────────────────────┐
                  │           Public Clients                │
                  │   (Browser / Mobile / CLI users)        │
                  └────────────┬────────────────────────────┘
                               │ HTTPS + user JWT
                               ▼
       ┌───────────────────────────────────────────────────────────┐
       │                    API Gateway / Edge                     │
       │             (planned; direct routing in Phase 1)          │
       └──┬─────────────┬─────────────┬─────────────┬──────────────┘
          │             │             │             │
          ▼             ▼             ▼             ▼
     ┌────────┐    ┌─────────┐   ┌─────────┐   ┌─────────┐
     │identity│    │ tenants │   │channels │   │messaging│   ...
     └───┬────┘    └────┬────┘   └────┬────┘   └────┬────┘
         │              │             │             │
         │  ◄───────────┴─────────────┴─────────────┘
         │     /internal/users/{id}        (validate user IDs)
         │     /reference/languages        (shared reference data)
         │     /reference/timezones
         │     /reference/locations
         │
         └── owns: users, profiles, settings, sessions,
                   languages, timezones, locations
```

Each service owns its own PostgreSQL database (database-per-service). No shared schema, no cross-database joins.

---

## Identifier Conventions

| Resource | Type | Reason |
|---|---|---|
| `users.id` | `uuid` | Implemented; matches `services/identity` source. |
| `languages.id`, `timezones.id`, `locations.id` | `integer` (auto-increment) | Internal PK, used by same-service FK only (`user_settings.language_id`, etc.). |
| `languages.uuid`, `timezones.uuid`, `locations.uuid` | `uuid` (auto-generated) | Cross-service identifier. All consumers outside `identity` must reference these. |
| `tenants.id`, `plans.id`, `tenant_members.id`, `invitations.id` | `integer` (auto-increment) | Matches existing `services/tenants/src/infrastructure/orms/tenant.orm.ts`. |
| Public-facing tenant identifier | `tenants.uuid` | Stable, opaque, exposed in URLs and APIs. |
| Planned channels / messaging / billing IDs | `integer` per current spec | May change to `uuid` if the implementing service chooses; revisit before that service is built. |

API responses prefer the `uuid` form for resources that have one (tenants, users, channels). Reference tables expose integer IDs because they are already integer-keyed.

---

## What This Document Does Not Cover

- Deployment topology (Kubernetes / Docker / VMs).
- Logging, tracing, metrics infrastructure.
- Secret management.
- Database backup and migration strategy beyond noting that each service has its own migration directory.
- Detailed sequence diagrams for individual flows.

These are tracked in operations runbooks (separate document set, out of scope here).

## See Also

- `01_shared_reference_resources.md` — the reference-data API contract (languages, timezones, locations).
- `docs/technical/api/00_api_overview.md` — public REST API conventions.
- `docs/analysis/specs/00_domain_alignment.md` — canonical vocabulary.
- Per-service ORM directories under `services/<name>/src/` — authoritative column definitions.
