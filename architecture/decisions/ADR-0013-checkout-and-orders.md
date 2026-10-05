# ADR-0013: Checkout, Orders, Price Snapshots, and Transactional Inventory Deduction

- **Status:** Accepted
- **Date:** 2026-10-05
- **Deciders:** Lumera Core Architecture Team
- **Consulted:** Backend, Frontend, CMS, DevOps Leads
- **Informed:** Entire Project

---

## 1. Context and Problem Statement

Step 5 established backend-authoritative customer carts and wishlists. In the cart, line prices and availability are dynamically derived from live `Product` catalog data on every read.

Step 6 introduces checkout and persistent customer orders. An order represents an immutable legal and commercial historical agreement between customer and merchant. 

The architecture must solve:
1. How to validate and snapshot purchase-time pricing, product metadata, and shipping information so that subsequent catalog or user address changes never alter existing orders.
2. How to ensure transactional order creation, row locking, and inventory deduction to prevent overselling under concurrent checkouts.
3. How to establish the order lifecycle and payment status foundation so that Step 7 Payments can integrate cleanly without breaking changes.
4. How to enforce customer isolation and operational CMS visibility.

---

## 2. Decision Drivers

- **Authoritative Integrity (BR-01, BR-03):** Prices, totals, discounts, and inventory deductions are computed and validated exclusively by the backend. Client-supplied monetary or total values are strictly ignored.
- **Historical Immutability (BR-03):** An order is a snapshot in time. Changing product prices, names, SKUs, or customer profile addresses must never retroactively change historical orders.
- **Transactional Atomicity (BR-08, BR-11):** Cart lock, product lock, stock validation, order creation, order items creation, inventory deduction, and cart clearing must execute in a single database transaction. Partial orders or uncommitted stock deductions are forbidden.
- **Concurrency Safety (BR-08):** Row locking (`select_for_update`) must prevent race conditions and overselling when multiple customers attempt to purchase the same inventory simultaneously.
- **Future Payment Decoupling (BR-12, BR-12a):** Payments will be implemented in Step 7. Orders created in Step 6 initialize with `status = "pending"` and `payment_status = "pending"`.

---

## 3. Decision

### 3.1 Domain Models
We introduce three domain models in `apps.orders`:
1. **`Order`**:
   - `public_id`: UUID4 for external URL and API references ([ADR-0004](ADR-0004-primary-keys.md)).
   - `order_number`: Unique human-readable identifier (format: `LUM-YYYYMMDD-XXXXXX`).
   - `user`: ForeignKey to `settings.AUTH_USER_MODEL` (`on_delete=models.PROTECT`).
   - `status`: Order fulfillment lifecycle (`pending`, `confirmed`, `processing`, `shipped`, `delivered`, `cancelled`).
   - `payment_status`: Payment state (`pending`, `paid`, `failed`, `refunded`).
   - `subtotal`, `shipping_cost`, `discount_total`, `total`: Backend-calculated `NUMERIC(12,2)`.
   - `currency`: Default `"USD"`.
   - Snapshotted shipping fields: `shipping_full_name`, `shipping_phone`, `shipping_address_line1`, `shipping_address_line2`, `shipping_city`, `shipping_area`, `shipping_postal_code`, `shipping_country`.
   - `customer_notes`: Optional customer note.
   - `created_at`, `updated_at`: Timestamps.

2. **`OrderItem`**:
   - `order`: ForeignKey to `Order` (`on_delete=models.CASCADE`).
   - `product`: Nullable ForeignKey to `products.Product` (`on_delete=models.PROTECT`).
   - `product_name`: Immutable snapshot of product name.
   - `product_sku`: Immutable snapshot of product SKU.
   - `product_image`: Immutable snapshot of primary image URL.
   - `unit_price`: Immutable snapshot of effective purchase price (`NUMERIC(12,2)`).
   - `quantity`: Positive integer.
   - `line_total`: `unit_price * quantity` (`NUMERIC(12,2)`).

3. **`OrderStatusHistory`**:
   - Append-only audit log of order status transitions (`order`, `from_status`, `to_status`, `changed_by`, `reason`, `created_at`).

### 3.2 Transactional Checkout Sequence
The order creation service executes inside a single `transaction.atomic()` block:
1. Lock customer's `Cart` via `select_for_update()`.
2. Retrieve and lock all cart items. Verify cart is not empty.
3. Lock all referenced `Product` records via `select_for_update()`.
4. Validate each product:
   - Must be storefront-eligible (`status = "active"`, `is_visible = True`, category active).
   - `stock_quantity >= item.quantity`. If not, abort with `409 insufficient_stock`.
5. Compute line totals and order totals using exact `Decimal` arithmetic.
6. Create `Order` with snapshotted shipping address and generated unique `order_number`.
7. Create `OrderItem` records with snapshot fields.
8. Deduct stock: `product.stock_quantity = product.stock_quantity - item.quantity`.
9. Record initial `OrderStatusHistory` entry (`pending`).
10. Clear cart items.

If any validation fails or an exception occurs, the transaction rolls back cleanly.

### 3.3 API Architecture
- **Checkout Preview:** `GET /api/v1/customer/checkout/` returns the authoritative preview of cart items, stock status, subtotal, and shipping requirements.
- **Order Placement:** `POST /api/v1/customer/orders/` accepts shipping address and customer notes; derives items and quantities solely from the server-side cart.
- **Customer Orders:**
  - `GET /api/v1/customer/orders/`: Paginated order list for the authenticated customer.
  - `GET /api/v1/customer/orders/{public_id}/`: Order detail with item snapshots and shipping address.
- **CMS Order Visibility:**
  - `GET /api/v1/admin/orders/`: Paginated order list guarded by `orders.read` permission.
  - `GET /api/v1/admin/orders/{public_id}/`: Full order detail for back-office staff.

---

## 4. Consequences

### Positive
- Strict protection against concurrent overselling.
- Complete price, product, and address snapshot immutability.
- Safe customer isolation; Customers cannot view or infer other customers' orders.
- Direct alignment with Step 7 Payments milestone.

### Negative / Trade-offs
- Cart clearing is coupled with order creation; if a payment failure occurs in Step 7, an order will exist in `payment_status="failed"` rather than reverting the cart immediately (or cart re-hydration service will be introduced in Step 7).
