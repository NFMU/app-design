# RBAC Enforcement Requirements

## Business Outcome

The platform must authorize actions consistently across platform, tenant or workspace, channel, and self scopes.

## Functional Requirements

- The system must support platform-scope management for `Super Admin`.
- The system must support tenant or workspace-scope management for tenant and workspace administrators.
- The system must support channel-scope management for channel administrators.
- The system must support self-scope actions for normal members.
- Permission checks must be enforceable for API calls, UI actions, and socket or event interactions.
- The platform must support seeded system roles for baseline operation.
- The platform must support assigning and revoking roles at collaboration-boundary level and at channel level.
- The platform may support custom roles in addition to seeded roles.

## Business Rules

- Roles bundle permissions; users do not receive raw permissions directly.
- Permission evaluation is contextual: the same user can have different rights at tenant and channel scope.
- Self-scope permissions should remain valid even when broader administrative scopes are absent.
- Role assignments are auditable and revocable.

## Persisted Concepts Expected By Downstream Design

- role
- permission
- role-to-permission relation
- collaboration-boundary role assignment
- channel role assignment
