# ADR-0002: Modular monolith backend
- Status: Accepted (specified by the project brief)
- Date: 2026-10-04

## Context
Domain is small-to-medium; operational simplicity matters.

## Decision
One Django project with apps: `users, products, categories, cart, orders, payments, reviews, common`. Business operations live in `services.py`; complex reads in optional `selectors.py`. No microservices, queues, or caches until measured need.

## Alternatives Considered
Microservices (rejected: operational overhead); one big app (rejected: poor modularity).

## Consequences
Apps interact through services/selectors, not views. Extraction to a service later remains possible via those seams.

**Amendment note:** the app list is extended by the Proposed [ADR-0011](ADR-0011-backend-app-additions.md) to support CMS features. Django Admin is explicitly not used.
