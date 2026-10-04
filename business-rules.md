# Business Rules

Rules marked **(assumed)** are placeholders pending product-owner confirmation (see requirements Open Questions). They are the minimum implied by the architectural principles.

## Pricing & Money
- BR-01 Prices, totals, discounts, and shipping are computed **only by the backend**. Client values are ignored.
- BR-02 Money is stored as `DECIMAL(12,2)`; never floats. One currency system-wide **(assumed)**.
- BR-02a Products have a regular `price` and an optional `discount_price` (< price). The **effective price** is computed by the backend from `discount_price`, active promotions, and coupons; CMS and storefront only display it.
- BR-02b Any CMS price/discount edit is validated and audit-logged; it affects future carts/orders only (BR-03).
- BR-03 Order items snapshot product name, SKU, and unit price at purchase time; later catalog changes never alter existing orders.

## Catalog
- BR-04 Only products with `status = active`, `is_visible = true`, and an active category are visible/purchasable by customers. Draft/archived items are invisible in storefront APIs.
- BR-05 Products/categories are never hard-deleted once referenced by an order; CMS "delete" = **archive**. A category containing active products cannot be archived without reassigning them.
- BR-06 SKU is unique.

## Inventory
- BR-07 Stock is validated by the backend at cart update and again at order placement.
- BR-08 Stock decrement happens inside the order-placement transaction with row locking; overselling is not allowed.

## Cart
- BR-09 A customer has at most one active cart. Cart quantities must be positive integers **(max per line: assumed 20)**.
- BR-10 Cart displays current prices; the price is only locked when an order is placed.

## Orders
- BR-11 Orders are created only by the backend from the server-side cart, never from client-supplied line prices.
- BR-12 **Order lifecycle (PROPOSED — needs business confirmation, see Open Questions):** `pending → confirmed → processing → shipped → delivered`. Cancellation: `cancelled` from `pending`/`confirmed`/`processing` (stock restored). Returns/refunds (`return_requested`, `returned`, refund tracking on Payment) are **not defined yet** and will be specified in the orders task. Transitions are enforced by one backend service used by both customer (cancel) and CMS (staff) paths; every transition writes `OrderStatusHistory` and an audit entry (when staff-initiated). Invalid transitions → `409 invalid_state_transition`.
- BR-12a **Payment status is separate** from order status: `pending / paid / failed / refunded / partially_refunded`. Whether `confirmed` requires `paid` (e.g. cash-on-delivery vs. prepaid) is an open question.
- BR-13 Customers can only view their own orders. Internal notes are never visible to customers.

## Payments
- BR-14 A payment becomes `paid` (and the order's payment status with it) only after **server-side verification** with the payment provider (webhook or server-to-server check). Client "success" redirects, and CMS users, cannot mark a provider payment as paid. Manual payment recording (e.g. offline methods) is only via an explicit permissioned, audited admin action, if the business requires it.
- BR-15 Payment processing is idempotent; duplicate webhooks must not double-apply.

## Reviews
- BR-16 One review per customer per product. Rating is an integer 1–5.
- BR-17 Only customers who purchased the product may review it **(assumed)**.
- BR-18 Unapproved/hidden reviews are not public; moderation policy TBD.

## Accounts & Access
- BR-19 Email is unique (case-insensitive) and is the login identifier.
- BR-20 Access is **permission-based** (`resource.action`, e.g. `orders.update`); roles are named sets of permissions managed as data. `customer` has no CMS permissions; CMS access requires `is_staff` + an assigned role. Only `super_admin` exists by default. Permissions are enforced server-side on every admin request.

## CMS / Operations
- BR-21 The CMS has no business authority: every CMS write goes through the same backend services and validation as any other caller.
- BR-22 Stock edits via CMS are explicit adjustments (`inventory.update`), validated (≥ 0), and audit-logged with before/after.
- BR-23 Admin-visible customer data follows least privilege: `customers.read` shows name/email/phone/status/order summary; no passwords/tokens/payment instruments. Individual carts are not visible to the CMS; only aggregate demand indicators.
- BR-24 Content blocks follow `draft → published` with optional schedule; storefront shows only published, in-window content. Content cannot embed scripts.
- BR-25 Coupons/discount rules (when introduced) have validity windows and usage limits enforced at order placement, atomically.
- BR-26 Integration secrets are write-only and never returned or logged; public configuration is a whitelist (ADR-0010). Changes to integration/payment/tracking configuration require `integrations.manage` and are audited.
- BR-27 Tracking: **purchase** events are sent by the backend from verified order data; client-sent tracking events are non-authoritative and never alter business state. Tracking failures never block orders.
- BR-28 Audited actions: price/discount, stock, product publish/archive, order status/notes, refunds, integration/tracking config, role/permission changes, reveal of masked data. Audit entries are append-only.
- BR-29 Dashboard metrics are computed by the backend. Metric definitions (e.g. "revenue" = sum of `total` of orders with payment `paid`, excluding cancelled/refunded) must be confirmed by the business before implementation.
