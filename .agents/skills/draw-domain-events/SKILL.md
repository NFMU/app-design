---
name: draw-domain-events
description: Infer a tactical domain-event catalog in editable draw.io XML from analysis, strategic event storming, and aggregate design docs. Use whenever the user asks for domain event inventories, emitters, consumers, event contracts, or event-focused tactical DDD documentation.
---

# Draw Domain Events

Create domain-event catalog diagrams as editable `.drawio` XML.

## Output Contract

- Write files to `docs/tatical/domain_events/*.drawio`.
- Emit `00_overview_domain_events.drawio` by default.
- Add detail files only when one overview becomes too dense.
- Keep XML editable and uncompressed.

## Source Order

1. `docs/analysis/requirements`
2. `docs/analysis/specs`
3. `docs/strategic/event_storming/*.drawio`
4. `docs/tatical/aggregate/*.drawio`
5. `docs/technical/**/*.drawio`

## What To Show

For each event, capture:

- event name
- emitter aggregate
- short trigger or invariant
- main downstream reaction or consumer

## Drawing Rules

- Group events by bounded context.
- Use one card per event.
- Use arrows only for meaningful downstream reactions, not every possible read model update.
- Keep names exactly aligned with the aggregate and event-storming docs.
