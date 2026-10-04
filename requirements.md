# Requirements

High-level scope only. Detailed requirements are added per task; nothing here is invented beyond the product brief. Open items are in "Open Questions".

## Functional Areas (from product brief)
| ID | Area | Summary |
|---|---|---|
| FR-AUTH | Customer authentication | Register, login, logout, session refresh |
| FR-CAT | Catalog & categories | Browse/search/filter/sort products by category |
| FR-CART | Cart | Add/update/remove items; persists for logged-in customers |
| FR-CHK | Checkout | Address selection, order review, order placement |
| FR-ORD | Orders | Order history and details for customers; management by staff |
| FR-PAY | Payments | Pay for an order; server-side payment verification |
| FR-REV | Reviews | Customers review purchased products |
| FR-ADM | Admin operations | Delivered through the custom **CMS** (see FR-CMS-*); Django Admin is **not** used |
| FR-SEO | Storefront | SEO-friendly, accessible, responsive |

## CMS Requirements (TASK-001-AMENDMENT)
The custom Next.js CMS ([cms-architecture](architecture/cms-architecture.md)) is a first-class system. Capabilities are delivered incrementally **with** their backend feature.
| ID | Area | Summary |
|---|---|---|
| FR-CMS-PROD | Products | Create/edit/archive; name, description, images, price, discount price, stock, SKU, status, visibility, categories, metadata, featured, ordering; attributes only if required |
| FR-CMS-CAT | Categories | Create/edit/archive; image, description, slug, ordering, visibility; flat unless hierarchy is justified |
| FR-CMS-ORD | Orders | List/search/filter/detail; items, quantities, prices, discounts, totals, shipping info, status update, payment status, role-appropriate payment info, internal notes |
| FR-CMS-CUST | Customers | List/search/profile/order history/account status/relevant addresses (policy-limited) |
| FR-CMS-OPS | Product operations visibility | Availability, stock, order quantities, aggregate cart/demand indicators; no individual private carts |
| FR-CMS-CONT | Content | Homepage sections, hero/promo banners, featured products/categories, announcement bar, static pages, SEO metadata, navigation content — structured blocks, **not** a page builder |
| FR-CMS-PROMO | Pricing & promotions | Product/discount price; coupons, discount rules, campaigns where required; backend recalculates |
| FR-CMS-PAY | Payments | Payment status, references, method, provider status, verification results, configuration; secrets never in browser; flow CMS → Backend → Provider |
| FR-CMS-INT | Integrations | Configure payment/shipping/SMS/email/analytics/marketing; public vs secret configuration |
| FR-CMS-TRK | Tracking | Pixel/analytics configuration; browser + server-side events (view content, add to cart, initiate checkout, purchase, …) via centralized architecture (ADR-0010) |
| FR-CMS-AUTHZ | Staff access | Separate CMS authentication; granular permission-based RBAC enforced by backend; staff & role management |
| FR-CMS-AUDIT | Auditability | Append-only audit log of important admin actions |
| FR-CMS-DASH | Dashboard | Total/pending orders, revenue, product count, low stock, recent orders, top products, sales trends, customer and marketing metrics — backend-computed |

## Non-Functional Requirements
| ID | Requirement |
|---|---|
| NFR-SEC | Server-side validation of all business-critical values; see [security](security/security.md) |
| NFR-A11Y | Storefront targets WCAG 2.1 AA |
| NFR-PERF | Fast storefront loads (SSR/ISR for catalog pages); paginated APIs. Numeric budgets to be set when baseline exists |
| NFR-REPRO | Deployments reproducible from source + config |
| NFR-TEST | Every feature ships with automated tests |
| NFR-OBS | Health checks, logging, monitoring in staging/production |

## Open Questions (need product owner)
1. Currency: single currency or multi-currency? (Assumed: single, configurable; see business-rules.)
2. Payment provider/gateways (not selected — see ADR-0005).
3. Guest checkout allowed? (Assumed: **no** until decided.)
4. Shipping: rates, zones, tax handling.
5. Review policy: verified-purchase only? moderation?
6. Email provider for transactional mail.
7. **Order lifecycle** to confirm (proposed in BR-12), plus cancellation/return/refund states and whether `confirmed` depends on payment.
8. Which CMS roles the business actually needs (only `super_admin` seeded by default); whether staff MFA is mandatory.
9. Which customer fields operators may see by default vs. via audited "reveal".
10. Which integrations/trackers are required at launch (Meta Pixel/CAPI, GA4, shipping, SMS) and cookie-consent/regional privacy requirements.
11. Whether payment secrets may be CMS-managed or must stay environment-only (recommended: environment-only).
12. Coupons/discount rules/campaigns: needed at launch, and which rule types?
13. Dashboard metric definitions (revenue, top products, trends periods).
14. Product attributes/variants and category hierarchy: needed? (Not designed until confirmed.)
15. Confirm CMS as a separate repo/app (ADR-0007) and the API namespace scheme (ADR-0008).
