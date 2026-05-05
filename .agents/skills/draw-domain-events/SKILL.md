---
name: draw-domain-events
description: Infer a tactical domain-event catalog in editable draw.io XML from analysis, strategic event storming, and aggregate design docs. Use whenever the user asks for domain event inventories, emitters, consumers, event contracts, or event-focused tactical DDD documentation.
---

# Draw Domain Events

Create domain-event catalog diagrams as editable `.drawio` XML.
Read [../_shared/drawio-routing-algorithm.md](../_shared/drawio-routing-algorithm.md) before placing connectors.

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
- Route every connector with the shared direct-routing algorithm.
- Use `edgeStyle=orthogonalEdgeStyle;rounded=0;curved=0` for connector styles.
- Prefer straight connectors, then one clean 90-degree elbow, then a two-elbow dogleg only when an obstacle requires it.
- Treat event cards, context groups, notes, and labels as inflated obstacles; connectors must not pass through them.
- Split into detail files before accepting crowded routes with repeated bends or overlapping segments.
