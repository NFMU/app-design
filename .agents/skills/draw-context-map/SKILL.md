---
name: draw-context-map
description: Infer a DDD context map in editable draw.io XML from `docs/analysis/*`, strategic artifacts, tactical artifacts, and existing technical docs. Use whenever the user asks for bounded contexts, upstream/downstream relationships, strategic DDD design, or a context map for this repository.
---

# Draw Context Map

Create a strategic DDD context map as editable `.drawio` XML.
Read [../_shared/drawio-routing-algorithm.md](../_shared/drawio-routing-algorithm.md) before placing connectors.

## Output Contract

- Write the final diagram to `docs/strategic/context_map/00_context_map.drawio`.
- Keep the XML editable and uncompressed.
- Model one bounded context per main box.
- Show upstream/downstream direction explicitly on every relationship.
- Use conservative relationship labels such as `customer/supplier`, `conformist`, `partnership`, or `shared kernel` only when the docs justify them.
- Add one small legend or note that explains the connector vocabulary.

## Source Order

Read sources in this order:

1. `docs/analysis/requirements`
2. `docs/analysis/specs`
3. `docs/analysis/usecase`
4. `docs/strategic/event_storming/*.drawio`
5. `docs/tatical/aggregate/*.drawio`
6. `docs/tatical/domain_events/*.drawio`
7. `docs/tatical/repositories/*.drawio`
8. `docs/technical/**/*.drawio`

If the docs disagree, prefer the higher-priority business wording from `analysis`, then tactical evidence, then technical evidence.

## Workflow

1. Identify the bounded contexts and their core responsibilities.
2. Decide which contexts are upstream, downstream, or collaborative peers.
3. Preserve the business distinction between `tenant` and `workspace` in strategic language when useful, but call out the Phase 1 mapping to `tenants` in a note.
4. Draw the map with roomy spacing and orthogonal connectors.
5. Keep the overview strategic. Do not turn this into an ERD or table map.

## Drawing Rules

- Use one large rounded rectangle per bounded context.
- Include 2-4 short bullets inside each context describing its responsibility.
- Use arrow labels for the relationship pattern and direction.
- Route every connector with the shared direct-routing algorithm.
- Use `edgeStyle=orthogonalEdgeStyle;rounded=0;curved=0` for connector styles.
- Prefer straight connectors, then one clean 90-degree elbow, then a two-elbow dogleg only when an obstacle requires it.
- Avoid crossing lines by repositioning contexts, choosing opposite-side anchors, or increasing gutters before adding bends.
- Treat context boxes, legend notes, labels, and group boundaries as inflated obstacles; connectors must not pass through them.
- Keep the canvas overview-first and readable in one screen.
