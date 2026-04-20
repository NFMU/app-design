---
name: draw-value-objects
description: Infer tactical value-object documentation in editable draw.io XML from analysis, aggregate design, and technical docs. Use whenever the user asks for immutable concepts, value object catalogs, reusable domain concepts, or value-object-focused DDD documentation.
---

# Draw Value Objects

Create value-object catalog diagrams as editable `.drawio` XML.

## Output Contract

- Write files to `docs/tatical/value_objects/*.drawio`.
- Emit `00_overview_value_objects.drawio` by default.
- Keep XML editable and uncompressed.

## Source Order

1. `docs/analysis/requirements`
2. `docs/analysis/specs`
3. `docs/tatical/aggregate/*.drawio`
4. `docs/technical/**/*.drawio`

## Workflow

1. Identify immutable concepts used to model meaning rather than identity.
2. Keep the list conservative; do not invent value objects with no support in the docs.
3. Show where each value object is used.

## Drawing Rules

- Group value objects by bounded context.
- Each card should show the value object name, its main attributes, and where it is applied.
- Use dashed usage connectors from aggregate roots or entities to value objects.
