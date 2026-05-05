---
name: draw-aggregate
description: Infer DDD aggregate design diagrams in editable draw.io XML from `docs/analysis/*`, strategic flows, and technical docs. Use whenever the user asks for aggregate roots, aggregate boundaries, entities, value objects, or tactical DDD aggregate design for this repository.
---

# Draw Aggregate

Create tactical aggregate diagrams as editable `.drawio` XML.
Read [../_shared/drawio-routing-algorithm.md](../_shared/drawio-routing-algorithm.md) before placing connectors.

## Output Contract

- Write files to `docs/tatical/aggregate/*.drawio`.
- Emit `00_overview_agg.drawio` plus per-context files when multiple aggregates exist.
- Keep XML editable and uncompressed.
- Show aggregate roots, entities, value objects, and emitted domain events.

## Source Order

1. `docs/analysis/requirements`
2. `docs/analysis/specs`
3. `docs/analysis/usecase`
4. `docs/strategic/event_storming/*.drawio`
5. existing `docs/tatical/aggregate/*.drawio`
6. `docs/technical/**/*.drawio`

## Workflow

1. Identify aggregate roots from lifecycle and transactional boundaries.
2. Place entities and value objects inside the owning aggregate boundary.
3. Show cross-aggregate references by ID only.
4. List emitted domain events near the owning aggregate.
5. Keep the overview file cross-context and keep detail files focused on one aggregate.

## Drawing Rules

- Aggregate root card: amber header.
- Entity card: blue header.
- Value object card: purple header.
- Domain event card: orange header.
- Use package or section headers for each aggregate context.
- Use dashed connectors for cross-aggregate references and label them as ID-based references when needed.
- Route every connector with the shared direct-routing algorithm.
- Use `edgeStyle=orthogonalEdgeStyle;rounded=0;curved=0` for connector styles.
- Prefer straight connectors, then one clean 90-degree elbow, then a two-elbow dogleg only when an obstacle requires it.
- Treat cards, package boundaries, event notes, and labels as inflated obstacles; connectors must not pass through them.
- Move shapes apart or split the file before accepting dense bends, colinear overlaps, or shape intersections.
