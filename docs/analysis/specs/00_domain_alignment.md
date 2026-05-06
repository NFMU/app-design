# Domain Alignment Spec

## Purpose

This folder translates business requirements into canonical vocabulary and persistence assumptions for downstream technical artifacts such as ERD and RMD diagrams.

## Source Precedence For Technical Artifacts

When generating technical documentation, use this order:

1. `docs/analysis/requirements`
2. files in `docs/analysis/specs`
3. `docs/analysis/usecase`
4. `docs/strategic/event_storming`
5. `docs/tatical/aggregate`
6. existing `docs/technical/*`

## Canonical Vocabulary

- `users`: account owners authenticated by the platform
- `plans`: commercial and capability catalog assignable to tenant accounts
- `tenants`: persisted collaboration boundary for Phase 1 technical design
- `tenant_members`: persisted participation of a user in that boundary
- `channels`: discussion spaces inside `tenants`
- `channel_members`: persisted user-channel participation with local preferences
- `plan_prices`: billable price options for a product or entitlement plan
- `billing_profiles`: billing contact and business details for tenant accounts
- `subscriptions`: billing lifecycle that applies a plan and billing period to a tenant
- `payment_method_refs`: display-safe references to provider-managed payment methods
- `invoices`: issued billing documents for subscription periods or adjustments
- `payment_transactions`: recorded payment collection, failure, settlement, refund, or credit activity
- `billing_adjustments`: approved credit, refund, write-off, or manual billing correction records
- `billing_overrides`: approved billing exceptions that can satisfy access policy without provider payment
- `billing_policies`: grace period, dunning, and restriction settings used for billing-state evaluation
- `billing_provider_events`: idempotent records of external payment-provider callbacks

## Conceptual To Technical Mapping

The business requirements still use both `tenant` and `workspace` language.
For Phase 1 technical design, use this mapping:

- the conceptual `workspace` collaboration boundary is persisted on the `tenants` boundary
- the "default workspace" requirement is satisfied by creating the tenant in an immediately usable collaboration state
- `Tenant Admin` and `Workspace Admin` are distinct business roles but operate over the same persisted boundary in Phase 1
- no separate `workspaces` table is introduced in Phase 1 technical diagrams unless a future spec explicitly restores multi-workspace persistence
- `tenants.plan_id` represents the current entitlement plan for Phase 1 and may be synchronized from the active subscription when subscription billing is enabled
- tenant API responses may expose a read-only entitlement summary derived from billing records, but payment state remains outside the `tenants` persistence model
- payment method credentials are not modeled as internal business data; only provider references and display-safe metadata are persisted

This rule is intentionally stronger than older diagrams because it reflects the current simplified technical target.

## Diagram Conventions

- ERD output goes to `docs/technical/erd/*.drawio`
- RMD output goes to `docs/technical/rmd/*.drawio`
- Strategic DDD artifacts go to `docs/strategic/**/*.drawio`
- Tactical DDD artifacts go to `docs/tatical/**/*.drawio`
- Diagrams are stored as editable, uncompressed draw.io XML
- Repository naming in technical diagrams prefers plural `snake_case`

## Persistence Principles

- keep the model conservative and avoid speculative tables
- only persist concepts with lifecycle, ownership, query value, or cross-context references
- use notes in diagrams only for material assumptions such as the workspace-to-tenant mapping
