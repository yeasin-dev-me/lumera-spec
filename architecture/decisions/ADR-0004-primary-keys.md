# ADR-0004: Primary key & public identifier strategy
- Status: **Proposed — needs approval**
- Date: 2026-10-04

## Context
Choice between sequential integers and UUIDs affects URLs, enumeration risk, and join performance.

## Decision (proposed)
- Internal PK: `BIGINT` auto id on all tables (fast, Django default `BigAutoField`).
- Public identifier in APIs: `public_id` UUID4 for `User`, `Address`, `Order`, `Payment` (sensitive/enumerable); `slug` for `Category`/`Product`; other entities are only addressed through a parent or by id where non-sensitive (e.g. cart items by id under the user's own cart, with ownership enforced).

## Alternatives Considered
UUID PKs everywhere (simpler uniformity, larger indexes); integer ids exposed everywhere (enumeration risk — and ownership checks must still exist regardless).

## Consequences
Two identifiers on a few models. Ownership checks remain mandatory; opaque ids are defense-in-depth only.
