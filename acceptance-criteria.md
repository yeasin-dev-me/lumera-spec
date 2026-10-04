# Acceptance Criteria

## Format
Given / When / Then, tied to a user story. Detailed criteria are authored in each task file. Below are the **cross-cutting** criteria that apply to every feature.

## Cross-Cutting (apply to all tasks)
- AC-X1 Endpoint exists in `contracts/api-contract.yaml` and implementation matches it.
- AC-X2 Business-critical values (price, total, stock, permission) are derived server-side; a test sends tampered values and asserts they are ignored/rejected.
- AC-X3 Unauthenticated/unauthorized access returns `401`/`403` per the error format; tested.
- AC-X4 Validation failures return `400` in the standard error format with field-level details.
- AC-X5 Automated tests are added and CI is green.
- AC-X6 UI features are keyboard-accessible, responsive at all breakpoints, and have loading, empty, and error states.
- AC-X7 No new dependency without justification in the PR.
- AC-X8 Docs/contract updated when behavior changes.
- AC-X9 **Vertical slice:** a feature task covers backend, storefront, CMS (where an operator capability applies), and integration/E2E tests, or states explicitly why a part is out of scope.
- AC-X10 Every `/admin/` endpoint declares a required permission and is tested for 401 (no token), 403 (token without permission), and 200 (with permission); customer tokens are rejected.
- AC-X11 Admin writes create audit entries (actor, action, resource, redacted before/after); secrets and tokens never appear in responses, logs, or audit entries.
- AC-X12 CMS UI hides unauthorized actions for usability, but tests prove the backend rejects them regardless.

## Seed Examples (to be refined in feature tasks)
**US-05 Cart** — Given a logged-in customer, when they POST a cart item with a quantity exceeding stock, then the API returns `409`/`400` with code `insufficient_stock` and the cart is unchanged.

**US-07 Checkout** — Given a cart, when the client submits an order with a `price` field in the payload, then the field is ignored and order totals equal the server-calculated values.

**US-08 Payment** — Given an order, when the client claims payment success but the provider has not confirmed, then the payment status remains `pending`/unpaid.

**US-11 CMS price edit** — Given an operator with `products.write`, when they set `discount_price` ≥ `price`, then the API returns `400 validation_error`; when valid, the change is saved, audit-logged, and existing orders keep their snapshot prices. An operator without the permission receives `403` even by calling the API directly.

**US-19 Secrets** — Given a saved integration secret, when any admin or storefront API returns that integration, then the response contains `is_set: true` and never the secret value.
