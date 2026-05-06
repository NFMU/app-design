# RBAC Enforcement Requirements

## Business Outcome

The platform must authorize actions consistently across platform, tenant or workspace, channel, and self scopes.

## Functional Requirements

- The system must support platform-scope management for `Super Admin`.
- The system must support tenant or workspace-scope management for tenant and workspace administrators.
- The system must support tenant billing administration for members assigned the `Billing Admin` role.
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
- Billing permissions are tenant-scoped and must be separate from general tenant administration so that invoice, payment method, provider reference, and billing override data are only visible to authorized roles.
- `Super Admin` may manage platform billing configuration and approve tenant billing exceptions.
- `Tenant Admin` may manage billing only when the tenant's seeded or custom role grants the relevant billing permissions.
- `Billing Admin` must be a seeded tenant-scoped role for subscription, invoice, billing profile, and payment method administration.
- Role assignments are auditable and revocable.

## Baseline Permission Families

- `platform.billing.prices.manage`: manage commercial plan prices and billing policies.
- `tenant.billing.profile.manage`: create or update tenant billing profile and invoice-delivery data.
- `tenant.billing.subscription.manage`: start, change, cancel, or recover tenant subscription lifecycle.
- `tenant.billing.payment_methods.manage`: start provider setup flows and update display-safe payment method references.
- `tenant.billing.invoices.read`: view tenant invoices, receipts, payment history, and billing contact details.
- `tenant.billing.overrides.apply`: apply, expire, or revoke approved manual billing overrides.
- `tenant.entitlements.read`: view current tenant entitlement source, plan, subscription status, and billing access state.

## Persisted Concepts Expected By Downstream Design

- role
- permission
- role-to-permission relation
- collaboration-boundary role assignment
- channel role assignment
