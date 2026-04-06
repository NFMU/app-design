# Claude Agent Instructions – Usecase Project

## Project Overview

This repository contains **PlantUML diagrams** for system design documentation organized by phase.

## Folder Structure

```
usecase/
└── phase1/
    ├── database/          # Schema definitions (entity fields + types, minimal relationships)
    ├── erd/               # Full ERD diagrams (entities + all cross-domain FK relationships)
    ├── usecase/           # Use case diagrams
    └── design/
        ├── event_storming/  # Event Storming diagrams (per bounded context)
        └── aggregate/       # Aggregate Design diagrams (DDD aggregates)
```

---

## `database/` — Schema Definitions

Source of truth for table schemas. Entity field definitions with types and constraints. Minimal relationships (within-domain only).

| File | Domain |
|------|--------|
| `00_overview.puml` | High-level package overview (entity names only) |
| `01_indentity_and_personal.puml` | users, user_profiles, user_settings, user_sessions, password_resets |
| `02_tenant_foundation.puml` | plans, tenants |
| `03_membership_invitation.puml` | tenant_members, channel_members, invitations |
| `04_channel.puml` | channels, channel_members |
| `05_chat.puml` | messages, message_reads, message_pins, message_attachments |
| `06_rbac.puml` | roles, permissions, role_permissions, tenant_member_roles, channel_member_roles |
| `07_full.puml` | All entities + all relationships in one diagram |

---

## `erd/` — ERD Diagrams

Visual ERD per domain. Each file is standalone: includes stub entities from other domains to show all FK relationships clearly.

| File | Domain |
|------|--------|
| `00_overview_erd.puml` | Full overview — all entities + all relationships |
| `01_identity_erd.puml` | Identity & User Management |
| `02_tenant_erd.puml` | Tenant & Subscription |
| `03_membership_erd.puml` | Membership & Invitation |
| `04_channel_erd.puml` | Channel Management |
| `05_chat_erd.puml` | Chat & Messaging |
| `06_rbac_erd.puml` | Role-Based Access Control |
| `08_workspace_erd.puml` | Workspace domain conceptual ERD (image reference) |

**ERD style rules:**
- `hide circle` + `skinparam linetype ortho` at the top
- `@startuml <name>` with a descriptive name
- Primary keys: `* id : bigint <<PK>>`
- Foreign keys: `<<FK>>`, unique: `<<UQ>>`
- Separator `--` between PK section and other fields
- Enum/status values via `note right of entity::field ... end note`
- Crow's Foot notation: `||--o{`, `||--||`, `}o--o{`
- Relationship labels in quotes: `entity1 ||--o{ entity2 : "label"`

---

## `usecase/` — Use Case Diagrams

| File | Domain |
|------|--------|
| `00_context.puml` | System context / actor overview |
| `01_identity.puml` | Authentication, registration, sessions |
| `02_tenant.puml` | Workspace management |
| `03_channel.puml` | Channel management |
| `04_core_chat.puml` | Messaging features |
| `05_personal_setting.puml` | User settings and preferences |
| `06_rbac_enforcement.puml` | Role-based access control |

---

## `design/event_storming/` — Event Storming

Per-bounded-context event storming diagrams following DDD conventions.

| File | Bounded Context |
|------|-----------------|
| `01_identity_es.puml` | Identity (register, login, password reset) |
| `02_workspace_es.puml` | Workspace (create, update, suspend, delete) |
| `03_membership_es.puml` | Membership & Invitation (invite, accept, leave, remove) |
| `04_channel_es.puml` | Channel (create, join, archive, remove member) |
| `05_messaging_es.puml` | Messaging (send, edit, delete, pin, read, attach) |
| `06_rbac_es.puml` | RBAC (seed roles, assign/revoke, check permission) |

**Color legend (sticky note convention):**

| Color | Element | PlantUML stereotype |
|-------|---------|---------------------|
| Blue `#1565C0` | Command | `<<Command>>` |
| Orange `#E65100` | Domain Event | `<<Event>>` |
| Yellow `#F9A825` | Aggregate | `<<Aggregate>>` |
| Purple `#6A1B9A` | Policy / Reaction | `<<Policy>>` |
| Green `#1B5E20` | Read Model | `<<ReadModel>>` |
| Dark Pink `#880E4F` | External System | `<<External>>` |

**Flow pattern:** `Actor → Command → Aggregate → Domain Event → (Policy) → Command → ...`

**Event Storming style rules:**
- `skinparam rectangle { RoundCorner 8 }` for rounded sticky notes
- Use `rectangle "Label" <<Stereotype>> as alias`
- Policy arrows use `[#6A1B9A]->` (purple) with `: triggers` label
- All skinparam color blocks must be included in every file (copy from existing)

---

## `design/aggregate/` — Aggregate Design

DDD aggregate boundary diagrams. Shows aggregate roots, entities, value objects, commands, and emitted domain events.

| File | Aggregate |
|------|-----------|
| `00_overview_agg.puml` | All aggregates + cross-aggregate dependencies |
| `01_user_agg.puml` | User Aggregate (users, profiles, sessions, resets) |
| `02_workspace_agg.puml` | Workspace Aggregate (tenants, plans) |
| `03_membership_agg.puml` | Membership Aggregate (tenant_members, invitations) |
| `04_channel_agg.puml` | Channel Aggregate (channels, channel_members) |
| `05_message_agg.puml` | Message Aggregate (messages, reads, pins, attachments) |
| `06_rbac_agg.puml` | RBAC Aggregate (roles, permissions, assignments) |

**Aggregate design conventions:**

| Stereotype | Meaning | Color |
|------------|---------|-------|
| `<<AggregateRoot>>` | Root entity of the aggregate | Yellow `#F9A825` |
| `<<Entity>>` | Entity within the aggregate boundary | Blue `#E3F2FD` |
| `<<ValueObject>>` | Immutable value object | Purple `#F3E5F5` |
| `<<DomainEvent>>` | Event emitted by the aggregate | Orange `#E65100` |

**Aggregate design rules:**
- Use `class` (not `entity`) for aggregates — enables method signatures
- Methods in aggregate root show `commandName() : EventEmitted` signature
- Composition (`*--`) for entities owned by the aggregate
- Dashed dependency (`..>`) for cross-aggregate ID references
- Never use direct object references between aggregates — IDs only
- Domain Events block listed separately from the aggregate package
- Add `note` for non-obvious policies triggered by key events

---

## Domain Model Summary

### Aggregates and Boundaries

```
User Aggregate
  Root: users
  Entities: user_profiles, user_settings, user_sessions, password_resets

Workspace Aggregate
  Root: tenants
  Value Objects: plans

Membership Aggregate
  Root: tenant_members
  Entities: invitations

Channel Aggregate
  Root: channels
  Entities: channel_members

Message Aggregate
  Root: messages
  Entities: message_reads, message_pins, message_attachments

RBAC Aggregate
  Root: roles
  Entities: role_permissions, tenant_member_roles, channel_member_roles
  Value Objects: permissions
```

### Key Design Decisions

- **Tenant = Workspace**: DB uses `tenants`; business domain says "workspace". No separate workspace table.
- **Single membership model**: `tenant_members` for workspace membership; `channel_members` for channel membership.
- **Two-level RBAC**: Roles scoped at `tenant` or `channel` level via `scope_type`.
- **Invitations unified**: `invitations` covers both workspace (`invite_type = 'tenant'`) and channel invites.
- **Cross-aggregate via IDs**: Aggregates reference each other by ID only — never direct object references.
- **Soft deletes**: `deleted_at` on users, tenants, channels, messages.

### Terminology Mapping (Business → DB)

| Business Term | DB Table |
|---------------|----------|
| workspace | tenants |
| membership | tenant_members |
| workspace invite | invitations |

---

## Agent Instructions

When working on this project:

1. **New entity in existing domain**: Add to the relevant `database/` file → update `07_full.puml` + `00_overview.puml` → update corresponding `erd/` file → update `aggregate/` file.
2. **New domain event**: Add to the corresponding `design/event_storming/` file → add emitted event to the `design/aggregate/` file.
3. **New bounded context / subdomain**: Create matching files in all four folders (`database/`, `erd/`, `design/event_storming/`, `design/aggregate/`) with the same numbering.
4. **New phase**: Create `phase2/` with the same four-folder structure.
5. **Naming**: snake_case for all table/field names.
6. **Notes**: Always add `note right of entity::field` for enum/status fields in ERD files.
7. **Aggregate rule**: Never add direct object references between aggregates — use IDs only.

## How to Render Diagrams

- VS Code: PlantUML extension (preview with `Alt+D`)
- Online: https://www.plantuml.com/plantuml/uml/
- CLI: `plantuml filename.puml` or `plantuml -r **/*.puml`
