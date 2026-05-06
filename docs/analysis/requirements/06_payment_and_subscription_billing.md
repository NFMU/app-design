# Payment And Subscription Billing Requirements

## Business Outcome

The platform must support monetizing tenant accounts through subscription plans, payment collection, invoices or receipts, and billing visibility while protecting collaboration data during billing lifecycle changes.

## Functional Requirements

- A `Super Admin` can configure active and inactive commercial plans and pricing options.
- A `Tenant Admin` or `Billing Admin` can start a tenant subscription by selecting an active plan and billing interval.
- A `Tenant Admin` or `Billing Admin` can maintain billing contact, business, tax, and invoice-delivery information.
- A `Tenant Admin` or `Billing Admin` can add, replace, or remove a default payment method through an external payment provider flow.
- The system stores payment method references and display-safe metadata only.
- The system can support trial, active, past-due, canceled, unpaid, and incomplete subscription states.
- The system can schedule cancellation at the end of the current paid period or cancel immediately when policy permits it.
- The system can change a subscription plan or billing interval while preserving the previous plan history and effective dates.
- The system can generate or receive invoices for recurring subscription periods and one-time billing adjustments.
- The system can record payment attempts, payment successes, payment failures, refunds, and credits.
- The system can notify billing contacts about trial ending, invoice issued, payment succeeded, payment failed, refund processed, subscription changed, and subscription canceled events.
- A `Tenant Admin`, `Billing Admin`, or `Super Admin` can view current subscription status, invoices, payment history, and billing contact details.
- A `Super Admin` can apply a billing exception or manual billing override for approved tenants.
- A `Super Admin` can configure billing policies that define grace period, dunning attempts, and the billing-driven access restriction mode.
- The system can apply grace-period and dunning policies before restricting or suspending a tenant for non-payment.

## Subscription Lifecycle

- A subscription can be trialing, active, past due, unpaid, canceled, incomplete, or expired.
- A subscription can renew automatically when the selected plan and billing policy allow recurring billing.
- A subscription can move from trialing to active only after billing requirements are satisfied or an approved exception exists.
- A subscription can become past due when a renewal invoice or payment attempt fails.
- A subscription can become unpaid after grace-period and dunning policies are exhausted.
- A canceled subscription should retain historical invoices, payments, plan changes, and provider references.
- A manual billing override can allow trial conversion, paid entitlement activation, or access restoration without a provider-collected payment when approved by platform policy.
- A manual billing override can expire or be revoked without deleting the tenant, subscription, invoices, or payment history.

## Business Rules

- A billable tenant must have one billing profile before paid subscription billing starts.
- A tenant may have at most one active subscription in Phase 1.
- The active subscription determines the tenant's commercial plan and billable entitlement period.
- Tenant feature limits should be derived from the active plan, while billing state should be derived from the subscription.
- Plan changes must keep an auditable history of previous plan, new plan, actor, reason, and effective date.
- Invoices and receipts are immutable once issued except for status changes, voiding, credits, or linked refunds.
- Failed payment handling must not delete tenant data, memberships, channels, messages, or audit history.
- Access restrictions caused by billing status must be reversible after payment recovery or administrator override.
- Members without billing permissions must not see payment method details, invoice delivery data, or provider references.
- Billing overrides must be auditable with approver, reason, effective dates, status, and revocation details.
- Dunning and grace-period rules must be deterministic for a subscription so the system can explain why access is active, restricted, suspended, or restored.
- External provider event handling must be idempotent and resilient to retries, duplicate callbacks, and delayed callbacks.
- Raw card, bank account, or sensitive payment credential data must remain outside the platform data store.

## Persisted Concepts Expected By Downstream Design

- billing profile
- plan price
- subscription
- payment method reference
- invoice
- payment transaction
- billing adjustment
- billing override
- billing policy
- billing provider event
