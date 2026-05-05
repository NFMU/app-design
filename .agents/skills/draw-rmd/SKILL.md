---
name: draw-rmd
description: Infer a physical relational model in editable draw.io XML from `docs/analysis/*`, supporting DDD artifacts, and existing diagrams. Use when Codex must map business concepts into physical tables, columns, keys, nullability, and readable orthogonal routing, then write one or more context-first diagrams to `docs/technical/rmd/*.drawio`.
---

# Draw RMD

Create a physical relational model diagram as an editable `.drawio` file directly from project documents.
Read [references/rmd-derivation-guide.md](references/rmd-derivation-guide.md) first.
Read [references/drawio-xml-patterns.md](references/drawio-xml-patterns.md) before writing XML.
Read [../_shared/drawio-routing-algorithm.md](../_shared/drawio-routing-algorithm.md) before placing connectors.

## Output Contract

- Write the final result to one or more files in `docs/technical/rmd`.
- Keep the XML editable and uncompressed.
- Draw one table box per physical table and one direct edge per parent-child relationship.
- Use plural `snake_case` table names by default, for example `users`, `tenant_members`, `message_attachments`.
- Use the 4-column table mode by default: `Key`, `Property Name`, `Type`, `Nullable`.
- Keep the schema conservative: omit speculative columns, speculative audit fields, and engine-specific details unless the docs justify them.
- Default strategy: `context-first + overview`.
- If the model spans multiple bounded contexts, emit multiple files such as `01_indentity_and_personal.drawio`, `02_tenant_foundation.drawio`, `03_membership_invitation.drawio`.
- When the model spans multiple bounded contexts, also emit one high-level overview file such as `00_overview.drawio`.
- The overview is mandatory by default when the output is split by context.

## Workflow

### 1. Gather evidence from docs

Inspect project docs in this order unless the user says otherwise:

1. `docs/analysis/requirements`
2. `docs/analysis/specs`
3. `docs/analysis/usecase`
4. `docs/strategic/event_storming`
5. `docs/tatical/aggregate`
6. existing `docs/technical/erd/*.drawio`
7. existing `docs/technical/rmd/*.drawio`

Use this source precedence when the artifacts disagree:

1. analysis requirements
2. analysis specs
3. analysis use cases
4. strategic event storming
5. tactical aggregate design
6. existing technical ERD
7. existing technical RMD

Treat existing technical diagrams as supporting evidence, not the only source of truth.
If the specs define canonical vocabulary or a conceptual-to-technical mapping, follow that mapping even when older diagrams use different terms.
If the docs still conflict after comparison, keep the relational model conservative:

- prefer fewer columns over invented columns
- prefer nullable fields when requiredness is unclear
- prefer a surrogate `id` for normal business tables when a natural key is not clearly justified
- preserve a documented business table instead of collapsing it into a purely technical shortcut

### 2. Derive tables from the business model

Convert persisted business entities into physical tables:

- each stable persisted business entity usually becomes one table
- table names must be database-ready plural `snake_case` nouns
- use the canonical names defined by specs when conceptual labels differ from the persisted model
- a direct many-to-many relation in the ERD usually becomes a technical junction table in the RMD
- if the docs already show the relation as a real business record, keep that table as a normal business table and do not add another technical junction

Important modeling rules:

- do not introduce technical tables that the docs or specs do not justify
- do not flatten a real business record into a pure junction table
- do not let an old diagram alone force redundant persistence tables if higher-priority analysis docs do not support them

### 2.5. Partition the output by context before drawing

Before writing XML, decide whether the schema should be split:

- use one file per bounded context when tables mainly interact inside that context
- keep external references lightweight when a context diagram needs upstream or downstream dependencies
- reuse repository naming when possible, for example `identity_and_personal`, `tenant_foundation`, `membership_invitation`, `channel`, `chat`, `rbac`
- avoid one giant schema canvas unless the user asked for a full schema view
- always pair the context split with one overview schema that shows the core tables and major cross-context foreign-key paths
- treat the overview as a companion file, not a replacement for the detailed context schemas

### 3. Derive columns and keys conservatively

Use this column policy:

- default mode: four columns -> `Key`, `Property Name`, `Type`, `Nullable`
- show `PK`, `FK`, `UQ`, `PK FK`, or `UQ FK` in the Key column
- use engine-neutral types such as `integer`, `string`, `date`, `boolean`, `float`, `text`, `timestamp`
- use only `yes` or `no` in the Nullable column

Default constraint rules:

- each normal business table gets `PK id` unless the docs clearly require another key strategy
- place foreign keys on the dependent side using `<parent>_id`
- use `UQ FK` when the docs imply a one-to-one relation
- for a pure junction table, prefer a composite primary key from the two foreign keys unless the docs or conventions require a surrogate key
- include business columns only when the docs justify them
- do not add audit or lifecycle columns unless the docs or existing conventions explicitly justify them

### 4. Map relationships into database implementation

Use direct table-to-table lines in the final RMD:

- `1 : 0..N` -> foreign key on the many-side table
- `1 : 0..1` -> unique foreign key on the optional dependent table
- `1 : 1` -> unique foreign key on the dependent table unless shared identity is explicitly required
- direct `N : N` in the ERD -> create a junction table in the RMD unless the relation is already modeled as its own business table

### 5. Author the draw.io file directly

Write the `.drawio` XML directly into one or more files under `docs/technical/rmd`.

Use [references/drawio-xml-patterns.md](references/drawio-xml-patterns.md) as the source for:

- the minimal `mxfile` skeleton
- HTML table box patterns used by this repository
- draw.io ER marker names and orthogonal edge styles
- routing rules that prevent overlapping lines
- direct orthogonal edge styles with curved routing disabled

Use [../_shared/drawio-routing-algorithm.md](../_shared/drawio-routing-algorithm.md) as the routing algorithm for every connector:

- place tables into rows or columns before adding edges
- choose the facing side and anchor lane first, then try straight, one-elbow, and two-elbow candidates in that order
- treat every table, note, label, and group boundary as an inflated obstacle that connectors must avoid
- write edges with `edgeStyle=orthogonalEdgeStyle;rounded=0;curved=0`
- omit waypoints for straight lines, use one waypoint for one elbow, and use two waypoints only for a necessary dogleg
- move tables or split the file before accepting a connector with more than two bends

When writing the diagram:

- create one table box per physical table
- use HTML `<table>` with four columns by default
- center the table-name header text in every table
- size each table box from its row count so no row is clipped
- route connectors orthogonally
- if the model feels crowded, split it into multiple RMD files instead of compressing the layout
- when you split by context, also write one overview RMD that shows the key tables and major inter-context relationships
- keep filenames aligned with the context, for example `04_channel.drawio` or `06_rbac.drawio`

For `00_overview.drawio`, use the shared routing algorithm's overview topology mode:

- show only the schema backbone and major foreign-key paths, not every detailed FK from the context files
- place each bounded context as a local cluster with nearby tables
- avoid long connectors from common tables such as `users` or `tenants` across the whole canvas
- use lightweight external-reference stubs or omit detail-level cross-context FKs when the numbered RMD owns the detail
- no relationship line should pass behind another context cluster

### 6. Beautify routing and verify table avoidance

After the diagram is logically correct, do one cleanup pass for readability. Follow the full layout cleanup rules in [references/rmd-derivation-guide.md](references/rmd-derivation-guide.md). Key principles:

- minimize line crossing by repositioning tables first before adding bends
- prioritize straight horizontal lines, then vertical, then one 90-degree elbow
- avoid diagonal, curved, and zig-zag connectors
- assign different anchor lanes when multiple edges leave the same table side
- keep connectors in empty corridors and never route them through another table box
- run obstacle-intersection and colinear-overlap checks from the shared routing algorithm

### 7. Review before finishing

Follow the full quality checklist in [references/rmd-derivation-guide.md](references/rmd-derivation-guide.md). Key checks:

- source precedence was respected where docs conflicted
- every important ERD relationship was mapped into a physical implementation
- no fake technical table was added where a documented business table already exists
- foreign keys sit on the correct dependent tables
- one-to-one cases use unique foreign keys when appropriate
- table names are valid plural `snake_case`
- terminology is aligned with the canonical glossary in `docs/analysis/specs`
- the file opens cleanly in draw.io

## Output Rules

- Default output folder: `docs/technical/rmd`
- Write the final result directly as `.drawio`
- Prefer multiple context files by default when the domain is not trivially small
- When context files are emitted, also emit one overview file by default
- Keep the schema conservative and analysis-driven

## Resources

### references/

- `references/rmd-derivation-guide.md`: heuristics for deriving physical tables, columns, keys, nullability, and context splits
- `references/drawio-xml-patterns.md`: draw.io XML patterns, table templates, and routing rules used by this repository
- `../_shared/drawio-routing-algorithm.md`: direct orthogonal routing algorithm and validation checks shared by draw.io skills
