# ADR-0012: Customer cart & wishlist model, pricing and ownership
- Status: **Proposed** — implemented under the Step 5 (Cart & Wishlist) task brief; pending formal owner sign-off like ADR-0003/0004/0008
- Date: 2026-10-05
- Deciders: Product owner (pending), engineering

## Context
Step 5 introduces a customer cart and a wishlist. The spec already fixes several points:
- BR-09 one active cart per customer, positive integer quantities (max 20 per line, *assumed*).
- BR-10 the cart shows **current** prices; the price is only locked when an order is placed. BR-03 order items snapshot name/SKU/unit price.
- BR-07 stock is validated at cart update and again at order placement; BR-08 stock is decremented only inside the order transaction.
- `database.md` defines `Cart (user one-to-one)` and `CartItem (cart, product, quantity)` with `UNIQUE(cart, product)` and `CHECK quantity > 0`; wishlists are listed as *not yet designed*.
- ADR-0008/`api-architecture.md` place the cart under `/customer/cart/` (singleton per user).
- ADR-0004 exposes products by `slug` publicly, while `Product.public_id` (UUID) already exists and is returned by every product serializer.

Open points this ADR settles: price snapshot vs. dynamic pricing, wishlist placement/model, the product identifier in cart/wishlist URLs, FK delete behaviour, and how ineligible products in an existing cart are represented.

## Decision
1. **Dynamic pricing (no snapshot in the cart).** `CartItem` stores only `product` + `quantity`. On every read the backend recomputes `unit_price = Product.effective_price`, `line_total = unit_price × quantity` (Python `Decimal`, quantized to 0.01, `ROUND_HALF_UP`), `subtotal = Σ line_total` of purchasable lines, `total = subtotal` (shipping/tax/discounts arrive with checkout). The snapshot happens only in `OrderItem` at order placement (BR-03). This keeps the future Orders slice the single place where prices are locked.
2. **Stock is validated, never reserved.** Add/update reject a resulting quantity above `stock_quantity` with `409 insufficient_stock`; quantity bounds (`1..20`) are `400 validation_error`. Reads re-evaluate each line and report `availability_status` (`available | insufficient_stock | out_of_stock | unavailable`); only `available` lines count toward `subtotal`. Lines are never silently mutated or deleted on read.
3. **Add = merge.** Adding a product already in the cart adds the requested quantity to the existing line and validates the resulting total; `UNIQUE(cart, product)` guarantees one line per product. The add path locks the cart row (`select_for_update`) inside `transaction.atomic` so concurrent adds cannot lose increments.
4. **Eligibility follows BR-04.** Only `status=active`, `is_visible=true`, active-category products can be newly added to cart or wishlist; anything else is indistinguishable from a missing product (`404 not_found`), matching the storefront APIs. Changing the quantity of a line whose product has since become ineligible returns `409 product_unavailable`; removing it is always allowed.
5. **Identifier:** cart/wishlist URLs and request bodies use the product `public_id` (UUID), e.g. `PATCH /customer/cart/items/{product_id}/`. It is stable across slug edits and already public. No internal integer ids appear in cart/wishlist responses. Cart and wishlist themselves have no client-visible id (implicit singleton per customer).
6. **Ownership** is derived exclusively from the authenticated request (`request.user`). No endpoint accepts `customer_id`/`user_id`/`owner_id`; every query is scoped to the caller's cart/wishlist, so another customer's data is unreachable (not merely forbidden).
7. **Wishlist lives in the existing `cart` app** (no new Django app, so the ADR-0002 app list is unchanged): `Wishlist (user one-to-one)` + `WishlistItem (wishlist, product)` with `UNIQUE(wishlist, product)`. It mirrors the Cart shape for consistency. Wishlist does not reserve stock, lock prices, or affect orders. Adding a duplicate returns `409 conflict` (wishlist unchanged); removal is idempotent (`204`).
8. **Delete behaviour:** `CartItem.cart`, `WishlistItem.wishlist` → `CASCADE` (pure children); `Cart.user`, `Wishlist.user` → `CASCADE` (users are deactivated, not deleted); `CartItem.product`, `WishlistItem.product` → `CASCADE`. Carts/wishlists are ephemeral intent, not historical records, so they must never block a permitted hard delete of a never-ordered product (BR-05 already routes CMS "delete" to archive).
9. **Responses:** mutating cart endpoints return the full recalculated `Cart` (201 when a new line is created, 200 otherwise) so the storefront never computes totals; deletes return `204`. Wishlist add returns the full `Wishlist`.
10. **No CMS surface.** Individual carts/wishlists are not exposed to the CMS (BR-23, cms-architecture). Aggregate demand indicators are deferred.

## Alternatives Considered
- **Price snapshot on `CartItem`** — conflicts with BR-10, needs reconciliation logic when prices change, and duplicates the order snapshot. Rejected.
- **Separate `wishlist` app** — cleaner name but changes the accepted ADR-0002 app list for two small models. Rejected for now; can be extracted later without API changes.
- **Flat `WishlistItem(user, product)` without a parent** — simpler, but diverges from the Cart shape. Rejected for consistency.
- **Slug as the product identifier** — slugs are editable in the CMS, which would break in-flight client requests. Rejected in favour of `public_id`.
- **Idempotent 200 on duplicate wishlist add** — friendlier, but `api-architecture.md` assigns duplicates to `409`. Rejected; clients treat `409 conflict` as "already saved".

## Consequences
- The Orders slice must read the server-side cart, re-validate stock under row locks (BR-08), and snapshot prices into `OrderItem`; the cart never carries authoritative historical prices.
- Displayed cart totals can change between visits when CMS prices change; the UI shows current backend values only.
- `database.md` gains `Wishlist`/`WishlistItem`; the contract gains `/customer/cart/*` and `/customer/wishlist/*`.
- Guest carts and cart merge on login remain an open question (database.md) and are out of scope.
- Token audience separation (ADR-0003/0009) is still pending; until then any authenticated user, including staff, owns a personal cart scoped to their own account.
