# Business Rules

Rules marked **(assumed)** are placeholders pending product-owner confirmation (see requirements Open Questions). They are the minimum implied by the architectural principles.

## Pricing & Money
- BR-01 Prices, totals, discounts, and shipping are computed **only by the backend**. Client values are ignored.
- BR-02 Money is stored as `DECIMAL(12,2)`; never floats. One currency system-wide **(assumed)**.
- BR-02a Products have a regular `price` and an optional `discount_price` (< price). The **effective price** is computed by the backend from `discount_price`, active promotions, and coupons; CMS and storefront only display it.
- BR-02b Any CMS price/discount edit is validated and audit-logged; it affects future carts/orders only (BR-03).
- BR-03 Order items snapshot product name, SKU, unit price, primary image, and shipping information at purchase time; later catalog changes or customer profile edits never alter existing orders ([ADR-0013](architecture/decisions/ADR-0013-checkout-and-orders.md)).

## Catalog
- BR-04 Only products with `status = active`, `is_visible = true`, and an active category are visible/purchasable by customers. Draft/archived items are invisible in storefront APIs.
- BR-05 Products/categories are never hard-deleted once referenced by an order; CMS "delete" = **archive**. A category containing active products cannot be archived without reassigning them.
- BR-06 SKU is unique.

## Inventory
- BR-07 Stock is validated by the backend at cart update and again at order placement.
- BR-08 Stock decrement happens inside the order-placement transaction with row locking (`select_for_update`); overselling is strictly prevented ([ADR-0013](architecture/decisions/ADR-0013-checkout-and-orders.md)).

## Cart
- BR-09 A customer has at most one active cart. Cart quantities must be positive integers **(max per line: assumed 20)**.
- BR-10 Cart displays current prices; the price is only locked when an order is placed.
- BR-10a Adding a product already in the cart adds to the existing line quantity; the resulting quantity is validated against BR-09 and current stock (`409 insufficient_stock`). One line per product. ([ADR-0012](architecture/decisions/ADR-0012-cart-and-wishlist.md))
- BR-10b Only BR-04-eligible products can be newly added to cart or wishlist. Lines whose product later becomes ineligible or short on stock stay visible with a backend-computed availability status and are excluded from the subtotal; their quantity cannot be increased (`409 product_unavailable`) but they can always be removed.
- BR-10c Cart and wishlist ownership is derived from the authenticated session only; no client-supplied owner identifier is accepted.

## Wishlist
- BR-10d A product appears at most once per customer's wishlist (duplicate add → `409 conflict`). A wishlist does not reserve stock, lock prices, or create orders.

## Orders
- BR-11 Orders are created only by the backend from the server-side cart, never from client-supplied line prices or quantities. Upon successful order creation, stock is deducted and the customer's cart is atomically cleared inside the same database transaction ([ADR-0013](architecture/decisions/ADR-0013-checkout-and-orders.md)).
- BR-12 **Order lifecycle:** `pending → confirmed → processing → shipped → delivered`. Cancellation: `cancelled` from `pending`/`confirmed`/`processing` (stock restored). Returns are separate delivered-order records and never mutate the order lifecycle. Initial order creation places orders into `pending`. Transitions are enforced by backend services; every transition writes `OrderStatusHistory`. Invalid transitions → `409 invalid_state_transition`.
- BR-12a **Payment status is separate** from order status: `pending / paid / failed / refunded`. Initial order creation sets payment status to `pending`.
- BR-13 Customers can only view their own orders. Internal notes are never visible to customers.
- BR-13a Saved addresses are customer-owned mutable convenience records. Order shipping fields are copied at order creation and are immutable historical snapshots.
- BR-13b Only delivered orders are eligible for returns. Requested quantities cannot exceed the remaining eligible quantity across active return requests.

## Payments
- BR-14 A payment becomes `paid` (and the order's payment status with it) only after **server-side verification** with the payment provider (webhook or server-to-server check). Client "success" redirects, and CMS users, cannot mark a provider payment as paid. Manual payment recording (e.g. offline methods) is only via an explicit permissioned, audited admin action, if the business requires it.
- BR-15 Payment processing is idempotent; duplicate webhooks must not double-apply.
- BR-15a Refund foundations reference an authoritative paid payment. Amounts are positive and cannot exceed the remaining refundable amount under concurrent requests. Internal refund records do not mean gateway settlement and do not set payment status to refunded.

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
- BR-24a Editorial content, banners, and promotional campaigns cannot alter product pricing, inventory stock, order calculations, or financial settlement states. Commercial rules remain backend-authoritative.
- BR-24b Content publication scheduling (`draft`, `scheduled`, `published`, `archived`) is dynamically evaluated server-side; draft, scheduled-future, expired, and archived content blocks are strictly excluded from public storefront APIs.
- BR-24c Next.js dynamic SEO metadata, canonical URLs, and structured JSON-LD schemas derive exclusively from authoritative backend models; private routes, carts, checkouts, and unpublished entities are strictly excluded from indexing and sitemaps.
- BR-25 Coupons/discount rules (when introduced) have validity windows and usage limits enforced at order placement, atomically.
- BR-26 Integration secrets are write-only and never returned or logged; public configuration is a whitelist (ADR-0010). Changes to integration/payment/tracking configuration require `integrations.manage` and are audited.
- BR-27 Tracking: **purchase** events are sent by the backend from verified order data; client-sent tracking events are non-authoritative and never alter business state. Tracking failures never block orders.
- BR-28 Audited actions: price/discount, stock, product publish/archive, order status/notes, refunds, integration/tracking config, role/permission changes, reveal of masked data. Audit entries are append-only.
- BR-29 Dashboard metrics are computed by the backend. Gross revenue uses authoritative paid payment completion data; externally settled refunds are reported separately and only those reduce net revenue. Internal refund foundations are operational records, not settlement.
- BR-30 Reporting is protected by `reports.read`. Order, payment, return, refund, shipment, and delivery metrics use their documented source timestamps and inclusive-start/exclusive-end date ranges.
- BR-31 Transactional communication is strictly separated from marketing communication ([ADR-0018](architecture/decisions/ADR-0018-customer-communication-notification-architecture.md)). Operational messages (order confirmation, payment receipt, shipping tracking, delivery, cancellation, return requests, refunds, account welcome) are required for service execution and are never suppressed by marketing unsubscribes. Marketing communications strictly require customer opt-in consent (`marketing_email = true`). Notification delivery is failure-isolated: provider or delivery exceptions never abort or roll back commerce transactions. Notification dispatch is idempotent across domain events. CMS operational visibility requires `notifications.read`, and manual resend requires `notifications.manage`.

