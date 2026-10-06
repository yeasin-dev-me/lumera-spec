# ADR-0015: Order Lifecycle, Fulfillment, and Customer Notifications

- **Status:** Accepted
- **Date:** 2026-10-06
- **Authors:** Lumera Core Architecture Team
- **Deciders:** Engineering, Operations, Product
- **Consulted:** Operations, Customer Experience, Warehouse Leads
- **Informed:** All Engineering Teams
- **Related ADRs:**
  - [ADR-0013: Checkout and Order Creation Architecture](ADR-0013-checkout-and-orders.md)
  - [ADR-0014: Payment Architecture and Gateway Selection](ADR-0014-payment-architecture-and-gateway.md)
  - [ADR-0009: CMS Auth, RBAC, and Audit Logging](ADR-0009-cms-auth-rbac-audit.md)

---

## 1. Context and Problem Statement

Step 6 implemented transactional order creation with snapshot immutability, inventory reservation, and initial `Order.status = 'pending'`, `Order.payment_status = 'pending'`. Step 7 integrated server-side payment verification (SSLCommerz) transitioning `Order.payment_status` to `'paid'` and `Order.status` to `'confirmed'`.

To complete the commerce loop, Lumera requires operational fulfillment: taking a confirmed, paid order through physical processing, courier dispatch, tracking number assignment, delivery confirmation, customer cancellation handling, and transactional customer notifications.

Key architectural questions addressed in this ADR:
1. What is the authoritative order lifecycle, and how are `confirmed` and warehouse packing represented?
2. What are the rules separating payment state from fulfillment progression?
3. Who has authority to cancel orders, and what happens to inventory and money upon cancellation?
4. How is courier tracking represented, and how are external delivery callbacks handled?
5. How is actor auditing recorded without redundant database columns?
6. How are customer notifications dispatched reliably without jeopardizing order state integrity?

---

## 2. Decision Drivers

- **Authoritative Backend Single Source of Truth:** The backend strictly controls all state transitions and permissions.
- **Payment vs. Fulfillment Separation:** `Order.payment_status` (financial) and `Order.status` (operational fulfillment) must remain distinct. Payment confirmation does not imply shipment.
- **Inventory Integrity:** Stock was already decremented during order creation. Fulfillment must never deduct stock again. Order cancellation before shipment must restore stock atomically.
- **Notification Failure Isolation:** Transactional emails/SMS are asynchronous side-effects. Notification infrastructure failures must never roll back a valid business transaction.
- **Lean Operations:** Do not introduce redundant state hops (e.g., intermediate `packed` status) or complex distributed queues before they are operationally required.
- **Audit Simplicity:** Reuse existing `changed_by` field to capture actor responsibility (system, staff, customer) without adding redundant database columns.

---

## 3. Considered Options

### Option A: Insert `packed` and Replace `confirmed` with `processing`
- Remove `confirmed`, change paid orders directly to `processing`, and add an intermediate `packed` status (`pending` -> `processing` -> `packed` -> `shipped` -> `delivered`).
- *Drawback:* Breaks Step 7's payment transition logic (which already transitions `pending` -> `confirmed` upon payment verification) and forces warehouse staff to click an unnecessary extra `packed` step before shipping.

### Option B (Selected): Preserve `confirmed`, Omit `packed`, and Transition `processing` to `shipped`
- Maintain `confirmed` as the financially verified post-payment status.
- Allow staff to transition `confirmed` -> `processing` (warehouse picking/packing) -> `shipped` (dispatch with courier and tracking number) -> `delivered`.
- *Rationale:* Zero schema migration friction with existing status enums, 100% backward compatibility with Step 6 & 7, and standard, efficient operational workflow.

---

## 4. Decision Outcome

### 4.1 Authoritative Order Lifecycle

The legal order status lifecycle is:

```
                  ┌──────────────┐
                  │   pending    │
                  └──────┬───────┘
                         │ (Payment verified / paid)
                         ▼
                  ┌──────────────┐
                  │  confirmed   │
                  └──────┬───────┘
                         │ (Staff starts fulfillment)
                         ▼
                  ┌──────────────┐
                  │  processing  │
                  └──────┬───────┘
                         │ (Courier assigned + tracking #)
                         ▼
                  ┌──────────────┐
                  │   shipped    │
                  └──────┬───────┘
                         │ (Delivered to customer)
                         ▼
                  ┌──────────────┐
                  │  delivered   │ (Terminal)
                  └──────────────┘

  Cancellation paths:
  - pending    ──► cancelled (Customer or Staff)
  - confirmed  ──► cancelled (Staff only)
  - processing ──► cancelled (Staff only)
```

**Legal State Transitions:**
- `pending` -> `confirmed` (System via payment verification, or Staff)
- `pending` -> `cancelled` (Customer or Staff)
- `confirmed` -> `processing` (Staff with `orders.manage`)
- `confirmed` -> `cancelled` (Staff with `orders.manage`)
- `processing` -> `shipped` (Staff with `orders.manage`, requires `courier` & `tracking_number`)
- `processing` -> `cancelled` (Staff with `orders.manage`)
- `shipped` -> `delivered` (Staff with `orders.manage`)

**Terminal State Invariants:**
- `delivered` and `cancelled` are terminal. Any attempt to transition out of them is rejected with `409 Conflict`.
- Skipping statuses (e.g. `pending` -> `shipped`) is strictly rejected.

### 4.2 Payment Prerequisite for Fulfillment
- Normal fulfillment (`processing`, `shipped`, `delivered`) requires `order.payment_status == 'paid'`.
- Attempting to transition an unpaid order to `processing` or `shipped` is rejected with `409 Conflict`.

### 4.3 Cancellation Authority & Paid-Order Semantics
- **Customer Cancellation Authority:** A customer may cancel their own order via `POST /api/v1/customer/orders/{public_id}/cancel/` only if `order.status == 'pending'` and `order.payment_status == 'pending'`. Once confirmed or paid, self-serve customer cancellation is disallowed.
- **Staff Cancellation Authority:** Staff holding `orders.manage` can cancel orders in `pending`, `confirmed`, or `processing` status. Orders in `shipped` or `delivered` status cannot be cancelled.
- **Paid-Order Behavior:** Cancelling a paid order transitions `order.status` to `'cancelled'`. `order.payment_status` **remains `'paid'`**. Cancellation does **not** trigger an automatic payment refund. Refunds remain an explicit financial accounting workflow.

### 4.4 Inventory Restoration Semantics
- **During Cancellation:** When an order is cancelled from `pending`, `confirmed`, or `processing`, all associated items' quantities are atomically restored to their respective products:
  $$\text{product.stock\_quantity} \leftarrow \text{product.stock\_quantity} + \text{item.quantity}$$
- **During Fulfillment:** Progression through `confirmed` -> `processing` -> `shipped` -> `delivered` **never** touches inventory. Inventory was already deducted atomically at order placement (Step 6).

### 4.5 Fulfillment Data Model & Courier Abstraction
- Extended `Order` model with operational fulfillment fields:
  - `courier`: CharField (e.g., `"Steadfast"`, `"Pathao"`, `"Paperfly"`, `"DHL"`, `"In-House"`)
  - `tracking_number`: CharField
  - `tracking_url`: URLField (optional direct link)
  - `shipped_at`: DateTimeField (set when status becomes `shipped`)
  - `delivered_at`: DateTimeField (set when status becomes `delivered`)
- **Transition Requirement:** Transitioning to `shipped` strictly validates that `courier` and `tracking_number` are provided and non-empty.
- **Delivery Callbacks:** External courier webhook callbacks are deferred until a live courier provider API is integrated. Delivery is marked authoritatively by staff.

### 4.6 Actor Tracking via Existing `changed_by`
- `OrderStatusHistory` retains its `changed_by` foreign key (nullable to `User`):
  - `changed_by is None` -> System action (e.g. payment webhook).
  - `changed_by.is_staff` -> Staff operator action.
  - `changed_by == order.user` -> Customer action.
- The API dynamically resolves `actor_type` (`"system"`, `"staff"`, `"customer"`) in serializers without storing a redundant database column.

### 4.7 Notification Architecture & Failure Isolation
- **Domain Events:** Transactional notifications are defined for major milestones: `order_placed`, `payment_confirmed`, `order_processing`, `order_shipped`, `order_delivered`, `order_cancelled`.
- **Failure Isolation:** Notifications are dispatched post-commit via `transaction.on_commit()` inside fail-safe handlers. An email delivery timeout or SMTP error will be logged as failed in the `Notification` record but will **never** roll back the database order transition.
- **Idempotency:** Each notification event generates an idempotency key (`{order.public_id}:{event_type}`) ensuring duplicate triggers do not spam customers.

---

## 5. Security & RBAC Considerations

- **Single Permission Code:** All operational lifecycle mutations (`transition`, `fulfillment`) require `HasPermission("orders.manage")`. Read queries require `HasPermission("orders.read")`.
- **Customer Isolation:** Customers can only view their own order timeline (`GET /api/v1/customer/orders/{public_id}/timeline/`) and cancel their own unpaid pending orders (`POST /api/v1/customer/orders/{public_id}/cancel/`).
- **Normalized Error Handling:**
  - `400 Bad Request`: Validation failure (e.g. shipping without tracking number).
  - `401 Unauthorized`: Unauthenticated request.
  - `403 Forbidden`: Insufficient permissions or ownership mismatch.
  - `404 Not Found`: Order does not exist.
  - `409 Conflict`: Illegal state transition or terminal state mutation attempt.

---

## 6. Implementation & Verification Plan

1. **`lumera-spec`:** Publish ADR-0015 and update OpenAPI contracts.
2. **`lumera-backend`:** Add fulfillment fields to `Order`, implement `transition_order_status`, `cancel_order`, and fail-safe notification services; expose admin and customer endpoints.
3. **`lumera-frontend`:** Render interactive order timeline and courier tracking card in customer order view.
4. **`lumera-cms`:** Implement state-gated transition actions, courier tracking form, and status history audit timeline in order management.
5. **`lumera-integration`:** Validate end-to-end fulfillment, inventory restoration on cancellation, tracking, and notification failure isolation.
6. **`lumera-devops`:** Configure notification environment variables in `.env.example`.
