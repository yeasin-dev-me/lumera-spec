# ADR-0018: Customer Communication & Notification Architecture

- Status: Accepted
- Date: 2026-10-07
- Deciders: Lumera product and engineering owners

## Context

Step 11 introduces production-quality customer communication and notification infrastructure.
Prior to this step:
- Step 8 introduced an initial `Notification` model and fail-safe dispatch for order lifecycle transitions.
- Step 9 introduced customer profiles, delivered-order return requests, and internal refund foundations.
- Step 10 established content and marketing foundations with dynamic scheduling and strict isolation from commerce totals.

A robust communication system is needed to:
1. Deliver operational transactional messages reliably across order placement, payment, fulfillment, returns, and refunds.
2. Separate transactional communication from marketing campaigns and consent-driven messaging.
3. Isolate notification failures so commerce operations (orders, payments, fulfillment) never roll back or fail because of email delivery.
4. Provide customer communication preference management without allowing operational order updates to be silently suppressed.
5. Provide operational delivery auditability and visibility in the CMS with role-based permissions (`notifications.read`, `notifications.manage`).

## Decision

### 1. Transactional vs Marketing Communication Boundary

We strictly separate **Transactional Communication** from **Marketing Communication**:

| Aspect | Transactional Communication | Marketing Communication |
| :--- | :--- | :--- |
| **Purpose** | Operational necessity tied to domain actions (order confirmation, shipping, returns, refunds, security) | Promotional messaging, product highlights, seasonal restock notices |
| **Consent Requirement** | Always sent; operational requirement for order fulfillment | Requires explicit customer consent (`marketing_email: true`) |
| **Trigger** | Domain events / state transitions | Scheduled editorial campaigns (Step 10 boundary) |
| **Unsubscribe Impact** | Marketing unsubscribe NEVER disables transactional notifications | Unsubscribe immediately halts promotional email dispatch |

### 2. Notification Model & Taxonomy

The `Notification` domain model in `apps.notifications` is extended with:
- `user`: Foreign key to `users.User` for authenticated customer isolation.
- `order`: Foreign key to `orders.Order` (nullable).
- `return_request`: Foreign key to `orders.ReturnRequest` (nullable).
- `refund`: Foreign key to `payments.Refund` (nullable).
- `subject`: Plaintext subject line recorded at dispatch time.
- `status`: `pending`, `sent`, `failed`.
- `attempt_count`: Counter tracking retry/delivery attempts.
- `sent_at` and `failed_at`: Explicit audit timestamps.
- `provider_message_id`: Provider receipt or deterministic message identifier.
- `error_message`: Captured exception string on failure.

Controlled Notification Taxonomy:
- **Account:** `account_welcome`
- **Orders:** `order_placed`, `order_processing`, `order_shipped`, `order_delivered`, `order_cancelled`
- **Payments:** `payment_confirmed`, `payment_failed`
- **Returns:** `return_requested`, `return_approved`, `return_rejected`, `return_received`
- **Refunds:** `refund_initiated`, `refund_completed`

### 3. Idempotency & Deduplication

To prevent duplicate emails on network retries or concurrent status triggers:
- Unique constraints protect `(order, event_type)`, `(return_request, event_type)`, `(refund, event_type)`, and `(user, event_type)` for non-repeating events.
- Dispatch services perform an idempotency check before creation; existing records are returned without re-sending.

### 4. Failure Isolation & Commerce Independence

The commerce transaction remains authoritative:
- Order placement, payment confirmation, fulfillment transitions, and refund records MUST NOT roll back if notification dispatch fails.
- Exceptions during template rendering or provider delivery are trapped, logged, and recorded as `status = "failed"` with `error_message`.
- Notification retries operate independently and never alter order or payment state.

### 5. Provider Abstraction

Communication channels use a provider adapter pattern (`BaseEmailProvider`):
- **Development/Test:** Uses Django's email backend (`locmem` in unit tests, `console` in local development) with structured message ID generation.
- **Production:** Configurable via standard Django SMTP or API-based email backends (`settings.EMAIL_BACKEND`, `EMAIL_HOST`, `EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD`, `DEFAULT_FROM_EMAIL`).
- The application business logic never references provider-specific SDKs directly.

### 6. Templates & Data Security

Email templates receive pre-calculated, authoritative domain context:
- No totals, discounts, or taxes are computed in templates; backend values are displayed as-is.
- HTML templates use safe escaped variables to prevent injection.
- Passwords, JWT tokens, payment cards, or provider secret keys are never included in email payloads or logged.
- Links use the canonical storefront base URL (`FRONTEND_URL` / `NEXT_PUBLIC_SITE_URL`).

### 7. Customer Communication Preferences

A `CommunicationPreference` record is attached to each customer user:
- `marketing_email` (default: `False`): Consent for promotional campaigns.
- `marketing_sms` (default: `False`): Consent for promotional text messages.
- `order_updates_email` (default: `True`): Operational indicator; cannot be unsubscribed by marketing settings.
- APIs:
  - `GET /api/v1/customer/communication-preferences/`
  - `PATCH /api/v1/customer/communication-preferences/`

### 8. Staff / CMS Operational Visibility

Staff with `notifications.read` permission can view delivery history:
- Endpoint: `GET /api/v1/admin/notifications/` with filters for `status`, `event_type`, `recipient`, and `channel`.
- Detail: `GET /api/v1/admin/notifications/<public_id>/`.
- Resend: `POST /api/v1/admin/notifications/<public_id>/resend/` with `notifications.manage` permission.

## Consequences

- Full operational transparency for customer delivery status without introducing heavy Celery/Redis dependencies.
- Absolute compliance with marketing consent while guaranteeing delivery of critical commerce updates.
- Storefront customers can monitor past communications and control their marketing preferences.
- Staff have an audited operational delivery log directly in the CMS.
