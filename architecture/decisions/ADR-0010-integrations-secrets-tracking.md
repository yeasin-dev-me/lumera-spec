# ADR-0010: Integrations, secret configuration, and tracking architecture
- Status: **Proposed — needs approval**
- Date: 2026-10-04

## Context
The CMS must configure external services (payment, shipping, SMS, email, analytics, marketing) and tracking (Meta Pixel/CAPI, Google). Secrets must never reach browsers; provider logic must not be scattered through the storefront.

## Decision (proposed)
### Integration configuration
- Backend `integrations` app stores `IntegrationSetting` records per provider/key with a `visibility`: `public` (safe for browsers) or `secret` (server-only).
- Public settings are exposed only via a **whitelisted** `GET /storefront/config/` (and fixed per-provider keys such as `meta.pixel_id`, `ga4.measurement_id`). Secret settings are **write-only** through the admin API (`is_set` flag only), encrypted at rest with an application key from the environment (adds the `cryptography` dependency — approval implied by this ADR).
- Bootstrap/infra secrets (DB credentials, Django secret key, encryption key, and — **recommended default** — payment provider secrets and webhook secrets) are environment/secret-store only; the CMS shows only their *status* (configured/healthy). Whether payment secrets become CMS-editable is an **open question** (see ADR-0005, requirements).
- Providers sit behind small adapter functions in `integrations/providers/<name>.py`; add an abstraction only when ≥2 providers of the same kind exist.

### Tracking
- Storefront has **one** tracking module (`lib/tracking`) exposing `track(eventName, payload)`; components never call provider SDKs directly. The module loads provider scripts according to the public config and consent state.
- Provider-neutral event catalog: `view_content`, `view_category`, `search`, `add_to_cart`, `remove_from_cart`, `initiate_checkout`, `add_payment_info`, `purchase`, `sign_up`. Each event has a client-generated `event_id` (UUID) for **browser/server deduplication**.
- Server-side: the backend `integrations.tracking` service receives events and dispatches to configured providers (Meta Conversions API, GA4 Measurement Protocol, …) asynchronously/best-effort. Hashing of personal identifiers (email, phone) is done server-side per provider rules.
- **Purchase** events are emitted **by the backend** after verified payment (authoritative values from the order, never from the client). Other events may be sent by the storefront to `POST /storefront/tracking/events/` (validated, rate-limited, no business-critical trust).
- Failures never block checkout/orders; delivery attempts are logged (no secrets/PII) and retried with a bounded policy. Whether a task queue is needed is decided when implementing (default: `transaction.on_commit` + simple retry; a queue requires its own ADR).
- Consent: tracking respects cookie-consent state; regional requirements are an open question.

## Alternatives Considered
Provider scripts/logic inline in components (rejected); Google Tag Manager only (possible later; doesn't cover server-side events); secrets only in env (safest but no CMS management — kept as the default for payments).

## Consequences
Adds an `integrations` app (ADR-0011), an encryption key to the secret inventory, and tests that assert secrets never appear in any response or log.
