# Change Log

## 2026-05-08 - Tenants Service Shared Types Extraction

### Code Changes — `services/tenants/src/shared/`

Promoted previously inline enum and JSON-shape types from the ORM layer to a service-wide `shared/` folder so application, domain, and presentation code can consume them without depending on `infrastructure/database/orms/*.orm.ts`.

**New files:**

```
services/tenants/src/shared/
├── enums/
│   ├── invitation-role-scope.enum.ts   # InvitationRoleScope
│   ├── invitation-status.enum.ts       # InvitationStatus
│   ├── invitation-type.enum.ts         # InvitationType
│   ├── plan-status.enum.ts             # PlanStatus
│   ├── tenant-member-status.enum.ts    # TenantMemberStatus
│   ├── tenant-status.enum.ts           # TenantStatus
│   └── index.ts
├── types/
│   ├── json-object.type.ts             # JsonValue, JsonObject
│   ├── plan-features.type.ts           # PlanFeaturesJson
│   ├── tenant-branding.type.ts         # TenantBrandingJson
│   ├── tenant-settings.type.ts         # TenantSettingsJson
│   └── index.ts
└── index.ts
```

**Why:** the enums (`TenantStatus`, `PlanStatus`, etc.) had been declared at the top of each `*.orm.ts` file. Anywhere outside the ORM layer that wanted to use them — DTOs, command handlers, validators — would have had to import from `infrastructure/database/orms/...`, which is a clean-architecture layering violation. Hoisting them into `shared/` resolves that.

**JSON-shape interfaces** replace the previous `Record<string, unknown>` typings on the four jsonb columns. `tenants.branding_json` is typed `TenantBrandingJson`; `tenants.settings_json` is `TenantSettingsJson`; `plans.features_json` is `PlanFeaturesJson`. Documented fields match the spec examples; `PlanFeaturesJson` is intentionally indexable for plan-specific extension flags.

**ORM imports updated:**

- `tenant.orm.ts` — `TenantStatus`, `TenantBrandingJson`, `TenantSettingsJson` from `src/shared`.
- `tenant-member.orm.ts` — `TenantMemberStatus` from `src/shared/enums`.
- `plan.orm.ts` — `PlanStatus`, `PlanFeaturesJson` from `src/shared`.
- `invitation.orm.ts` — `InvitationStatus`, `InvitationType`, `InvitationRoleScope` from `src/shared/enums`.

ORM file paths reflect the in-progress reorganization to `services/tenants/src/infrastructure/database/orms/` (was `infrastructure/orms/`).

### Operational Notes

- DTOs and command handlers should now import enums directly from `src/shared/enums` (or `src/shared` barrel) rather than from ORM modules.
- `JsonObject` / `JsonValue` aliases are available for genuinely free-form payload columns (e.g. provider event payloads) where a structured interface would be premature.

## 2026-05-07 - Reference Tables UUID Column

### Schema Changes

Added `uuid` column to identity-owned reference tables (`languages`, `timezones`, `locations`) so cross-service consumers can hold a stable, environment-agnostic identifier instead of an integer that depends on seed order.

| Table | New column | Type | Constraints |
|---|---|---|---|
| `languages` | `uuid` | `uuid` | NOT NULL, unique (`uq_languages_uuid`), DB-generated via `uuid_generate_v4()`. |
| `timezones` | `uuid` | `uuid` | NOT NULL, unique (`uq_timezones_uuid`), DB-generated. |
| `locations` | `uuid` | `uuid` | NOT NULL, unique (`uq_locations_uuid`), DB-generated. |

Cross-service consumer columns updated to UUID:

- `tenants.timezone_id`: `integer` → `uuid` (references `identity.timezones.uuid`).
- `tenants.language_id`: `integer` → `uuid` (references `identity.languages.uuid`).

Same-service FKs in identity (`user_settings.language_id`, `user_settings.timezone_id`, `user_profiles.location_id`) remain integer — these are intra-DB and benefit from the smaller key.

### Code Changes — `services/identity`

- Updated entities `language.entity.ts`, `timezone.entity.ts`, `location.entity.ts` with `@Generated('uuid')` column.
- New migration `1778500000000-add-reference-uuid-columns.ts` adds the columns and unique indexes; backfills existing rows via `uuid_generate_v4()` default.
- Reference outputs (`LanguageOutput`, `TimezoneOutput`, `LocationOutput`) now return both `uuid` (cross-service) and `id` (internal).
- Reference repository exposes `getXxxByUuid` (cross-service) and keeps `getXxxById` (internal-only, documented as such).
- Public `/reference/{type}/{uuid}` and internal `/internal/reference/{type}/{uuid}` endpoints now take UUID in the path (was integer ID).
- `ReferenceController` no longer requires JWT auth — reference lists are open data.

### Code Changes — `services/tenants`

- `tenant.orm.ts`: `timezoneId` and `languageId` columns retyped to `uuid` (`string` in TypeScript).

### Doc Changes

- `docs/analysis/specs/01_identity_and_profile.md` — added `uuid` row to each reference table; clarified usage rule.
- `docs/analysis/specs/02_collaboration_foundation.md` — `tenants.{timezone,language}_id` documented as `uuid` referencing `identity.{timezones,languages}.uuid`.
- `docs/technical/architecture/00_microservice_overview.md` — split identifier conventions into internal `id` vs cross-service `uuid` rows; updated cross-service reference inventory.
- `docs/technical/architecture/01_shared_reference_resources.md` — schema recap, API examples, and consumer patterns now show UUID; added a *Resolved Decisions* entry.
- `docs/technical/architecture/02_identity_service.md` — owned-tables row notes dual key; internal API table shows UUID paths.
- `docs/technical/architecture/03_tenants_service.md` — outbound dependencies and cross-service reference table updated to UUID.

### Operational Notes

- The migration uses `uuid_generate_v4()` from the `uuid-ossp` extension. The extension is already enabled by the initial migration (`PrimaryGeneratedColumn('uuid')` on `users`); the new migration re-issues `CREATE EXTENSION IF NOT EXISTS` for safety.
- Existing rows in deployed databases will receive freshly generated UUIDs, which means the `uuid` value differs across environments. Cross-service consumers must read the UUID from the relevant API rather than hardcoding it. If deterministic UUIDs are needed (e.g. for fixture-driven tests), plan a follow-up to seed UUIDs by `code` mapping.

## 2026-05-07 - Reference ID Migration And Per-Service Architecture Docs

### Schema Changes

- `tenants.timezone` (varchar(100) string) replaced with `tenants.timezone_id` (integer, cross-service reference to `identity.timezones`).
- `tenants.locale` (varchar(20) string) replaced with `tenants.language_id` (integer, cross-service reference to `identity.languages`).
- Updated `services/tenants/src/infrastructure/orms/tenant.orm.ts` to match.
- Updated `docs/analysis/specs/02_collaboration_foundation.md` column documentation.
- Resolved the open question previously raised in `01_shared_reference_resources.md`: all cross-service references to identity-managed reference data now use IDs.

### Added — Per-Service Architecture Docs

Six new documents under `docs/technical/architecture/` describing each service in detail (responsibility, owned tables, module layout, public + internal API surfaces, inter-service dependencies, cross-service reference columns):

- `02_identity_service.md` (implemented).
- `03_tenants_service.md` (skeleton).
- `04_channels_service.md` (planned).
- `05_messaging_service.md` (planned).
- `06_rbac_service.md` (planned).
- `07_billing_service.md` (planned).

### Added — Reference API Implementation In `services/identity`

- `src/modules/references/` — full module implementing the contract from `01_shared_reference_resources.md`:
  - `reference.controller.ts` exposes public `/reference/{type}` and `/reference/{type}/{id}` (Bearer auth).
  - `internal-reference.controller.ts` exposes `/internal/reference/{type}` and `/internal/reference/{type}/{id}` (service-token auth).
  - `reference.service.ts`, `reference.repository.ts`, `outputs/{language,timezone,location}.output.ts`.
- `src/core/guards/service-token.guard.ts` — validates `X-Service-Token` against `INTERNAL_SERVICE_TOKEN` env var.
- `src/core/decorators/service-auth.decorator.ts` — Swagger header decorator for documenting internal endpoints.
- `src/core/errors/reference.error.ts` — error catalog (`REFERENCE-{LANGUAGE,TIMEZONE,LOCATION}_NOT_FOUND`, `REFERENCE-SERVICE_TOKEN_INVALID`).
- `app.module.ts` updated to import `ReferenceModule`.

### Operational Notes

- Set `INTERNAL_SERVICE_TOKEN` in the deployment environment of every service that calls identity.
- The current implementation does not yet add ETag / `If-None-Match` support; reference tables lack `updated_at` columns. Documented as future enhancement.
- Internal endpoints are mounted alongside public endpoints in the same Nest application; they should be excluded from the public OpenAPI document via a Swagger include filter (deferred).

## 2026-05-07 - Microservice Architecture Documentation

### Added

- Added `docs/technical/architecture/00_microservice_overview.md` — system design document covering service inventory (`identity`, `tenants` implemented; `channels`, `messaging`, `rbac`, `billing` planned), per-service data ownership map, inter-service communication conventions, and identifier types.
- Added `docs/technical/architecture/01_shared_reference_resources.md` — formal contract for the `languages`, `timezones`, and `locations` reference tables owned by `identity` and consumed cross-service, including public `/reference/*` and internal `/internal/reference/*` API surfaces.

### Updated

- Updated `docs/analysis/specs/00_domain_alignment.md` with explicit service-to-table ownership assignments and a *Cross-Service Reference Rule* section: ID-only references, no cross-database foreign keys, validation at the application layer, hydration at the API boundary.

### Modeling Decisions

- Reference data with low write frequency (`languages`, `timezones`, `locations`) lives in `identity`. Other services consume by ID only.
- Cross-service columns use a type compatible with the owning side's PK (`uuid` for `users.id`, `integer` for reference IDs). No database-level FK constraints span service boundaries.
- Inter-service authentication uses a static `X-Service-Token` in Phase 1; mTLS / signed service JWTs are deferred.
- `tenants.timezone` and `tenants.locale` continue to store IANA / locale strings directly in Phase 1 to avoid an inter-service hop on tenant reads, with an open question to revisit when `billing` reporting requires stable IDs.

## 2026-05-06 - Workspace Separation Rollback

### Updated

- Reverted the experimental workspace-separation pass and restored the previous Phase 1 simplification.
- Restored `tenants` as the persisted collaboration boundary for workspace business flows.
- Removed separate workspace REST endpoints, `workspaces` table modeling, `workspace_members`, `workspace_member_roles`, and `workspace_id` channel ownership from current technical artifacts.
- Restored channel specs, API, ERD, RMD, aggregates, repositories, and event-storming diagrams so channels operate inside the tenant boundary.
- Restored RBAC requirements, specs, API, ERD, RMD, and tactical diagrams to platform, tenant, channel, and self scopes.

### Modeling Decisions

- Workspace remains valid business language in requirements and diagrams.
- Phase 1 persistence and REST APIs do not introduce a separate workspace table or resource.
- Billing remains tenant-scoped and does not duplicate workspace-level payment records in Phase 1.

## 2026-05-06 - Payment And Subscription Billing Analysis

### Added

- Added payment and subscription billing requirements in `docs/analysis/requirements/06_payment_and_subscription_billing.md`.
- Added billing persistence/spec guidance in `docs/analysis/specs/06_payment_and_subscription_billing.md`.
- Added billing use-case diagram in `docs/analysis/usecase/07_payment_subscription_billing.drawio`.
- Added billing event-storming diagram in `docs/strategic/event_storming/07_billing_es.drawio`.
- Added billing aggregate diagram in `docs/tatical/aggregate/07_billing_agg.drawio`.
- Added billing REST API contract in `docs/technical/api/07_billing_api.md`.
- Added billing ERD in `docs/technical/erd/07_billing_erd.drawio`.
- Added billing RMD in `docs/technical/rmd/07_billing.drawio`.

### Updated

- Updated the Phase 1 requirements overview with `Billing Admin`, subscription, billing profile, invoice, payment method reference, and payment transaction concepts.
- Updated tenant/workspace requirements to distinguish entitlement plan from billable subscription state.
- Updated domain alignment and collaboration foundation specs to keep `plans` as the entitlement catalog and model billing lifecycle separately.
- Updated use-case, context-map, aggregate, entity, value-object, repository, domain-event, ERD, RMD, and API overviews so the new billing context is discoverable.
- Renamed tenant technical overview labels from "Tenant & Subscription" to "Tenant & Entitlement" to avoid conflating tenant plan entitlement with subscription billing lifecycle.
- Added RMD historical notes clarifying that billing tables are a separate 2026-05-06 context and are not part of the earlier tenant/workspace refactoring comparison.

### Completed Alignment Pass

- Added tenant-scoped billing permissions, seeded `Billing Admin` guidance, and billing permission examples to RBAC requirements, specs, API, and diagrams.
- Added `billing_overrides` and `billing_policies` to billing requirements, specs, aggregate, ERD, RMD, and overview diagrams.
- Added a tenant entitlement summary API so tenant/workspace documents expose plan, subscription, override, and billing access state without exposing sensitive payment data.
- Clarified that workspace remains business language while Phase 1 technical persistence and REST APIs use the `tenants` boundary.
- Fixed billing RMD invoice breakdown and provider event uniqueness by adding `subtotal_minor`, `tax_minor`, and composite provider/event uniqueness guidance.
- Clarified that payment-provider events can reconcile subscription, invoice, payment method, and payment transaction state.

### Modeling Decisions

- A tenant may have at most one active subscription in Phase 1, while historical subscriptions can remain for audit.
- Billing records restrict or restore tenant access through policy; they do not replace tenant lifecycle records or delete collaboration data.
- Raw card, bank account, and sensitive payment credential data remain outside the platform data store.
- Payment-provider callbacks are recorded through `billing_provider_events` and must be processed idempotently.
