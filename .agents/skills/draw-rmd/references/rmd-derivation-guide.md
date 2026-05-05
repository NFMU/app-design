# RMD Derivation Guide

## Purpose

Translate `docs/analysis/*` artifacts into a physical relational model diagram that fits the repository's draw.io conventions in `docs/technical/rmd`.
The output must match the repository's preferred RMD style: grid-style HTML tables, four visible columns, orthogonal connectors, and explicit routing checks that prevent overlapping lines.

## Source Precedence

Use the sources in this order:

1. `docs/analysis/requirements`
2. `docs/analysis/specs`
3. `docs/analysis/usecase`
4. `docs/strategic/event_storming`
5. `docs/tatical/aggregate`
6. existing `docs/technical/erd/*.drawio`
7. existing `docs/technical/rmd/*.drawio`

Interpretation rules:

- analysis requirements and specs decide business rules and requiredness
- analysis specs can intentionally collapse conceptual terms into persisted technical terms
- use cases help fill missing nouns, lifecycle hints, and optionality
- strategic and tactical diagrams help confirm lifecycle and aggregate boundaries
- existing technical diagrams are supporting evidence only

If the sources still conflict after comparison, stay conservative:

- omit questionable columns instead of inventing them
- prefer nullable fields when requiredness is unclear
- prefer a simple surrogate `id` on normal business tables when a natural key is not clearly justified
- preserve a documented business table rather than replacing it with a shortcut junction

## Evidence Extraction

Extract structure from the docs in this order:

- persisted nouns and business records -> candidate tables
- verbs and ownership phrases -> candidate relationships
- payload properties, forms, and required fields -> candidate columns
- words like `must`, `always`, `required`, `exactly one` -> not-null and mandatory relationship evidence
- words like `may`, `can`, `optional`, `zero or one` -> nullable and optional relationship evidence

Usually do not persist these unless the docs explicitly say they are stored:

- temporary UI state
- read models used only for query projections
- command names
- policies and events
- display-only aggregates

## What to Draw

Draw these elements:

- tables as grid-style HTML tables
- default visible columns: `Key`, `Property Name`, `Type`, `Nullable`
- direct ER-style lines between parent and child tables

Do not draw:

- diamonds for relationships
- SQL engine-specific syntax
- indexes, triggers, or partitions unless the user asks
- speculative columns or constraints that are not supported by the docs

## Table Rules

Promote something to a table when one or more of these is true:

- it is a persisted business record
- it has its own lifecycle, state, ownership, or approval flow
- other records refer to it directly
- the ERD or aggregate model already treats it as a real entity

Naming rules:

- table names should be lowercase plural `snake_case`
- convert conceptual labels such as `Message Attachment` into `message_attachments`
- if specs define a canonical persisted name, use that name even when requirements use a different user-facing term

Important modeling rules:

- do not invent technical tables unless the physical implementation truly requires them and the docs justify them
- do not flatten a real business record into a pure junction table
- do not use generic audit implications alone as a reason to add a new table

## Column Rules

Use the default four-column mode:

- `Key`
- `Property Name`
- `Type`
- `Nullable`

Rules:

- `Key` contains `PK`, `FK`, `UQ`, `PK FK`, or `UQ FK`
- `Property Name` uses `snake_case`
- `Type` uses common engine-neutral names like `integer`, `string`, `date`, `boolean`, `float`, `text`, `timestamp`
- `Nullable` uses only `yes` or `no`

## Relationship Mapping

### One to Many

- put the foreign key on the child table
- draw the line from parent to child
- use `ERone` and `ERzeroToMany` or `ERoneToMany`

### One to Optional One

- put the foreign key on the optional dependent table
- mark it unique if the dependency must stay one-to-one
- use `ERone` and `ERzeroToOne`

### One to One

- choose the more dependent table to carry the foreign key
- prefer a unique foreign key rather than shared identity unless the docs strongly require shared identity

### Many to Many

- a direct many-to-many relation in the ERD usually becomes a junction table in the RMD
- use a technical junction table only when the relation has no separate lifecycle or payload
- use a business table instead when the relation already has its own business meaning

## Grouping

Default to per-context diagrams when the schema spans multiple bounded contexts.
When the output is split by context, also create one overview schema by default.

Recommended output strategy:

- one file per bounded context
- one mandatory overview file that shows the shared backbone and major cross-context foreign-key paths
- keep external tables as light dependencies instead of duplicating the whole schema in every context file

Overview-specific rule:

- The overview should not repeat every context-detail foreign key.
- Keep the overview readable as a topology map by showing context-local clusters and only the most important cross-context backbone.
- If a foreign-key path would require a long line through other contexts, either move the clusters, use a lightweight external-reference stub near the destination cluster, or leave that relationship to the context-specific RMD.
- Avoid hub-and-spoke canvases where `users`, `tenants`, or another common table sends long lines to many distant contexts.

## Layout Cleanup Rules

After placing the tables, do a dedicated routing cleanup pass.
Use the shared direct routing algorithm in `../../_shared/drawio-routing-algorithm.md` for anchor choice, route candidates, and validation checks.

Use these rules:

- reduce crossings by moving tables before simplifying routes
- prefer straight horizontal lines
- if horizontal is not possible, prefer straight vertical lines
- if straight routing is not possible, use one clean orthogonal elbow
- use a two-elbow dogleg only when an obstacle makes it necessary
- disable curved routing in edge styles with `curved=0`
- enforce unique anchor lanes per table side
- separate parallel routes into distinct lanes of at least `30 px`
- never let a connector pass through another table box
- keep straight edges free of waypoint arrays; use one waypoint for one elbow and two waypoints for a dogleg
- if a route feels squeezed, move tables apart first instead of forcing dense bends

Run these three checks before finishing:

1. **Anchor uniqueness**: no two outgoing edges on the same table reuse the same `(exitX, exitY)`, and no two incoming edges reuse the same `(entryX, entryY)`.
2. **Colinear overlap**: no two horizontal or vertical segments occupy the same corridor for an overlapping interval.
3. **Obstacle intersection**: every segment must stay outside unrelated table bounding boxes.

## Quality Checklist

Before finalizing the `.drawio` file or file set, confirm:

- source precedence was respected
- the RMD implements every important ERD relationship
- pure many-to-many ERD relations became junction tables in the RMD
- business relation entities were preserved instead of duplicated
- foreign keys sit on the correct dependent tables
- one-to-one cases use unique foreign keys when appropriate
- table names are valid plural `snake_case`
- the schema stays conservative when the source docs are incomplete
- glossary alignment from specs was respected
- an overview file exists whenever the output is split by context
- the chosen file split keeps each context understandable without forcing an unnecessary all-in-one schema
