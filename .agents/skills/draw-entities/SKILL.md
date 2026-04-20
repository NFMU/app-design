---
name: draw-entities
description: Infer a tactical entity catalog in editable draw.io XML from analysis, aggregate design, and technical docs. Use whenever the user asks for entity inventories, identity-bearing objects, lifecycle-bearing records, or entity-focused DDD documentation.
---

# Draw Entities

Create entity catalog diagrams as editable `.drawio` XML.

## Output Contract

- Write files to `docs/tatical/entities/*.drawio`.
- Emit `00_overview_entities.drawio` by default.
- Keep XML editable and uncompressed.
- Separate aggregate roots from child entities clearly.

## Source Order

1. `docs/analysis/requirements`
2. `docs/analysis/specs`
3. `docs/tatical/aggregate/*.drawio`
4. `docs/technical/erd/*.drawio`
5. `docs/technical/rmd/*.drawio`

## Drawing Rules

- Group entities by bounded context and aggregate.
- Each card should show the entity name, identity hint, and 2-5 core responsibilities or fields.
- Aggregate roots should be visually emphasized over child entities.
- Do not duplicate value objects here; only reference them when needed.
