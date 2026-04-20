# ERD Derivation Guide

## Purpose

Translate `docs/analysis/*` artifacts into a conceptual ERD that models persisted business data for the technical layer.
The output must match the repository's draw.io ERD style: rectangles for entities, diamonds for relationships, symbolic markers on orthogonal lines, and readable routing without line collisions.

## Source Precedence

When the artifacts disagree, use this precedence:

1. `docs/analysis/requirements`
2. `docs/analysis/specs`
3. `docs/analysis/usecase`
4. `docs/strategic/event_storming`
5. `docs/tatical/aggregate`
6. existing `docs/technical/erd/*.drawio`

Treat existing technical diagrams as supporting evidence, not the primary source of truth.
If `docs/analysis/specs` defines canonical vocabulary or a conceptual-to-technical mapping, follow it over older diagrams.
If the conflict remains unresolved, prefer the more conservative model:

- keep optionality rather than inventing mandatory participation
- keep direct many-to-many rather than inventing a business entity
- avoid speculative nouns that appear only once without persistence evidence

## Evidence Extraction

Before drawing, extract three things from the docs:

1. persisted nouns -> candidate entities
2. business verbs -> candidate relationships
3. modal words such as `must`, `may`, `optional`, `one or more`, `zero or many` -> cardinality evidence

If a noun only appears as UI state, a command step, or a temporary read model, do not promote it by default.

## What to Draw

Draw these elements only:

- entities as rectangles
- relationships as diamonds
- symbolic markers on relationship lines

Do not draw:

- attribute lists
- technical columns
- PK/FK/type metadata
- tables that exist only to implement a pure many-to-many relation

## Naming Rules

Use these label rules:

- prefer plural `snake_case` entity names such as `users`, `tenant_members`, `message_attachments`
- keep relationship labels short and verb-first, such as `has`, `joins`, `requests`, `assigned to`
- preserve glossary decisions from specs when conceptual terms differ from persisted terms

Example:

- if requirements mention `workspace` but specs say the current technical layer persists that concept as `tenants`, draw `tenants` and capture the assumption with a note when needed

## Entity Rules

Promote a noun to an entity when one or more of these is true:

- it must be stored and queried later
- it has a lifecycle or status
- it has ownership or audit meaning
- it appears across multiple use cases as a reusable business record
- the tactical aggregate model treats it as a persisted aggregate root or child entity

Usually exclude these unless persistence is explicit:

- screens
- buttons
- commands, events, and policies
- read models that do not materialize as persisted records in the technical layer

## Relationship Rules

Use a relationship when a business verb links two entities.

Examples:

- `users` has `user_profiles`
- `tenants` has `tenant_members`
- `roles` grants `permissions`

Prefer short present-tense labels.
Avoid noun-like labels when the relation should instead be modeled as a real entity.

## Many-to-Many Rule

Default rule:

- If the relation is only `many relates to many`, keep it as a direct many-to-many relationship in the ERD.
- Do not introduce an intermediate entity only because the database will eventually need a join table.

Create an intermediate entity only when the relation is itself a business concept.

Good reasons to create an intermediate entity:

- the relation has its own status
- the relation has dates such as assigned date or revoked date
- the relation stores actor or audit information
- the relation is referenced elsewhere as its own record

## Symbolic Cardinality Heuristics

Use these mappings:

- exactly one -> `ERone`
- zero or one -> `ERzeroToOne`
- zero or many -> `ERzeroToMany`
- one or many -> `ERoneToMany`

Heuristics:

- `can`, `may`, `optional`, `zero or more` -> optional side
- `must`, `always`, `required`, `exactly one` -> required side
- if the lower bound is not safe to infer, prefer the optional form

## Grouping

Default to per-context diagrams when the domain spans multiple bounded contexts.
When the output is split by context, also create one high-level overview by default.

Recommended output strategy:

- one file per bounded context
- one mandatory overview file that shows the shared backbone and major cross-context relations
- keep cross-context references as light stubs instead of copying the whole model into every file

## Layout Cleanup Rules

After the first pass, do a dedicated routing cleanup pass.

Use these rules:

- reduce crossings by moving entities first
- prefer straight horizontal lines
- if horizontal is not possible, prefer straight vertical lines
- if straight routing is not possible, use one clean 90-degree elbow
- avoid diagonal or zig-zag shapes
- leave at least `120-180 px` between an entity edge and a diamond edge
- on rhombus shapes, use only vertex anchor points: top, right, bottom, left
- give each busy entity side multiple anchor lanes such as `0.2`, `0.5`, and `0.8`
- when two relations leave the same entity side, keep their lanes distinct
- if two relations would overlap, move the second relation into an empty corridor instead of sharing a trunk
- keep diamond labels readable by reserving whitespace around each diamond
- if one lane is busy and an adjacent lane is empty, move the relation into the empty lane
- if the domain becomes too dense, split the ERD into multiple bounded-context diagrams

## Quality Checklist

Before finalizing the `.drawio` file or file set, confirm:

- each rectangle is a real persisted business entity
- each diamond is a meaningful verb phrase
- source precedence was respected when docs disagreed
- direct many-to-many relationships are not over-modeled as fake entities
- the chosen markers match the source docs
- every edge touches the diamond at top, right, bottom, or left vertex only
- connectors are routed cleanly without overlapping other relations where a simple lane or waypoint would avoid it
- grouping reflects the bounded contexts in the analysis docs
- an overview file exists whenever the output is split by context
- glossary alignment from specs was respected
- the chosen file split keeps each context readable without forcing an unnecessary all-in-one canvas
