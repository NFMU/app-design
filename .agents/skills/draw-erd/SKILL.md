---
name: draw-erd
description: Infer a conceptual ERD in editable draw.io XML from `docs/analysis/*`, supporting DDD artifacts, and existing diagrams. Use when Codex must identify business entities, verb relationships, symbolic cardinalities, and readable orthogonal routing, then write one or more context-first diagrams to `docs/technical/erd/*.drawio`.
---

# Draw ERD

Create a conceptual ERD as an editable `.drawio` file directly from project documents.
Read [references/erd-derivation-guide.md](references/erd-derivation-guide.md) first.
Read [references/drawio-xml-patterns.md](references/drawio-xml-patterns.md) before writing XML.

## Output Contract

- Write the final result to one or more files in `docs/technical/erd`.
- Keep the XML editable and uncompressed.
- Draw one rectangle per entity and one diamond per relationship.
- Use symbolic markers on every relationship segment.
- Render entity names only. Do not render attributes, PK/FK metadata, or physical table details in the ERD.
- Prefer plural `snake_case` entity labels in this repository so the ERD maps cleanly into `docs/technical/rmd`.
- Use short verb phrases for relationship labels, for example `has`, `belongs to`, `requests`, `assigns`, `tracks`.
- Default strategy: `context-first + overview`.
- If the model spans multiple bounded contexts, emit multiple files such as `01_identity_erd.drawio`, `02_tenant_erd.drawio`, `03_membership_erd.drawio`.
- When the model spans multiple bounded contexts, also emit one high-level overview file such as `00_overview_erd.drawio`.
- The overview is mandatory by default when the output is split by context.

The notation is:

- entity: rectangle
- relationship: diamond
- entity-side connector: default single bar
- relationship-side connector: `ERone`, `ERzeroToOne`, `ERzeroToMany`, or `ERoneToMany`

## Workflow

### 1. Gather evidence from docs

Inspect project docs in this order unless the user says otherwise:

1. `docs/analysis/requirements`
2. `docs/analysis/specs`
3. `docs/analysis/usecase`
4. `docs/strategic/event_storming`
5. `docs/tatical/aggregate`
6. existing `docs/technical/erd/*.drawio`

Use this source precedence when the artifacts disagree:

1. analysis requirements
2. analysis specs
3. analysis use cases
4. strategic event storming
5. tactical aggregate design
6. existing technical ERD diagrams

Treat existing diagrams as supporting evidence, not the only source of truth.
If a spec file defines canonical vocabulary or conceptual-to-technical mapping, follow that mapping even when older diagrams use different terms.
If the docs still conflict after comparison, prefer the less committal cardinality and keep the model conservative.

### 2. Derive entities and relationships

Convert the docs into a business data model:

- keep persisted business nouns as entities
- convert business verbs into relationships
- keep only entity names in the rendered ERD
- reason about attributes internally if needed, but do not draw attribute lists in the final diagram
- preserve repository terminology when the specs intentionally collapse conceptual aliases

Important modeling rule:

- If a many-to-many relationship is only a relationship and has no separate business meaning, do not create an intermediate entity.
- Only create a bridge or junction entity when the relation itself is a real business record with its own lifecycle, audit meaning, state, or required fields.
- Do not introduce technical entities that only exist for database implementation.
- Do not collapse a real business record into a plain diamond if the relation has its own status, dates, ownership, or audit meaning.

### 2.5. Partition the output by context before drawing

Before writing XML, decide whether the model should be split:

- use one file per bounded context when entities mainly interact inside that context
- keep cross-context stub entities minimal when a context diagram needs to show an external dependency
- reuse the repo's context naming when possible, for example `identity`, `tenant`, `membership`, `channel`, `chat`, `rbac`
- avoid forcing unrelated contexts into one crowded canvas
- always pair the context split with one overview file that helps the reader see the whole bounded-context landscape
- treat the overview as a companion file, not a replacement for the context diagrams

### 3. Choose symbolic cardinality carefully

Use draw.io ER markers:

- `ERone` = exactly one
- `ERzeroToOne` = zero or one
- `ERzeroToMany` = zero or many
- `ERoneToMany` = one or many

If the docs say `can`, `may`, `optional`, or the record might not exist yet, prefer optional forms.
If the docs say `must`, `always`, `exactly one`, or `required`, prefer required forms.
If the docs justify a lower bound, prefer `0..N` or `1..N` over a vague many-side assumption.

### 4. Author the draw.io file directly

Write the `.drawio` XML directly into one or more files under `docs/technical/erd`.

Use [references/drawio-xml-patterns.md](references/drawio-xml-patterns.md) as the source for:

- the minimal `mxfile` skeleton
- entity and relationship styles used by this repository
- marker-based connector styles for one, zero-or-one, many, one-or-many
- waypoint and lane patterns that prevent overlapping lines

When writing the diagram:

- create one rectangle per entity
- create one diamond per relationship
- connect entity -> relationship and relationship -> entity with the correct markers at both ends of each segment
- attach every relationship segment to one of the four diamond vertices only: top, right, bottom, or left
- place shapes with explicit `x`, `y`, `width`, and `height`
- leave breathing room around every diamond; do not place a diamond flush against an entity edge
- keep names short and domain-focused
- if the domain is large or spans multiple bounded contexts, split it into multiple ERD files instead of forcing one crowded diagram
- when you split by context, also write one overview ERD that shows the core entities and the most important cross-context relations
- keep filenames aligned with the context, for example `01_identity_erd.drawio` or `05_chat_erd.drawio`

### 5. Beautify the routing after the first draft

After the diagram is logically correct, do one cleanup pass for readability. Follow the full layout cleanup rules in [references/erd-derivation-guide.md](references/erd-derivation-guide.md). Key principles:

- minimize line crossing by repositioning entities first before adding bends
- prioritize straight horizontal lines, then vertical, then one 90-degree elbow
- avoid diagonal, curved, and zig-zag connectors
- assign different anchor lanes when multiple relations leave the same entity side
- keep connectors in empty corridors and never route them through another entity box
- if a diagram becomes crowded, split it into multiple files instead of compressing the layout

### 6. Review before finishing

Follow the full quality checklist in [references/erd-derivation-guide.md](references/erd-derivation-guide.md). Key checks:

- every rectangle is a true business entity, every diamond is a verb phrase
- source precedence was respected where docs conflicted
- many-to-many relations stay direct unless a real business entity is required
- every connector touches the diamond at a vertex, not halfway along an edge
- layout uses straight lines or single elbows, no overlapping routes
- each diamond has visible whitespace around it
- terminology is aligned with any canonical glossary in `docs/analysis/specs`
- the file opens cleanly in draw.io

## Output Rules

- Default output folder: `docs/technical/erd`
- Write the final result directly as `.drawio`
- Prefer multiple context files by default when the domain is not trivially small
- When context files are emitted, also emit one overview file by default
- Keep the ERD conceptual even though the file lives under `technical`

## Resources

### references/

- `references/erd-derivation-guide.md`: heuristics for deriving entities, relations, optionality, and context splits
- `references/drawio-xml-patterns.md`: draw.io XML skeletons, diamond patterns, and routing rules used by this repository
