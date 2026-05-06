# Payment And Subscription Billing API

These endpoints are planned. Billing APIs operate at tenant scope and assume payment credentials are collected by an external payment provider; the platform stores only provider references and display-safe payment method metadata.

## Resource Shapes

Billing profile:

```json
{
  "id": 7001,
  "tenant_id": 1001,
  "billing_email": "billing@acme.example",
  "company_name": "Acme",
  "tax_id": "US-123456789",
  "billing_address": {},
  "invoice_delivery_preferences": {},
  "status": "active"
}
```

Subscription:

```json
{
  "id": 7101,
  "tenant_id": 1001,
  "plan_id": 1,
  "plan_price_id": 101,
  "billing_policy_id": 8001,
  "status": "active",
  "current_period_start": "2026-05-01T00:00:00.000Z",
  "current_period_end": "2026-06-01T00:00:00.000Z",
  "trial_ends_at": null,
  "cancel_at_period_end": false
}
```

Payment method reference:

```json
{
  "id": 7201,
  "tenant_id": 1001,
  "method_type": "card",
  "display_brand": "visa",
  "display_last4": "4242",
  "expires_at": "2028-12-31T23:59:59.000Z",
  "is_default": true,
  "status": "active"
}
```

Billing policy:

```json
{
  "id": 8001,
  "code": "standard-card-dunning",
  "name": "Standard Card Dunning",
  "grace_period_days": 7,
  "dunning_attempt_limit": 3,
  "restriction_mode": "restrict_then_suspend",
  "status": "active",
  "effective_from": "2026-05-01T00:00:00.000Z",
  "effective_to": null
}
```

Billing override:

```json
{
  "id": 8101,
  "tenant_id": 1001,
  "subscription_id": 7101,
  "override_type": "manual_billing",
  "reason": "Approved annual offline invoice",
  "status": "active",
  "effective_from": "2026-05-01T00:00:00.000Z",
  "effective_until": "2027-05-01T00:00:00.000Z",
  "approved_by": "29c68a53-6632-41b6-aabd-0678b7115c5d",
  "revoked_by": null,
  "created_at": "2026-05-01T00:00:00.000Z",
  "revoked_at": null
}
```

Invoice:

```json
{
  "id": 7301,
  "tenant_id": 1001,
  "subscription_id": 7101,
  "invoice_number": "INV-2026-0001",
  "currency_code": "USD",
  "subtotal_minor": 2900,
  "tax_minor": 0,
  "total_minor": 2900,
  "amount_due_minor": 0,
  "status": "paid",
  "issued_at": "2026-05-01T00:00:00.000Z",
  "due_at": "2026-05-08T00:00:00.000Z",
  "paid_at": "2026-05-01T00:05:00.000Z"
}
```

## Endpoint Summary

| Method | Path | Purpose |
|---|---|---|
| GET | `/plans/{planId}/prices` | List active prices for a plan. |
| POST | `/plans/{planId}/prices` | Create a plan price. |
| PATCH | `/plan-prices/{priceId}` | Update status or effective-date metadata for a plan price. |
| GET | `/billing-policies` | List active billing policies. |
| POST | `/billing-policies` | Create a billing policy. |
| PATCH | `/billing-policies/{policyId}` | Update billing policy status or effective dates. |
| GET | `/tenants/{tenantId}/billing-profile` | Get billing profile. |
| PUT | `/tenants/{tenantId}/billing-profile` | Create or replace billing profile. |
| GET | `/tenants/{tenantId}/subscription` | Get current subscription. |
| POST | `/tenants/{tenantId}/subscription` | Start subscription. |
| PATCH | `/tenants/{tenantId}/subscription` | Change plan, billing interval, or cancellation settings. |
| GET | `/tenants/{tenantId}/billing-overrides` | List manual billing exceptions or overrides. |
| POST | `/tenants/{tenantId}/billing-overrides` | Apply manual billing exception or override. |
| POST | `/billing-overrides/{overrideId}/revoke` | Revoke an active billing override. |
| GET | `/tenants/{tenantId}/payment-methods` | List display-safe payment method references. |
| POST | `/tenants/{tenantId}/payment-methods/setup-intent` | Start provider setup flow for a payment method. |
| PATCH | `/payment-methods/{paymentMethodId}` | Set default or disable a payment method reference. |
| GET | `/tenants/{tenantId}/invoices` | List invoices. |
| GET | `/invoices/{invoiceId}` | Get invoice detail. |
| GET | `/tenants/{tenantId}/payment-transactions` | List payment attempts and outcomes. |
| POST | `/billing/provider-events` | Receive payment-provider callbacks. |

## POST `/plans/{planId}/prices`

Requires platform billing administration permission.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `code` | string | yes | Unique price code. |
| `currency_code` | string | yes | ISO 4217 currency code. |
| `amount_minor` | integer | yes | Amount in minor currency units. |
| `billing_interval` | string | yes | One of `month`, `year`, or `one_time`. |
| `trial_days` | integer | no | Non-negative integer. |
| `billing_policy_id` | integer | no | Optional policy override for subscriptions using this price. |
| `effective_from` | string | no | ISO timestamp. |
| `effective_to` | string | no | ISO timestamp after `effective_from`. |

Success `data`: plan price resource.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 403 | `FORBIDDEN` | Caller cannot manage plan prices. |
| 404 | `PLAN-NOT_FOUND` | Plan was not found. |
| 404 | `BILLING-POLICY_NOT_FOUND` | Billing policy was not found. |
| 409 | `BILLING-PRICE_CODE_ALREADY_EXISTS` | Price code already exists. |

## POST `/billing-policies`

Requires platform billing administration permission.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `code` | string | yes | Unique policy code. |
| `name` | string | yes | Display name. |
| `grace_period_days` | integer | yes | Non-negative number of days before access restriction may begin. |
| `dunning_attempt_limit` | integer | yes | Non-negative number of failed collection attempts before unpaid handling. |
| `restriction_mode` | string | yes | One of `none`, `restrict_only`, `suspend`, or `restrict_then_suspend`. |
| `effective_from` | string | no | ISO timestamp. |
| `effective_to` | string | no | ISO timestamp after `effective_from`. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 403 | `FORBIDDEN` | Caller cannot manage billing policies. |
| 409 | `BILLING-POLICY_CODE_ALREADY_EXISTS` | Policy code already exists. |

## PUT `/tenants/{tenantId}/billing-profile`

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `billing_email` | string | yes | Valid email address. |
| `company_name` | string | no | Billing legal or display name. |
| `tax_id` | string | no | Tax identifier when supplied by the customer. |
| `billing_address` | object | no | Provider-safe billing address payload. |
| `invoice_delivery_preferences` | object | no | Email and delivery preferences. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 403 | `FORBIDDEN` | Caller cannot manage tenant billing. |
| 404 | `TENANT-NOT_FOUND` | Tenant was not found. |

## POST `/tenants/{tenantId}/subscription`

Starts a tenant subscription. The request may require a billing profile and payment method reference unless the tenant is free, trial-only, manually billed, or covered by an approved override.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `plan_id` | integer | yes | Active billable plan. |
| `plan_price_id` | integer | yes | Active price for the selected plan. |
| `billing_policy_id` | integer | no | Optional policy selected by platform rules; defaults from price or platform policy. |
| `payment_method_ref_id` | integer | no | Required when provider policy requires a default payment method. |
| `trial_requested` | boolean | no | Subject to pricing and eligibility policy. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `BILLING-MISSING_PROFILE` | Billing profile is required before subscription start. |
| 400 | `BILLING-INVALID_PLAN_PRICE` | Plan price does not belong to the selected plan or is inactive. |
| 400 | `BILLING-INVALID_POLICY` | Billing policy is inactive or not applicable. |
| 400 | `BILLING-MISSING_PAYMENT_METHOD` | Payment method is required for this subscription. |
| 403 | `FORBIDDEN` | Caller cannot manage tenant subscription. |
| 404 | `TENANT-NOT_FOUND` | Tenant was not found. |
| 409 | `BILLING-ACTIVE_SUBSCRIPTION_EXISTS` | Tenant already has an active subscription. |

## POST `/tenants/{tenantId}/billing-overrides`

Requires permission to apply tenant billing overrides. An override satisfies billing access policy but does not alter issued invoice totals or payment transaction history.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `subscription_id` | integer | no | Optional subscription covered by the override. |
| `override_type` | string | yes | One of `manual_billing`, `trial_extension`, `complimentary_access`, or `payment_exception`. |
| `reason` | string | yes | Audit reason for the approved exception. |
| `effective_from` | string | no | Defaults to current time. |
| `effective_until` | string | no | Required unless policy permits open-ended override. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 403 | `FORBIDDEN` | Caller cannot apply billing overrides. |
| 404 | `TENANT-NOT_FOUND` | Tenant was not found. |
| 404 | `BILLING-SUBSCRIPTION_NOT_FOUND` | Subscription was not found. |
| 409 | `BILLING-ACTIVE_OVERRIDE_EXISTS` | An active overlapping override already exists. |

## POST `/billing-overrides/{overrideId}/revoke`

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `reason` | string | yes | Audit reason for revocation. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot revoke billing overrides. |
| 404 | `BILLING-OVERRIDE_NOT_FOUND` | Billing override was not found. |
| 409 | `BILLING-OVERRIDE_NOT_ACTIVE` | Only active overrides can be revoked. |

## PATCH `/tenants/{tenantId}/subscription`

Request body accepts one supported change at a time:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `plan_price_id` | integer | no | Changes the plan or billing interval. |
| `cancel_at_period_end` | boolean | no | Schedules or clears end-of-period cancellation. |
| `cancel_immediately` | boolean | no | Cancels now when policy allows. |
| `reason` | string | no | Audit reason. |

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `BILLING-INVALID_SUBSCRIPTION_CHANGE` | Requested change is not allowed. |
| 403 | `FORBIDDEN` | Caller cannot manage tenant subscription. |
| 404 | `BILLING-SUBSCRIPTION_NOT_FOUND` | Subscription was not found. |
| 409 | `BILLING-SUBSCRIPTION_STATE_CONFLICT` | Current state does not allow the requested change. |

## POST `/tenants/{tenantId}/payment-methods/setup-intent`

Creates a provider setup flow token or redirect payload. The API must not accept raw card, bank account, or sensitive payment credential values.

Request body:

| Field | Type | Required | Rules |
|---|---|---:|---|
| `return_url` | string | yes | Client URL to return to after provider setup. |
| `method_type` | string | no | Provider-supported payment method type. |

Success `data` includes provider setup information only.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | Validation failed. |
| 403 | `FORBIDDEN` | Caller cannot manage payment methods. |
| 404 | `TENANT-NOT_FOUND` | Tenant was not found. |

## GET `/tenants/{tenantId}/invoices`

Query:

| Field | Type | Notes |
|---|---|---|
| `status` | string | Optional invoice status filter. |
| `limit` | integer | Pagination limit. |
| `cursor` | string | Pagination cursor. |

Success `data` is a paginated invoice list.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 403 | `FORBIDDEN` | Caller cannot view tenant billing. |
| 404 | `TENANT-NOT_FOUND` | Tenant was not found. |

## POST `/billing/provider-events`

Public-to-provider callback endpoint protected by provider signature verification. Processing must be idempotent by provider and provider event ID.

Request body is provider-specific and should be persisted by reference or safe storage policy rather than copied into domain records.

Provider events can reconcile subscription, invoice, payment method reference, and payment transaction state. The callback record is not limited to one payment transaction because providers may send subscription lifecycle, invoice, or payment setup events.

Errors:

| HTTP | Code | Message |
|---:|---|---|
| 400 | `BILLING-INVALID_PROVIDER_EVENT` | Event payload or signature is invalid. |
| 409 | `BILLING-PROVIDER_EVENT_ALREADY_PROCESSED` | Event was already processed. |
| 500 | `BILLING-PROVIDER_EVENT_PROCESSING_FAILED` | Event could not be processed safely. |
