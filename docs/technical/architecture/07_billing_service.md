# Billing Service

## Status

**Planned.** No source code exists yet under `services/`. Spec authority remains `docs/analysis/specs/06_payment_and_subscription_billing.md`.

## Responsibility

The billing service is the authority for **monetization lifecycle**: which prices exist, which tenants are subscribed, what invoices have been issued, what payments succeeded or failed, and what billing exceptions are in effect.

It does **not** define the entitlement catalog (`plans` lives in `tenants`). It does **not** decide what a subscription unlocks — that is determined by the referenced `plan_id`, which `tenants` exposes as `tenants.plan_id`.

It also does **not** store raw payment credentials. Payment instrument storage is delegated to a third-party provider (Stripe / PayOS / etc.); only **display-safe references** (brand, last4, expiry, provider IDs) live here.

## Owned Tables

| Table | PK type | Purpose |
|---|---|---|
| `plan_prices` | `integer` | Currency / interval / amount options for a `plan`. One plan can have many prices. |
| `billing_profiles` | `integer` | Billing contact, tax ID, address per tenant. |
| `subscriptions` | `integer` | Active / past subscription period. References plan + plan price + billing profile. |
| `payment_method_refs` | `integer` | Display-safe references to provider-managed payment methods. |
| `invoices` | `integer` | Issued billing documents. Immutable after issuance; corrections via adjustments. |
| `payment_transactions` | `integer` | Payment attempt outcomes (collect / fail / refund / credit). |
| `billing_adjustments` | `integer` | Approved credits, refunds, write-offs. |
| `billing_overrides` | `integer` | Approved exceptions that satisfy access policy without payment. |
| `billing_policies` | `integer` | Grace period, dunning attempt limit, restriction mode rules. |
| `billing_provider_events` | `integer` | Idempotent log of provider webhook callbacks. |

Full column definitions: `docs/analysis/specs/06_payment_and_subscription_billing.md`.

## Module Layout (Target)

```
services/billing/src/
├── app.module.ts
├── main.ts
├── core/
├── infrastructure/
│   ├── clients/
│   │   ├── identity.client.ts
│   │   ├── tenants.client.ts
│   │   ├── rbac.client.ts
│   │   └── provider/
│   │       ├── stripe.client.ts     # one client per supported provider
│   │       └── payos.client.ts
│   ├── database/database.module.ts
│   ├── orms/                        # one file per table
│   └── webhooks/
│       └── provider-webhook.handler.ts
└── modules/
    ├── plan-prices/                 # CRUD (admin)
    ├── billing-profiles/
    ├── subscriptions/
    ├── payment-methods/
    ├── invoices/
    ├── payments/                    # transaction history
    ├── adjustments/
    ├── overrides/
    ├── policies/                    # admin-managed
    └── entitlements/                # entitlement summary endpoint
```

## Planned Public API Surface

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/plan-prices` | Bearer | List active prices. Filter by plan / currency / interval. |
| GET | `/plans/{planId}/prices` | Bearer | List prices for one plan. |
| POST | `/plan-prices` | Bearer + `platform.billing.prices.manage` | Create a price. |
| PATCH | `/plan-prices/{id}` | Bearer + `platform.billing.prices.manage` | Update or deprecate. |
| GET | `/tenants/{tenantUuid}/billing/profile` | Bearer + `tenant.billing.profile.manage` | Get billing profile. |
| PUT | `/tenants/{tenantUuid}/billing/profile` | Bearer + `tenant.billing.profile.manage` | Create or update billing profile. |
| POST | `/tenants/{tenantUuid}/billing/subscription` | Bearer + `tenant.billing.subscription.manage` | Start or change subscription. |
| GET | `/tenants/{tenantUuid}/billing/subscription` | Bearer + `tenant.billing.subscription.manage` | Get current subscription. |
| POST | `/tenants/{tenantUuid}/billing/subscription/cancel` | Bearer + `tenant.billing.subscription.manage` | Cancel (immediate or at period end). |
| GET | `/tenants/{tenantUuid}/billing/payment-methods` | Bearer + `tenant.billing.payment_methods.manage` | List payment method references. |
| POST | `/tenants/{tenantUuid}/billing/payment-methods` | Bearer + `tenant.billing.payment_methods.manage` | Attach a provider-tokenized payment method. |
| DELETE | `/payment-methods/{id}` | Bearer + `tenant.billing.payment_methods.manage` | Detach. |
| GET | `/tenants/{tenantUuid}/billing/invoices` | Bearer + `tenant.billing.invoices.read` | List invoices. |
| GET | `/invoices/{id}` | Bearer + `tenant.billing.invoices.read` | Invoice detail. |
| GET | `/invoices/{id}/pdf` | Bearer + `tenant.billing.invoices.read` | Download invoice PDF. |
| GET | `/tenants/{tenantUuid}/billing/transactions` | Bearer + `tenant.billing.invoices.read` | Payment transaction history. |
| POST | `/tenants/{tenantUuid}/billing/overrides` | Bearer + `tenant.billing.overrides.apply` | Apply an approved override. |
| DELETE | `/billing-overrides/{id}` | Bearer + `tenant.billing.overrides.apply` | Revoke an override. |
| GET | `/tenants/{tenantUuid}/entitlement-summary` | Bearer + `tenant.entitlements.read` | Composite read model: plan, subscription status, override, billing access state. |

Full request/response shapes: `docs/technical/api/07_billing_api.md`.

## Webhook Surface

Single endpoint per provider, public but signature-verified:

| Method | Path | Provider verification |
|---|---|---|
| POST | `/webhooks/stripe` | Stripe signature header (`Stripe-Signature`). |
| POST | `/webhooks/payos` | PayOS HMAC. |

Each callback is recorded into `billing_provider_events` keyed by `(provider, provider_event_id)` for idempotent processing. Reprocessing is safe.

## Internal API Surface

| Method | Path | Purpose |
|---|---|---|
| GET | `/internal/tenants/{uuid}/entitlement-summary` | Used by `tenants` to compose the tenant detail response. |
| GET | `/internal/tenants/{uuid}/access-state` | Returns `{ allowed: bool, reason: 'paid' \| 'grace' \| 'override' \| 'restricted' \| 'suspended' }`. Used by all services that gate user actions on billing state. |

## Inter-Service Dependencies

**Outbound:**

| Target | Endpoint(s) | When |
|---|---|---|
| `identity` | `GET /internal/users/{id}` | Validate `created_by`, `approved_by`, `revoked_by`. |
| `tenants` | `GET /internal/tenants/{id}` | Resolve plan, slug, name for invoice generation. |
| `tenants` | `POST /internal/tenants/{id}/sync-plan` | Notify tenants of plan change after subscription update. |
| `rbac` | Permission checks for all admin endpoints. |

**Inbound:**

| Caller | Endpoint(s) | When |
|---|---|---|
| `tenants` | `GET /internal/tenants/{uuid}/entitlement-summary` | Hydrate tenant detail response. |
| `channels`, `messaging`, `tenants` | `GET /internal/tenants/{uuid}/access-state` | Per-request access gate. Returns `402 PAYMENT_REQUIRED` upstream if billing state blocks access. |

## Cross-Service Reference Columns

| Column | Owning service / table | Validation strategy |
|---|---|---|
| `plan_prices.plan_id` | `tenants.plans` (integer) | Validate via `/internal/plans/{id}` on create. |
| `billing_profiles.tenant_id` | `tenants.tenants` (integer) | Validate-on-write. |
| `subscriptions.tenant_id` | `tenants.tenants` (integer) | Validate-on-write. |
| `subscriptions.plan_id` | `tenants.plans` (integer) | Validate-on-write. |
| `payment_method_refs.tenant_id` | `tenants.tenants` (integer) | Validate-on-write. |
| `invoices.tenant_id` | `tenants.tenants` (integer) | Validate-on-write. |
| `billing_adjustments.created_by` | `identity.users` (uuid) | Trusted. |
| `billing_overrides.approved_by`, `revoked_by` | `identity.users` (uuid) | Trusted. |

## Access State Computation

`GET /internal/tenants/{uuid}/access-state` is the single source of truth for "is this tenant allowed to use the platform right now?". The decision tree:

```
1. Active billing_override?      → allowed (reason: 'override')
2. Subscription = 'active'?      → allowed (reason: 'paid')
3. Subscription = 'trialing'?    → allowed (reason: 'paid')
4. Subscription = 'past_due'?
   ├─ within billing_policy.grace_period_days? → allowed (reason: 'grace')
   └─ else                                      → denied (reason: 'restricted')
5. Subscription = 'unpaid' / 'canceled'?       → denied (reason: 'restricted')
6. tenants.status = 'suspended'?               → denied (reason: 'suspended')
```

The result is cacheable for ~30 seconds. Cache invalidation on subscription / override change is best-effort.

## Idempotency & Audit

Every mutation in billing must be:

- **Idempotent on retry.** Use the provider's idempotency key when calling the provider; record the result before returning.
- **Audited.** `created_by`, `approved_by`, `revoked_by` are required fields, not optional.
- **Reversible only via new rows.** Issued invoices are not edited; use `billing_adjustments` for corrections.

## Compliance Notes

- **No raw card / bank account / CVV ever stored or logged.** All sensitive flow happens client-side via the provider's tokenized SDK.
- **PCI scope is minimized to "merchant of record using a fully-tokenized provider"** (SAQ-A under PCI DSS).
- Tax ID and billing address are PII — restrict access to `tenant.billing.profile.manage` and audit reads.
