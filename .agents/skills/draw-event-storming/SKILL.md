---
name: draw-event-storming
description: Infer event-storming diagrams in editable draw.io XML from `docs/analysis/*` and existing DDD artifacts. Use whenever the user asks for commands, events, policies, read models, or strategic event storming flows for this repository.
---

# Draw Event Storming

Create strategic event-storming diagrams as editable `.drawio` XML.

## Output Contract

- Write files to `docs/strategic/event_storming/*.drawio`.
- Keep one file per main context by default, for example `01_identity_es.drawio` and `02_workspace_es.drawio`.
- Keep XML editable and uncompressed.
- Represent actors, commands, aggregates, domain events, policies, read models, and external systems with distinct colors.

## Source Order

1. `docs/analysis/requirements`
2. `docs/analysis/specs`
3. `docs/analysis/usecase`
4. existing `docs/strategic/event_storming/*.drawio`
5. `docs/tatical/aggregate/*.drawio`
6. `docs/tatical/domain_events/*.drawio`

## Workflow

1. Split the flows by bounded context.
2. For each context, identify actors, commands, aggregate touchpoints, resulting events, and follow-up policies.
3. Keep the sequence left-to-right.
4. Use read models only when the docs imply a query or projection target.
5. Keep the flow narrative readable before adding extra branches.

## Drawing Rules

- Actor: pale box or actor shape.
- Command: blue.
- Aggregate: amber.
- Event: orange.
- Policy: purple.
- Read model: green.
- External system: magenta.
- Prefer straight or single-elbow connectors.
- Do not overload one canvas with multiple unrelated contexts.
