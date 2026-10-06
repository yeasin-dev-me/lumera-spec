# ADR-0014: Payment Architecture, Gateway Abstraction, and Server-Side Verification

- **Status:** Accepted
- **Date:** 2026-10-06
- **Deciders:** Lumera Core Architecture Team
- **Consulted:** Backend, Frontend, CMS, DevOps, Security Leads
- **Informed:** Entire Project
- **Supersedes:** [ADR-0005: Payment provider](ADR-0005-payment-provider.md)

---

## 1. Context and Problem Statement

Step 6 implemented transactional checkout, order creation with immutable product and price snapshots, shipping address snapshots, and inventory deductions ([ADR-0013](ADR-0013-checkout-and-orders.md)). At the conclusion of Step 6, orders are placed with `status = "pending"` and `payment_status = "pending"`.

Step 7 must connect the existing order system to a secure, production-grade payment architecture. 

The architecture must address:
1. How to maintain a gateway-agnostic payment domain that can easily accommodate future gateways (bKash, Nagad, Stripe, etc.) without coupling core order logic to a specific provider.
2. Which payment gateway best matches Lumera's initial deployment context (Bangladesh-first beauty and skincare e-commerce).
3. How to guarantee that clients cannot manipulate payment amounts, currency, or payment status.
4. How to ensure secure, idempotent callback and webhook processing with authoritative server-side verification.
5. How to guarantee concurrency safety and terminal payment state immutability.

---

## 2. Decision Drivers

- **Backend Authority (BR-01, BR-03):** The backend is the single source of truth. Payment amounts, currencies, and success states are determined strictly by the backend. Client browser inputs, redirects, or query parameters are never trusted.
- **Order & Payment Separation:** An order represents a commercial agreement; payments represent financial settlements. An order may have multiple payment attempts, but only one authoritative successful payment can mark the order as `paid`.
- **Bangladesh-First Deployment Context:** Lumera operates primarily in Bangladesh (BDT / Cards / MFS). The gateway must natively support major Bangladeshi payment methods (bKash, Nagad, Rocket, local Visa/Mastercard/Amex, and internet banking).
- **Server-Side Verification & Defense in Depth:** Hosted payment gateway callbacks cannot be trusted at face value. The backend must query the gateway's validation API server-to-server before marking any record as `paid`.
- **Idempotency & Concurrency Safety:** Webhooks and callbacks may arrive multiple times or concurrently. Processing must be idempotent with row locking (`select_for_update`) and database constraints.
- **Secret Isolation:** Gateway credentials, merchant secrets, and API keys remain strictly on the backend and are loaded via environment configuration.

---

## 3. Decision

### 3.1 Gateway Selection
For Step 7, **SSLCommerz** is selected as the primary gateway:
- It is the leading payment gateway aggregator in Bangladesh, providing unified support for credit/debit cards, bKash, Nagad, Rocket, and net banking without requiring separate direct merchant integrations.
- It provides a standardized hosted session initiation API (`gwprocess/v4/api.php`), IPN callback support, and a robust server-side transaction validation API (`validator/api/validationserverAPI.php`).
- It is fully configurable via environment variables (`PAYMENT_SSLCOMMERZ_STORE_ID`, `PAYMENT_SSLCOMMERZ_STORE_PASSWORD`, `PAYMENT_SSLCOMMERZ_IS_SANDBOX`).

### 3.2 Gateway Abstraction
To keep the architecture gateway-agnostic, we introduce an abstract base class `PaymentGateway`:
```python
class PaymentGateway(ABC):
    gateway_id: str

    @abstractmethod
    def initiate(self, payment: Payment, return_urls: dict[str, str]) -> dict[str, Any]:
        """Initiate payment session with provider. Returns redirect/session details."""
        ...

    @abstractmethod
    def parse_callback(self, request_data: dict[str, Any]) -> dict[str, Any]:
        """Normalize raw webhook/callback payload."""
        ...

    @abstractmethod
    def verify(self, payment: Payment, callback_data: dict[str, Any]) -> dict[str, Any]:
        """Perform server-to-server transaction validation."""
        ...
```
A gateway registry (`get_gateway(gateway_id)`) instantiates the configured adapter. Additional gateways (bKash Direct, Stripe) can be added in future steps by implementing `PaymentGateway` without modifying order or payment business services.

### 3.3 Domain Data Model (`apps.payments`)
1. **`Payment`**:
   - `public_id`: UUID4 for external and API references.
   - `order`: ForeignKey to `apps.orders.models.Order` (`on_delete=models.PROTECT`, `related_name="payments"`).
   - `gateway`: Identifier string (e.g. `"sslcommerz"`).
   - `method`: Snapshotted payment method (e.g. `"card"`, `"bkash"`, `"nagad"`).
   - `amount`: Snapshotted authoritative order total (`NUMERIC(12, 2)`).
   - `currency`: Currency code (matching `order.currency`).
   - `status`: Authoritative payment state (`pending`, `initiated`, `processing`, `paid`, `failed`, `cancelled`, `refunded`).
   - `provider_transaction_id`: Gateway transaction identifier (e.g. `bank_tran_id` / `val_id`).
   - `provider_reference`: Gateway session or transaction reference (e.g. `tran_id` / `sessionkey`).
   - `paid_at`: Timestamp when authoritative verification succeeded.
   - `metadata`: Sanitized non-sensitive gateway response metadata.
   - **Database Constraints:**
     - Check constraint: `amount > 0`.
     - Partial Unique Constraint: `UniqueConstraint(fields=["order"], condition=Q(status="paid"), name="unique_paid_payment_per_order")` ensures an order can never have more than one successful payment.

2. **`PaymentAttempt`**:
   - Captures individual gateway interactions for retry auditing.
   - Fields: `payment`, `order`, `attempt_number`, `gateway`, `provider_reference`, `provider_transaction_id`, `amount`, `currency`, `status`, `error_message`, `raw_response`, timestamps.

### 3.4 Payment State Machine
Allowed transitions:
```text
pending  ───►  initiated  ───►  processing  ───►  paid (terminal)
   │               │                 │
   │               ├───►  failed     └───►  failed
   │               │
   ▼               ▼
cancelled       cancelled
```
- `paid` and `refunded` are terminal states. A normal callback or retry can never overwrite `paid` to `pending`, `failed`, or `cancelled`.
- Unrecognized or backward transitions raise `InvalidPaymentStateTransitionError`.

### 3.5 Authoritative Money & Payable Order Rules
- The payment amount is always read directly from `order.total`. Client-supplied amounts are ignored.
- An order is **payable** if and only if:
  1. It belongs to the authenticated customer (`order.user == request.user`).
  2. `order.payment_status != "paid"`.
  3. `order.status != "cancelled"`.
- If an order is already paid, initiation returns `409 Conflict (order_already_paid)`.
- If an order is cancelled, initiation returns `409 Conflict (order_cancelled)`.

### 3.6 Callback & Webhook Verification Sequence
1. Gateway posts callback to `/api/v1/internal/payments/webhook/{gateway}/` (or `/api/v1/payments/webhook/{gateway}/`).
2. Endpoint is public (`permissions.AllowAny`) and CSRF exempt.
3. Within `transaction.atomic()`, acquire a database row lock on the referenced `Payment` and `Order` via `select_for_update()`.
4. If `payment.status == "paid"`, immediately return idempotent `200 OK`.
5. Call `gateway.verify(payment, payload)`: backend queries SSLCommerz validation API server-to-server.
6. Verify response status is `VALID` or `VALIDATED`.
7. Authoritatively verify:
   - `verified_amount == payment.amount == order.total`
   - `verified_currency.upper() == payment.currency.upper()`
   - `tran_id` matches payment reference.
8. If validation fails or amount mismatches: transition payment attempt to `failed`, do NOT mark order as paid.
9. If validation succeeds:
   - Set `payment.status = "paid"`, `payment.paid_at = now()`, `payment.provider_transaction_id = bank_tran_id`.
   - Set `order.payment_status = "paid"`.
   - Record `OrderStatusHistory(from_status="pending", to_status="confirmed", reason="Payment verified via SSLCommerz")`.

### 3.7 Customer & CMS Interface Boundaries
- **Customer Storefront:**
  - `POST /api/v1/customer/orders/{public_id}/payment/`: Customer initiates payment and receives gateway redirect URL.
  - `GET /api/v1/customer/orders/{public_id}/payment/`: Customer retrieves authoritative payment status.
  - Storefront UI checks backend for authoritative status upon return; does not trust query parameters like `?status=success`.
- **CMS Back-Office:**
  - `GET /api/v1/admin/orders/{public_id}/`: Exposes payment details, gateway, transaction IDs, and attempt history for audit.
  - **No manual payment edit controls:** Staff cannot manually toggle `payment_status` to `"paid"`. Status transitions occur strictly through verified payment processing.

---

## 4. Consequences

### Positive
- Strict separation of commercial order creation and financial settlement.
- Robust prevention of amount tampering, replay attacks, and duplicate payments.
- Clean gateway abstraction allows seamless addition of future gateways (e.g., bKash Direct, Stripe) without altering core order or payment services.
- Operational visibility in CMS without exposing dangerous manual state manipulation.

### Negative / Trade-offs
- External network latency during server-to-server validation API calls (mitigated with strict HTTP timeouts).
- Customer must be redirected to hosted gateway page rather than embedded inline iframe (complies with ADR-0005: card data never touches Lumera servers).
