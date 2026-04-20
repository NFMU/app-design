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
- `tenants`: persisted collaboration boundary for Phase 1 technical design
- `tenant_members`: persisted participation of a user in that boundary
- `channels`: discussion spaces inside `tenants`
- `channel_members`: persisted user-channel participation with local preferences

## Conceptual To Technical Mapping

The business requirements still use both `tenant` and `workspace` language.
For Phase 1 technical design, use this mapping:

- the conceptual `workspace` collaboration boundary is persisted on the `tenants` boundary
- the "default workspace" requirement is satisfied by creating the tenant in an immediately usable collaboration state
- `Tenant Admin` and `Workspace Admin` are distinct business roles but operate over the same persisted boundary in Phase 1
- no separate `workspaces` table is introduced in Phase 1 technical diagrams unless a future spec explicitly restores multi-workspace persistence

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
