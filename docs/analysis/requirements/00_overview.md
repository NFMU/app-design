# Phase 1 Requirements Overview

## Goal

Phase 1 delivers the foundation of an internal collaboration platform with:

- identity and account access
- tenant and workspace onboarding
- channel-based communication
- real-time messaging
- personal profile and preference management
- role-based access control across platform, tenant or workspace, channel, and self scopes
- payment and subscription billing for tenant accounts

## Primary Actors

- `Super Admin`: platform-level operator who can bootstrap and govern tenants
- `Tenant Admin`: organization-level administrator
- `Workspace Admin`: day-to-day administrator for the collaboration boundary used by members
- `Billing Admin`: tenant-level operator who manages subscription, invoices, and payment settings
- `Channel Admin`: channel-level moderator and operator
- `Member`: authenticated end user
- `Guest / Invited User`: unauthenticated or partially onboarded user

## Business Scope

The product must support a multi-organization collaboration model where:

- users can own one account and participate in multiple collaboration spaces
- collaboration happens inside channels
- channels contain real-time messages and lightweight moderation
- memberships and roles control access
- personal preferences affect the way each user experiences the same channel set
- tenant accounts can be monetized through plans, subscriptions, invoices, and payment collection

## Conceptual Boundaries

The business language distinguishes these concepts:

- `Tenant`: organization or top-level customer account
- `Workspace`: day-to-day collaboration boundary created for a tenant
- `Channel`: discussion space inside the collaboration boundary
- `Membership`: persisted participation of a user in the collaboration boundary
- `Role`: a bundle of permissions assignable to members
- `Plan`: commercial and capability package available to a tenant
- `Subscription`: billing lifecycle that applies a plan to a tenant over time
- `Billing Profile`: contact, business, and payment-administration information for a tenant
- `Invoice`: bill issued for a subscription period or adjustment
- `Payment Method Reference`: tokenized pointer to an externally managed payment method
- `Payment Transaction`: recorded attempt to collect, settle, refund, or fail a payment
- `Billing Policy`: grace-period, dunning, and access restriction rules for subscription recovery
- `Billing Override`: approved exception that can satisfy billing access policy without provider payment

The requirements keep both `tenant` and `workspace` language because that is how the business scope is described.
Any technical simplification must be called out explicitly in the specs.

## Cross-Cutting Requirements

- Every persisted business record must have a clear lifecycle and ownership rule.
- Invitations, memberships, channel joins, and role assignments must be auditable.
- Soft-delete behavior is preferred where restoring or historical tracking matters.
- The system must support authorization at API, UI, and socket or event interaction points.
- Real-time messaging and unread tracking are part of the MVP, not post-MVP enhancements.
- Billing, subscription, invoice, payment, and refund records must be auditable.
- Billing policies and manual billing overrides must be auditable and explain billing-driven access decisions.
- Raw card or bank account details must not be stored by the collaboration platform.
- Payment-provider callbacks must be processed idempotently because external providers may retry or reorder events.
- Billing-driven restriction or suspension must preserve tenant data, memberships, channels, and message history.
