# ADR-0017: Content, SEO, and Marketing Architecture
- Status: Accepted
- Date: 2026-10-07
- Deciders: Lumera product and engineering owners

## Context

Step 10 establishes the content management, search engine optimization (SEO), and marketing foundations for Lumera. While the platform already possesses an authoritative commerce engine (catalog, inventory, cart, checkout, orders, payments, fulfillment, and refunds), it lacks:
1. CMS-managed editorial and promotional content (heroes, announcements, promotional banners, content sections).
2. Dynamic publication lifecycles (draft, scheduled, published, archived) with dynamic server-side schedule evaluation.
3. Server-side SEO foundations in the Next.js Storefront (dynamic metadata, canonical URLs, robots.txt, dynamic XML sitemap, and structured JSON-LD).
4. A vendor-neutral marketing and tracking abstraction boundary.

## Core Architectural Principle: Content vs Commerce Separation

A fundamental architectural boundary is strictly enforced:
- **Commerce** (Products, pricing, stock, cart, checkout, payments, fulfillment, refunds) remains strictly backend-authoritative in PostgreSQL.
- **Content & Marketing** (Banners, promotional copy, hero sections, campaigns) describes and organizes promotional messaging.
- CMS content can reference products and categories via stable identifiers, but **cannot mutate commercial state**. For example, a CMS banner stating "20% off botanical oils" is purely promotional copy; actual product pricing and discount calculations remain governed exclusively by the backend pricing engine.

## Decisions

### 1. Reusable Content & Campaign Domain Models

A dedicated `apps.content` Django application is introduced:
- **`Campaign`**: An operational grouping for promotional efforts (e.g. "Summer Botanicals"), containing `name`, `slug`, `badge_text`, `status`, and publication dates (`start_at`, `end_at`).
- **`ContentBlock`**: A polymorphic content model supporting multiple slots:
  - `content_type`: `hero`, `banner`, `announcement`, `promotional_section`, `content_section`.
  - Fields: `public_id`, `title`, `slug`, `body`, `image_url`, `alt_text`, `cta_label`, `cta_url`, `sort_order`.
  - Entity references: optional foreign keys to `Product` (`linked_product`) and `Category` (`linked_category`), and `Campaign` (`campaign`).
  - Dates: `start_at` and `end_at` for scheduled publication.

### 2. Publication Lifecycle & Dynamic Scheduling

Content lifecycle states:
```text
draft -> published -> archived
draft -> scheduled -> published -> archived
```

- **Draft**: Visible only to authenticated staff via CMS admin endpoints.
- **Published**: Publicly visible if within the active date window.
- **Scheduled**: Evaluated dynamically by backend queries without requiring background workers (Celery/Redis):
  - A block is active when `status IN ('published', 'scheduled')` AND (`start_at IS NULL OR start_at <= NOW()`) AND (`end_at IS NULL OR end_at > NOW()`).
- **Archived / Expired**: Excluded from all public storefront endpoints.

### 3. Content Security & Input Sanitization

- CMS content fields (`title`, `body`, `cta_label`, etc.) are plain text or structured copy.
- Arbitrary executable script injection (`<script>`, inline JS handlers) is rejected and sanitized server-side.
- Next.js renders text and images through standard React JSX and components, eliminating `dangerouslySetInnerHTML`.

### 4. API Endpoints & RBAC

- **Public Storefront APIs**:
  - `GET /api/v1/content/home/`: Returns active hero, announcement, banners, and promotional sections for the storefront homepage.
  - `GET /api/v1/content/banners/`: Returns active announcements and promotional banners.
  - `GET /api/v1/content/{slug}/`: Returns a specific published content block.
  - `GET /api/v1/storefront/config/`: Public configuration including whitelisted tracking keys and site metadata.
- **CMS Admin APIs**:
  - `GET, POST /api/v1/admin/content/`
  - `GET, PATCH, DELETE /api/v1/admin/content/{public_id}/`
  - `POST /api/v1/admin/content/{public_id}/publish/`
  - `POST /api/v1/admin/content/{public_id}/unpublish/`
  - `POST /api/v1/admin/content/{public_id}/archive/`
  - `GET, POST /api/v1/admin/campaigns/`
  - `GET, PATCH /api/v1/admin/campaigns/{public_id}/`
- **Permissions**:
  - `content.read`: View content blocks and campaigns in the CMS.
  - `content.manage`: Create, update, publish, unpublish, and archive content.

### 5. SEO & Metadata Architecture

Implemented natively in Next.js Storefront:
- **Dynamic Metadata**: `generateMetadata()` on product, category, and content routes using backend data (`seo_title`, `seo_description`).
- **Canonical URLs**: Generated deterministically using `src/lib/seo/canonical.ts`, ignoring query parameters and session state.
- **Structured Data (JSON-LD)**: Schema.org `Product` JSON-LD rendered on product pages using backend-authoritative data (`effective_price`, SKU, stock availability).
- **robots.txt (`src/app/robots.ts`)**: Allows public storefront catalog routes; disallows private routes (`/account/*`, `/checkout/*`, `/cart`, `/api/*`).
- **XML Sitemap (`src/app/sitemap.ts`)**: Dynamically indexes active categories and products, omitting draft, archived, or customer pages.

### 6. Marketing & Tracking Integration Boundary

A decoupled tracking abstraction is established in `src/lib/tracking/`:
- Exposes a unified `track(event, payload)` function.
- Provider-neutral event catalog: `page_view`, `view_item`, `view_category`, `search`, `add_to_cart`, `begin_checkout`, `purchase`.
- Configured strictly via public environment variables (`NEXT_PUBLIC_GA4_ID`, `NEXT_PUBLIC_META_PIXEL_ID`) and public `/storefront/config/`.
- **Privacy & Security**: Zero transmission of customer passwords, tokens, or financial payment details.
- Failures in tracking dispatch never block or degrade commercial storefront operations.

## Alternatives Considered

1. **Third-party Headless CMS (Contentful/Sanity)**: Rejected to preserve full multi-repo self-contained sovereignty and unified RBAC.
2. **HTML / Rich Text WYSIWYG**: Rejected to prevent XSS vulnerabilities and maintain strict design system aesthetic consistency.
3. **Celery Worker for Publishing**: Rejected because dynamic database query filtering handles scheduled time boundaries with zero infrastructure overhead.
4. **Client-side Price Deductions in Banners**: Explicitly rejected per core architectural principle.

## Consequences

- Clean separation between editorial marketing content and commercial pricing/inventory.
- Fully automated SEO compliance (robots, sitemap, OpenGraph, JSON-LD) driven by backend models.
- Production-ready foundation for future customer communication and marketing automation.
