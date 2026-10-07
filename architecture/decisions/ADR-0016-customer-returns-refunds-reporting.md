# ADR-0016: Customer Profiles, Returns, Refund Foundations, and Operational Reporting
- Status: Accepted
- Date: 2026-10-06
- Deciders: Lumera product and engineering owners

## Context

Step 9 adds customer profile and saved-address management, delivered-order return
requests, an internal refund foundation, and operational reporting. The existing
backend already owns customer identity, immutable order shipping snapshots,
payment state, order lifecycle, and staff permissions. The step must extend those
domains without creating a second payment system or allowing customer/CMS input
to mutate authoritative financial state.

## Decision

### Customer profile and saved addresses

The existing `GET /api/v1/customer/me/` endpoint remains the canonical customer
profile endpoint and also supports `PATCH` for `first_name`, `last_name`, and
`phone`. Email changes are deferred until an email-verification flow exists.
Password, role, staff, activation, and authentication fields are never writable
through this endpoint.

Saved addresses are customer-owned records accessed only through authenticated,
user-scoped queries. The backend enforces at most one default address per
customer. Deleting a default promotes the oldest remaining address, or leaves
the customer with no default when none remain. Checkout may use an address as
input convenience, but order shipping fields remain copied historical snapshots
and never reference mutable address data.

### Returns

Only delivered orders are eligible for customer return requests. A return
request belongs to the authenticated customer and one order, and its items
reference the original `OrderItem` records with requested quantities. The
backend calculates remaining eligible quantity and rejects mismatched orders,
invalid quantities, duplicate active requests, and unauthorized access.

Return states are:

```text
requested -> approved -> received
requested -> rejected
requested -> cancelled
```

Return approval or receipt does not automatically refund or restock inventory.
Inventory return-to-stock behavior is deferred until warehouse inspection
workflow is implemented.

### Refund foundation

Refund records reference the authoritative `Order` and paid `Payment`, and may
reference a `ReturnRequest`. A refund amount must be positive and cannot exceed
the remaining refundable amount. Amount validation and creation occur inside a
transaction with appropriate row locking.

Refund records are internal operational records. They do not represent external
gateway settlement. In Step 9, staff may create or approve an internal refund
record, but the backend must not mark payment or order payment status as
refunded and must not mark the refund as externally completed without a future
gateway settlement confirmation.

### Reporting

Reporting is computed by backend database aggregation and protected by an
explicit `reports.read` permission. Revenue is reported in the configured order
currency (currently USD):

- `gross_revenue`: totals for orders with an authoritative paid payment in the
  selected payment-completion period
- `refunded_amount`: externally settled refunds only; Step 9 internal refund
  records do not reduce this value
- `net_revenue`: gross revenue minus externally settled refunds

Order counts use order creation time. Payment revenue uses payment completion
time. Return and internal refund counts use their creation time. Shipment and
delivery metrics use their corresponding fulfillment timestamps or status
history transitions. Date filters use inclusive start and exclusive end
boundaries.

## Alternatives Considered

- Adding a separate `/customer/profile/` endpoint: rejected because `/customer/me/`
  already represents the authenticated profile.
- Linking historical orders directly to mutable addresses: rejected because it
  would compromise historical shipping integrity.
- Treating return approval as a completed refund: rejected because approval is
  not gateway settlement.
- Calculating reports in the CMS: rejected because the backend is the financial
  and operational source of truth.

## Consequences

Step 9 remains compatible with the existing order and payment state machines,
requires no gateway refund API, and provides a safe operational foundation.
Gateway settlement, warehouse inspection, automatic restocking, and email
verification remain future work.
