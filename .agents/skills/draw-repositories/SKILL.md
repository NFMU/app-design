---
name: draw-repositories
description: Infer tactical repository documentation in editable draw.io XML from analysis, aggregate design, and technical docs. Use whenever the user asks for repository boundaries, persistence interfaces, aggregate loading rules, or repository-focused DDD documentation.
---

# Draw Repositories

Create repository diagrams as editable `.drawio` XML.

## Output Contract

- Write files to `docs/tatical/repositories/*.drawio`.
- Emit `00_overview_repositories.drawio` by default.
- Keep XML editable and uncompressed.
- Model repositories around aggregate roots, not arbitrary tables.

## Source Order

1. `docs/analysis/requirements`
2. `docs/analysis/specs`
3. `docs/tatical/aggregate/*.drawio`
4. `docs/tatical/entities/*.drawio`
5. `docs/technical/rmd/*.drawio`

## Workflow

1. Identify one repository per aggregate root unless the docs justify a separate persistence boundary.
2. Show the aggregate root served by the repository.
3. List only the main load, save, lookup, and existence-check methods needed by the domain.
4. Keep complex reporting queries out of repository diagrams unless they are part of the domain contract.

## Drawing Rules

- Use one interface-style card per repository.
- Include the aggregate root it serves and the key query methods.
- Add notes for important persistence constraints such as uniqueness, optimistic locking, or ID-based cross-aggregate references when justified.
