# ADR-0005: Payment provider
- Status: **Proposed / Open — product owner input required**
- Date: 2026-10-04

## Context
Payments are in scope but no provider/gateway is named. Provider availability depends on the business's country and currency, which are also unspecified.

## Decision
**None yet.** Not invented here. Constraints that apply to any provider:
- Card data never touches Lumera servers (hosted checkout/elements).
- Order becomes `paid` only after server-side verification (webhook with signature check and/or server fetch).
- Idempotent webhook handling; unique `(provider, provider_reference)`.
- Provider credentials per environment via secrets; test mode in dev/staging.
- `payments` app hides the provider behind one small internal interface only when a second provider is actually needed.

## Consequences
Payments feature tasks are blocked until this is decided. Contract includes no payment endpoints yet.
