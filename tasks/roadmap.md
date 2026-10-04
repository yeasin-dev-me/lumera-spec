# Roadmap — Vertical Slices (Storefront + CMS + Backend + Integration)

Created by TASK-001-AMENDMENT. **Task numbers are suggestions** until the owner confirms the plan. Principle: CMS capability evolves **with** each business feature; it is not deferred to the end.

## Foundation (TASK-002: Completed ✅)
| Task | Domain(s) | Scope | Status |
|---|---|---|---|
| Backend scaffold | Backend | Django project, settings split, `common`, `users` skeleton (custom user), error format, health endpoint, namespace URL modules, pytest, CI hooks. **No Django Admin.** | **Done** |
| Storefront scaffold | Frontend | Next.js app, Tailwind tokens, API client, core design system primitives, live status probe | **Done** |
| CMS scaffold | CMS | Custom Next.js app, tokens, admin API client, permission guards, layout/nav shell, metric cards, live health probe | **Done** |
| Dev environment & CI | DevOps | Multi-stage Dockerfiles, Docker Compose orchestrator, Nginx reverse proxy, CI workflow, env template | **Done** |
| Contract & integration harness | Integration | Automated cross-service integration test suite (`test_integration.mjs`) covering 5/5 boundary contracts | **Done** |

## Feature Slices
Each slice = Backend API (+ permissions, audit) → Storefront UI → CMS management → Integration/E2E tests, with contract first.

| # | Slice | Backend | Storefront | CMS | Integration / E2E |
|---|---|---|---|---|---|
| 1 | **Auth & staff access** (needs ADR-0003/0009 approval) | Customer auth; CMS auth; roles/permissions; `audit` base; `super_admin` seed | Register/login/account shell | CMS login, permission-aware shell, staff & roles screens, audit log viewer | Token audience isolation, permission matrix |
| 2 | **Categories & Products** | `/storefront` + `/admin` APIs, images, discount price, stock, featured/ordering | Listing, filters, detail, SEO | Product & category management, image upload, stock | CMS → storefront propagation |
| 3 | **Content** | `content` app, storefront content API | Home sections, banners, announcement bar, pages from API | Content module (blocks, schedule, SEO) | Content publish E2E |
| 4 | **Cart & Checkout** — *Cart & Wishlist part done (Step 5, [ADR-0012](../architecture/decisions/ADR-0012-cart-and-wishlist.md)); checkout pending* | Cart ✅, wishlist ✅, server-side pricing ✅; checkout service ⏳ | Cart ✅ / wishlist ✅ UI; checkout UI, `add_to_cart`/`initiate_checkout` tracking events ⏳ | Aggregate cart/demand indicators, stock visibility ⏳ (no individual carts, BR-23) | Cart/wishlist E2E ✅; tampered-price tests (cart ✅, checkout ⏳) |
| 5 | **Orders** (needs lifecycle confirmation) | Orders, status service, history, notes | Order history/detail | Order list/search/filter/detail/status/notes, customers module (read) | Order flow E2E |
| 6 | **Payments** (blocked on ADR-0005) | Provider adapter, webhook, verification | Payment step | Payment status/reference/method, provider status, (refunds if required) | Payment test-mode E2E |
| 7 | **Integrations & Tracking** (ADR-0010) | `integrations` app, settings API, tracking service, Meta CAPI/GA4 adapters as required | `lib/tracking` providers from public config, consent | Integrations & tracking configuration UI | Event dedup/purchase event tests |
| 8 | **Promotions** (if required) | `promotions`, effective price calc | Discount display, coupon entry | Promotions module | Price recalculation tests |
| 9 | **Reviews** | Review APIs, moderation | Review display/create | Moderation queue | Review flow E2E |
| 10 | **Dashboard & Analytics** | `analytics` selectors (definitions confirmed) | — | Dashboard & analytics screens | Metrics consistency tests |
| 11 | **Production readiness** | Hardening | Perf/SEO | CMS hardening | Smoke, rollback drills; DevOps staging/prod (CMS host, Cloudflare Access) |

Slices 2–5 are the minimum to sell a product; CMS parts of each ship in the same slice.
