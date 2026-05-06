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

- `plans` remains the product and capability catalog owned by the collaboration foundation.
- `plan_prices` adds currency, amount, interval, trial, and effective-date details for billable plans.
- `billing_profiles` belongs to one `tenants` record and stores billing contact, business, tax, and invoice-delivery data.
- `subscriptions` belongs to one `tenants` record and applies one plan and pricing option over a billable period.
- `payment_method_refs` stores externally managed payment method references and display-safe metadata only.
- `invoices` snapshot issued billing amounts, periods, and delivery status.
- `payment_transactions` records attempts and outcomes for collecting, settling, failing, refunding, or crediting payment.
- `billing_adjustments` records approved credits, refunds, write-offs, or manual billing corrections.
- `billing_overrides` records approved tenant billing exceptions that can temporarily or permanently satisfy billing access policy without collecting payment through the provider.
- `billing_policies` defines grace period, dunning, and restriction settings used to evaluate subscription recovery and access restriction.
- `billing_provider_events` stores external callback identity and processing state for idempotent event handling.

## Suggested Core Attributes

`plan_prices`
- `id`
- `plan_id`
- `billing_policy_id`
- `code`
- `currency_code`
- `amount_minor`
- `billing_interval`
- `trial_days`
- `status`
- `effective_from`
- `effective_to`

`billing_profiles`
- `id`
- `tenant_id`
- `billing_email`
- `company_name`
- `tax_id`
- `billing_address_json`
- `invoice_delivery_preferences_json`
- `status`

`subscriptions`
- `id`
- `tenant_id`
- `plan_id`
- `plan_price_id`
- `billing_profile_id`
- `billing_policy_id`
- `provider_subscription_ref`
- `status`
- `current_period_start`
- `current_period_end`
- `trial_ends_at`
- `cancel_at_period_end`
- `canceled_at`

`payment_method_refs`
- `id`
- `tenant_id`
- `billing_profile_id`
- `provider_customer_ref`
- `provider_payment_method_ref`
- `method_type`
- `display_brand`
- `display_last4`
- `expires_at`
- `is_default`
- `status`

`invoices`
- `id`
- `tenant_id`
- `subscription_id`
- `provider_invoice_ref`
- `invoice_number`
- `currency_code`
- `subtotal_minor`
- `tax_minor`
- `total_minor`
- `amount_due_minor`
- `status`
- `issued_at`
- `due_at`
- `paid_at`
- `voided_at`

`payment_transactions`
- `id`
- `tenant_id`
- `invoice_id`
- `subscription_id`
- `provider_payment_ref`
- `amount_minor`
- `currency_code`
- `status`
- `failure_code`
- `failure_message`
- `attempted_at`
- `settled_at`
- `refunded_at`

`billing_adjustments`
- `id`
- `tenant_id`
- `invoice_id`
- `payment_transaction_id`
- `adjustment_type`
- `amount_minor`
- `currency_code`
- `reason`
- `status`
- `created_by`
- `created_at`

`billing_overrides`
- `id`
- `tenant_id`
- `subscription_id`
- `override_type`
- `reason`
- `status`
- `effective_from`
- `effective_until`
- `approved_by`
- `revoked_by`
- `created_at`
- `revoked_at`

`billing_policies`
- `id`
- `code`
- `name`
- `grace_period_days`
- `dunning_attempt_limit`
- `restriction_mode`
- `status`
- `effective_from`
- `effective_to`

`billing_provider_events`
- `id`
- `provider`
- `provider_event_id`
- `event_type`
- `received_at`
- `processed_at`
- `processing_status`
- `payload_ref`

## Constraint Hints

- A tenant may have at most one active subscription in Phase 1.
- A billable subscription should reference one active `plan_prices` row.
- A subscription should reference the billing policy used to evaluate past-due, unpaid, and restriction transitions.
- A plan price may reference the default billing policy used when a subscription does not explicitly select another active policy.
- `billing_profiles.tenant_id` should be unique in Phase 1 unless multiple billing accounts are explicitly introduced later.
- `provider_event_id` should be unique per provider to support idempotent callback processing.
- Invoice and transaction amounts should use minor currency units to avoid decimal rounding ambiguity.
- Issued invoices should not be overwritten; use status transitions, voiding, credits, or refunds for corrections.
- Provider references may be nullable only for free, manual, or approved offline-billed tenants.
- At most one active `billing_overrides` record should apply to a tenant and subscription at the same time.
- Billing overrides must not mutate invoice totals; use `billing_adjustments` for credits, refunds, write-offs, or billing corrections.
- Payment method fields must remain display-safe and must not contain raw card, bank account, or credential values.

## Modeling Note

Billing state may influence tenant access through entitlement or suspension policy, but billing records do not replace the tenant lifecycle model. Subscription recovery, payment success, or administrator override should restore access without recreating tenant, membership, channel, or message records.
