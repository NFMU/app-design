# Payment And Subscription Billing Spec

## Canonical Tables

- `plan_prices`
- `billing_profiles`
- `subscriptions`
- `payment_method_refs`
- `invoices`
- `payment_transactions`
- `billing_adjustments`
- `billing_overrides`
- `billing_policies`
- `billing_provider_events`

## Structural Rules

- `plans` (owned by the collaboration foundation) remains the product and capability catalog.
- `plan_prices` extends `plans` with currency, amount, interval, trial, and effective-date details for billable plans.
- `billing_profiles` belongs to one `tenants` record and stores billing contact, business, tax, and invoice-delivery data.
- `subscriptions` belongs to one `tenants` record and applies one plan and pricing option over a billable period.
- `payment_method_refs` stores externally managed payment method references and display-safe metadata only; raw credentials are never persisted.
- `invoices` snapshot issued billing amounts, periods, and delivery status.
- `payment_transactions` records attempts and outcomes for collecting, settling, failing, refunding, or crediting payment.
- `billing_adjustments` records approved credits, refunds, write-offs, or manual billing corrections.
- `billing_overrides` records approved tenant billing exceptions that can satisfy billing access policy without collecting payment.
- `billing_policies` defines grace period, dunning, and restriction settings evaluated during subscription recovery.
- `billing_provider_events` stores external callback identity and processing state for idempotent event handling.

---

## Table Definitions

### `plan_prices`

Billable price option for a product plan. One plan may have multiple active prices (e.g. monthly and annual).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `plan_id` | `bigint` | NOT NULL, FK → `plans.id` RESTRICT, index | The product plan this price option belongs to. |
| `billing_policy_id` | `bigint` | nullable, FK → `billing_policies.id` SET NULL | Default billing policy applied to new subscriptions using this price. Overridable at the `subscriptions` level. |
| `code` | `varchar(50)` | NOT NULL, unique | Stable machine identifier, e.g. `pro_monthly_usd`, `pro_annual_usd`. Used in subscription creation and provider sync. |
| `currency_code` | `varchar(3)` | NOT NULL | ISO 4217 currency code, e.g. `USD`, `VND`, `JPY`. |
| `amount_minor` | `bigint` | NOT NULL | Price amount in the currency's smallest unit (cents, đồng, yen). Avoids decimal rounding issues. |
| `billing_interval` | `varchar(20)` | NOT NULL | Billing period: `monthly` \| `annual` \| `one_time`. |
| `trial_days` | `integer` | NOT NULL, default `0` | Number of free trial days before the first charge. `0` means no trial. |
| `status` | `varchar(20)` | NOT NULL, default `'active'` | `active` \| `deprecated`. Deprecated prices can be retained by existing subscriptions but not selected for new ones. |
| `effective_from` | `date` | NOT NULL | The date from which this price is valid. |
| `effective_to` | `date` | nullable | The date after which this price is no longer offered. `NULL` means open-ended. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** unique on `code`, index on `plan_id`

**`billing_interval` values:** `monthly` | `annual` | `one_time`

---

### `billing_profiles`

Billing contact and business details for a tenant account.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `tenant_id` | `bigint` | NOT NULL, FK → `tenants.id` CASCADE, unique | Owning tenant. One billing profile per tenant in Phase 1. |
| `billing_email` | `varchar(255)` | NOT NULL | Email address for invoice delivery and billing communications. May differ from the owner's account email. |
| `company_name` | `varchar(255)` | nullable | Legal business name printed on invoices. |
| `tax_id` | `varchar(50)` | nullable | VAT number, tax registration number, or EIN depending on jurisdiction. |
| `billing_address_json` | `jsonb` | NOT NULL, default `'{}'` | Structured postal address: `line1`, `line2`, `city`, `state`, `postal_code`, `country_code`. |
| `invoice_delivery_preferences_json` | `jsonb` | NOT NULL, default `'{}'` | Delivery settings: `send_email`, `cc_emails[]`, `pdf_format`. |
| `status` | `varchar(20)` | NOT NULL, default `'active'` | `active` \| `suspended`. Suspended billing profiles cannot initiate new subscriptions. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** unique on `tenant_id`

**`billing_address_json` shape:**
```json
{
  "line1": "123 Main St",
  "line2": "Suite 400",
  "city": "Ho Chi Minh City",
  "state": "",
  "postal_code": "70000",
  "country_code": "VN"
}
```

---

### `subscriptions`

Billing lifecycle record applying a plan and price to a tenant over a billable period.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `tenant_id` | `bigint` | NOT NULL, FK → `tenants.id` CASCADE, index | Owning tenant. At most one active subscription per tenant in Phase 1. |
| `plan_id` | `bigint` | NOT NULL, FK → `plans.id` RESTRICT | The product plan for this subscription period. Denormalized from `plan_price_id` for efficient entitlement checks. |
| `plan_price_id` | `bigint` | NOT NULL, FK → `plan_prices.id` RESTRICT | The specific price option (currency, amount, interval) in effect. |
| `billing_profile_id` | `bigint` | NOT NULL, FK → `billing_profiles.id` RESTRICT | Billing contact for invoices generated against this subscription. |
| `billing_policy_id` | `bigint` | nullable, FK → `billing_policies.id` SET NULL | Active billing policy for grace-period and dunning evaluation. Overrides the `plan_prices.billing_policy_id` default if set. |
| `provider_subscription_ref` | `varchar(255)` | nullable, unique | Provider-side subscription ID (e.g. Stripe `sub_xxx`). `NULL` for free, manual, or offline-billed tenants. |
| `status` | `varchar(20)` | NOT NULL | Subscription state: `trialing` \| `active` \| `past_due` \| `unpaid` \| `canceled` \| `paused`. |
| `current_period_start` | `timestamptz` | NOT NULL | Start of the current billing cycle. |
| `current_period_end` | `timestamptz` | NOT NULL | End of the current billing cycle. Invoice is generated near this date. |
| `trial_ends_at` | `timestamptz` | nullable | End of the trial period. `NULL` if no trial applies. |
| `cancel_at_period_end` | `boolean` | NOT NULL, default `false` | If `true`, the subscription cancels automatically at `current_period_end` without renewal. |
| `canceled_at` | `timestamptz` | nullable | Timestamp when cancellation was confirmed, either immediately or at period end. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** unique on `provider_subscription_ref` (sparse), index on `tenant_id`, index on `status`

**Status values:** `trialing` | `active` | `past_due` | `unpaid` | `canceled` | `paused`

---

### `payment_method_refs`

Display-safe reference to an externally managed payment method.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `tenant_id` | `bigint` | NOT NULL, FK → `tenants.id` CASCADE, index | Owning tenant. |
| `billing_profile_id` | `bigint` | NOT NULL, FK → `billing_profiles.id` CASCADE, index | Associated billing profile. |
| `provider_customer_ref` | `varchar(255)` | NOT NULL | Provider-side customer ID (e.g. Stripe `cus_xxx`). Used to look up the customer during payment. |
| `provider_payment_method_ref` | `varchar(255)` | NOT NULL, unique | Provider-side payment method ID (e.g. Stripe `pm_xxx`). Must be unique to prevent duplicates. |
| `method_type` | `varchar(20)` | NOT NULL | Payment instrument category: `card` \| `bank_transfer` \| `wallet`. |
| `display_brand` | `varchar(50)` | nullable | Card network or wallet brand for display, e.g. `"Visa"`, `"Mastercard"`, `"PayPal"`. |
| `display_last4` | `varchar(4)` | nullable | Last 4 digits of the card or account number for identification in the UI. Never more than 4 digits. |
| `expires_at` | `date` | nullable | Card expiry date. Used to warn users of upcoming expiry. |
| `is_default` | `boolean` | NOT NULL, default `false` | If `true`, this method is used for automatic subscription charges. At most one per tenant. |
| `status` | `varchar(20)` | NOT NULL, default `'active'` | `active` \| `expired` \| `detached`. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** unique on `provider_payment_method_ref`, index on `tenant_id`

**Security constraint:** This table must never contain raw card numbers, CVV codes, bank account numbers, or any credential values.

---

### `invoices`

Snapshot of an issued billing document for a subscription period or adjustment.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `tenant_id` | `bigint` | NOT NULL, FK → `tenants.id` CASCADE, index | Owning tenant. |
| `subscription_id` | `bigint` | nullable, FK → `subscriptions.id` SET NULL, index | Associated subscription. `NULL` for one-time or manual invoices. |
| `provider_invoice_ref` | `varchar(255)` | nullable, unique | Provider-side invoice ID (e.g. Stripe `in_xxx`). |
| `invoice_number` | `varchar(50)` | NOT NULL, unique | Human-readable invoice number used in PDFs and communications, e.g. `INV-2026-00042`. |
| `currency_code` | `varchar(3)` | NOT NULL | ISO 4217 currency code matching the subscription's plan price. |
| `subtotal_minor` | `bigint` | NOT NULL | Gross amount before tax and adjustments, in minor currency units. |
| `tax_minor` | `bigint` | NOT NULL, default `0` | Tax amount in minor currency units. |
| `total_minor` | `bigint` | NOT NULL | `subtotal_minor + tax_minor`. |
| `amount_due_minor` | `bigint` | NOT NULL | Remaining amount to collect after any credits or discounts applied. |
| `status` | `varchar(20)` | NOT NULL | Invoice state: `draft` \| `open` \| `paid` \| `void` \| `uncollectible`. |
| `issued_at` | `timestamptz` | nullable | When the invoice was finalized and sent. `NULL` for drafts. |
| `due_at` | `timestamptz` | nullable | Payment due date. |
| `paid_at` | `timestamptz` | nullable | When full payment was received. |
| `voided_at` | `timestamptz` | nullable | When the invoice was voided. Voided invoices are not collected. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** unique on `invoice_number`, unique on `provider_invoice_ref` (sparse), index on `tenant_id`, index on `subscription_id`

**Status values:** `draft` | `open` | `paid` | `void` | `uncollectible`

**Immutability rule:** Issued invoices are not overwritten. Use status transitions, voiding, credits, or refunds for corrections.

---

### `payment_transactions`

Record of a payment attempt and its outcome.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `tenant_id` | `bigint` | NOT NULL, FK → `tenants.id` CASCADE, index | Owning tenant. Denormalized for efficient tenant-level payment history queries. |
| `invoice_id` | `bigint` | nullable, FK → `invoices.id` SET NULL, index | Invoice this transaction is attempting to pay. |
| `subscription_id` | `bigint` | nullable, FK → `subscriptions.id` SET NULL, index | Subscription this payment is associated with. |
| `provider_payment_ref` | `varchar(255)` | nullable, unique | Provider-side charge or payment intent ID (e.g. Stripe `ch_xxx`, `pi_xxx`). |
| `amount_minor` | `bigint` | NOT NULL | Amount collected or attempted, in minor currency units. |
| `currency_code` | `varchar(3)` | NOT NULL | ISO 4217 currency code. |
| `status` | `varchar(20)` | NOT NULL | Transaction outcome: `pending` \| `succeeded` \| `failed` \| `refunded` \| `credited`. |
| `failure_code` | `varchar(50)` | nullable | Provider or internal failure code, e.g. `card_declined`, `insufficient_funds`. |
| `failure_message` | `text` | nullable | Human-readable failure description from the provider. For internal logging only. |
| `attempted_at` | `timestamptz` | NOT NULL | When the payment attempt was initiated. |
| `settled_at` | `timestamptz` | nullable | When funds were confirmed settled. |
| `refunded_at` | `timestamptz` | nullable | When the transaction was fully or partially refunded. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** unique on `provider_payment_ref` (sparse), index on `tenant_id`, index on `invoice_id`

**Status values:** `pending` | `succeeded` | `failed` | `refunded` | `credited`

---

### `billing_adjustments`

Approved credit, refund, write-off, or manual billing correction linked to an invoice or transaction.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `tenant_id` | `bigint` | NOT NULL, FK → `tenants.id` CASCADE, index | Owning tenant. |
| `invoice_id` | `bigint` | nullable, FK → `invoices.id` SET NULL, index | The invoice this adjustment applies to. |
| `payment_transaction_id` | `bigint` | nullable, FK → `payment_transactions.id` SET NULL | The specific transaction being refunded or credited, if applicable. |
| `adjustment_type` | `varchar(20)` | NOT NULL | Type of correction: `credit` \| `refund` \| `write_off` \| `discount`. |
| `amount_minor` | `bigint` | NOT NULL | Adjustment amount in minor currency units. Always positive; direction is determined by `adjustment_type`. |
| `currency_code` | `varchar(3)` | NOT NULL | ISO 4217 currency code. |
| `reason` | `text` | NOT NULL | Internal justification for the adjustment, required for audit. |
| `status` | `varchar(20)` | NOT NULL, default `'applied'` | `applied` \| `reversed`. A reversed adjustment is no longer in effect. |
| `created_by` | `bigint` | NOT NULL, FK → `users.id` RESTRICT | The admin user who created the adjustment. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** index on `tenant_id`, index on `invoice_id`

**`adjustment_type` values:** `credit` | `refund` | `write_off` | `discount`

---

### `billing_overrides`

Approved exception that allows a tenant to maintain access without a successful payment collection.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `tenant_id` | `bigint` | NOT NULL, FK → `tenants.id` CASCADE, index | The tenant receiving the exception. |
| `subscription_id` | `bigint` | nullable, FK → `subscriptions.id` SET NULL | The subscription this override relates to. |
| `override_type` | `varchar(30)` | NOT NULL | Exception category: `manual_approval` \| `partnership_deal` \| `extended_grace` \| `free_access`. |
| `reason` | `text` | NOT NULL | Required justification recorded for audit. |
| `status` | `varchar(20)` | NOT NULL, default `'active'` | `active` \| `expired` \| `revoked`. |
| `effective_from` | `date` | NOT NULL | The date from which this override is valid. |
| `effective_until` | `date` | nullable | The date on which this override expires. `NULL` means indefinitely approved. |
| `approved_by` | `bigint` | NOT NULL, FK → `users.id` RESTRICT | The platform admin who approved the exception. |
| `revoked_by` | `bigint` | nullable, FK → `users.id` SET NULL | The platform admin who revoked the override before its expiry. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `revoked_at` | `timestamptz` | nullable | Timestamp of revocation. |

**Indexes:** index on `tenant_id`, composite index on `(tenant_id, status)`

**Constraint:** At most one active `billing_overrides` row per tenant and subscription at the same time.

**Constraint:** `billing_overrides` must not mutate invoice totals. Use `billing_adjustments` for credits, refunds, or write-offs.

---

### `billing_policies`

Grace period, dunning, and restriction settings used when evaluating subscription recovery and access restriction.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `code` | `varchar(50)` | NOT NULL, unique | Stable machine identifier, e.g. `default`, `enterprise_grace`, `strict`. |
| `name` | `varchar(120)` | NOT NULL | Human-readable label, e.g. `"Default Policy"`. |
| `grace_period_days` | `integer` | NOT NULL, default `0` | Number of days after `current_period_end` before access is restricted for non-payment. |
| `dunning_attempt_limit` | `integer` | NOT NULL, default `3` | Maximum number of automatic payment retry attempts during the dunning cycle before the subscription moves to `unpaid`. |
| `dunning_interval_days` | `integer` | NOT NULL, default `3` | Days between automatic retry attempts. |
| `restriction_mode` | `varchar(20)` | NOT NULL, default `'read_only'` | Access level applied when the grace period expires: `read_only` \| `suspended` \| `none`. |
| `status` | `varchar(20)` | NOT NULL, default `'active'` | `active` \| `deprecated`. |
| `effective_from` | `date` | NOT NULL | The date from which this policy applies to new subscriptions. |
| `effective_to` | `date` | nullable | The date after which this policy is no longer assigned to new subscriptions. `NULL` means open-ended. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** unique on `code`

**`restriction_mode` values:** `read_only` | `suspended` | `none`

---

### `billing_provider_events`

Idempotency record for external payment-provider webhook callbacks.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `bigint` | PK, auto-increment | Surrogate key. |
| `provider` | `varchar(50)` | NOT NULL | Name of the payment provider that sent the event, e.g. `stripe`, `payos`. |
| `provider_event_id` | `varchar(255)` | NOT NULL | Provider-assigned unique event ID, e.g. Stripe `evt_xxx`. Used for deduplication. |
| `event_type` | `varchar(100)` | NOT NULL | Provider event type string, e.g. `invoice.paid`, `customer.subscription.deleted`. |
| `received_at` | `timestamptz` | NOT NULL | When the platform received the webhook callback. |
| `processed_at` | `timestamptz` | nullable | When the event finished processing. `NULL` means in-progress or pending. |
| `processing_status` | `varchar(20)` | NOT NULL, default `'pending'` | Handler outcome: `pending` \| `succeeded` \| `failed` \| `ignored`. |
| `payload_ref` | `varchar(1000)` | nullable | Pointer to the stored raw payload (e.g. an object-storage path or log reference). The payload is not stored inline to keep the row size small. |
| `created_at` | `timestamptz` | NOT NULL, auto-set | Row creation timestamp. |
| `updated_at` | `timestamptz` | NOT NULL, auto-updated | Row last-modification timestamp. |

**Indexes:** composite unique on `(provider, provider_event_id)`, index on `processing_status`

**Idempotency rule:** Before processing, check for an existing row matching `(provider, provider_event_id)`. If found and `processing_status = 'succeeded'`, skip. If `failed`, retry if within the retry limit.

---

## Constraint Summary

| Rule | Details |
|---|---|
| At most one active subscription per tenant | Enforced by application service; only one `status IN ('trialing','active','past_due')` row per tenant. |
| Billable subscription references an active price | `plan_price_id` must point to a `plan_prices` row with `status = 'active'`. |
| Billing profile is unique per tenant | Unique constraint on `billing_profiles.tenant_id`. |
| Provider event IDs are unique per provider | Composite unique on `(provider, provider_event_id)` for idempotent handling. |
| Amounts are in minor currency units | Prevents decimal rounding. All `*_minor` columns are `bigint`. |
| Issued invoices are immutable | Status transitions, voiding, or adjustments are used; the issued row is not overwritten. |
| Provider references may be null | Free, manual, or offline-billed tenants have no provider references. |
| Payment method fields are display-safe only | No raw card, bank account, or credential values in `payment_method_refs`. |
| One active billing override at a time | At most one active `billing_overrides` row per `(tenant_id, subscription_id)`. |
| Overrides do not mutate invoices | Use `billing_adjustments` for credits, refunds, or write-offs. |

## Modeling Note

Billing state may influence tenant access through entitlement or suspension policy, but billing records do not replace the tenant lifecycle model.
Subscription recovery, payment success, or administrator override should restore access without recreating tenant, membership, channel, or message records.
