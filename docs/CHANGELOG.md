# Change Log

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
