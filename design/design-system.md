# Design System Foundation

Brand feel: **premium, calm, modern skincare** — generous whitespace, soft warm neutrals, restrained accent, refined serif headings, gentle motion. Implemented in Tailwind via tokens (CSS variables mapped in `tailwind.config`); components never use raw hex values.

**Applicability:** tokens (color, type, spacing, radius, states) apply to both the storefront and the CMS. The CMS uses the same tokens with a denser, utility-first layout (data tables, filter bars, forms, confirmation dialogs) and a simpler type treatment (Inter for headings is acceptable); it is not a marketing surface. Tokens are shared by copying from this document until a shared package is justified (ADR-0007).

Values below are the *initial* proposal; they may be refined by the designer without an ADR but must be updated here.

## Color Tokens
| Token | Value | Use |
|---|---|---|
| `--color-bg` | `#FBF8F4` | Page background (warm ivory) |
| `--color-surface` | `#FFFFFF` | Cards, inputs |
| `--color-surface-muted` | `#F3EDE6` | Sections, skeletons |
| `--color-border` | `#E5DCD2` | Borders, dividers |
| `--color-text` | `#2B2623` | Primary text |
| `--color-text-muted` | `#6B615A` | Secondary text (≥4.5:1 on bg) |
| `--color-primary` | `#3F5B4B` | Deep sage — primary buttons, links |
| `--color-primary-hover` | `#334A3C` | |
| `--color-accent` | `#C48B73` | Rose clay — highlights, badges |
| `--color-success` | `#2F7A4F` | |
| `--color-warning` | `#B7791F` | |
| `--color-danger` | `#B3392F` | |
| `--color-info` | `#2F6B8F` | |
| `--color-focus` | `#3F5B4B` @ 2px ring + 2px offset | Focus indicator |

All text/background pairs must meet WCAG AA contrast. Dark mode is out of scope for v1.

## Typography
- Headings: **Cormorant Garamond** (or similar refined serif) — weights 500/600.
- Body/UI: **Inter** — 400/500/600. Loaded with `next/font` (self-hosted, no layout shift).
- Scale (rem): `xs .75`, `sm .875`, `base 1`, `lg 1.125`, `xl 1.25`, `2xl 1.5`, `3xl 1.875`, `4xl 2.25`, `5xl 3`. Body line-height 1.6; headings 1.2.
- One `<h1>` per page, proper heading hierarchy.

## Spacing & Layout
- 4px base unit; use Tailwind scale (1=4px … ). Section vertical rhythm: 64px mobile / 96px desktop.
- Container max-width 1280px, side padding 16 / 24 / 32px.
- Grid: 4 columns mobile, 8 tablet, 12 desktop; product grids 2 / 3 / 4 columns.

## Breakpoints (Tailwind defaults, mobile-first)
`sm 640`, `md 768`, `lg 1024`, `xl 1280`, `2xl 1536`. Design mobile first; touch targets ≥ 44×44px.

## Radius, Shadow, Motion
- Radius: `sm 6px` (inputs/badges), `md 10px` (buttons), `lg 16px` (cards), `full` (pills/avatars).
- Shadows: `soft` `0 1px 2px rgb(43 38 35 / .06)`; `lifted` `0 8px 24px rgb(43 38 35 / .08)` (hover cards).
- Motion: 150–250ms ease-out; respect `prefers-reduced-motion`.

## Components (specs)
**Buttons** — variants: primary (filled sage), secondary (outline), ghost, danger. Sizes: sm 36 / md 44 / lg 52px height. States: hover, focus-visible ring, disabled (50%, no pointer), loading (spinner, width preserved, `aria-busy`).

**Forms** — label above input (always visible; placeholders are not labels); 44px inputs, radius `sm`, border `--color-border`, focus ring; helper text and inline error text (danger color + icon, linked with `aria-describedby`); required indicated in text; errors from the API `details` map to fields.

**Cards** — surface background, radius `lg`, 1px border, padding 16–24px; hover `lifted` only if interactive.

**Product card** — 4:5 image (next/image, alt text from API, lazy), optional badge (New/Low stock), name (2-line clamp), category/short descriptor muted, price (server-provided string), rating if available, quick "Add to cart" secondary action. Entire card is one link; out-of-stock shows muted state + "Sold out", button disabled.

**Alerts** — inline banners: info/success/warning/danger with icon + text (never color alone), `role="alert"` for errors/`role="status"` for success; dismissible when non-critical. Toasts only for transient confirmations.

**Loading states** — skeletons matching final layout (product grid, card, detail) for content; spinner only inside buttons/small regions; avoid layout shift. Route-level `loading.tsx`.

**Empty states** — illustration/icon, one-line explanation, single clear action (e.g., empty cart → "Continue shopping"). Also for no search results (suggest clearing filters).

**Error states** — friendly message, retry action, never raw API errors; route-level `error.tsx`.

## Accessibility Baseline
WCAG 2.1 AA; semantic HTML; visible focus; keyboard operable; labels/alt text; `aria-live` for cart/async updates; form errors announced; reduced-motion respected.

## Implementation Notes
Primitives live in `frontend/src/components/ui/`; features compose primitives. No component library dependency is chosen yet — adding one (e.g., headless UI primitives) requires justification per principle 7.
