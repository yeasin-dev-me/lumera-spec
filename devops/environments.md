# Environments

No infrastructure is configured by TASK-001; this defines the strategy.

| | Development | Staging | Production |
|---|---|---|---|
| Purpose | Local building and debugging | Production-like validation, integration/E2E, release candidates | Real customers |
| Where | Developer machine, Docker Compose | VPS/cloud (smaller), behind Cloudflare | VPS/cloud behind Cloudflare + Nginx |
| Deploy trigger | Manual | Automatic from `main` after CI | Version tag + manual approval |
| `DEBUG` | true | false | false |
| HTTPS / HSTS | no | yes | yes |
| Database | Local PostgreSQL container, disposable | Dedicated PostgreSQL, non-production data | Dedicated PostgreSQL, backups, restricted access |
| Data | Seed/fake | Anonymized/fake only — **never** raw production data | Real |
| Payments | Provider test mode / stub | Provider test mode | Live keys |
| Email | Console/mail catcher | Sandbox | Real provider |
| Secrets | `.env` (git-ignored) | Secret store, distinct from prod | Secret store, restricted |
| CORS/hosts | localhost allow-list | staging domain only | prod domain only |
| Monitoring/backups | none | basic health + logs | health, error tracking, alerts, automated backups, tested restore |
| Access | Developers | Team (basic auth/Cloudflare Access recommended; not indexed by search engines) | Public |

## Configuration Rules
- 12-factor: configuration only through environment variables; same images/artifacts promoted staging → production.
- Each app provides `.env.example` with all variables and **placeholder** values; real values never committed.
- Django settings split: `base.py`, `development.py`, `staging.py`, `production.py` selected via `DJANGO_SETTINGS_MODULE`. Next.js uses `NEXT_PUBLIC_*` only for non-secret values.
- Initial variable catalog (to be finalized by DevOps/Backend): `DJANGO_SECRET_KEY`, `DJANGO_SETTINGS_MODULE`, `DJANGO_ALLOWED_HOSTS`, `DATABASE_URL`, `CORS_ALLOWED_ORIGINS` (storefront **and** CMS origins), `CSRF_TRUSTED_ORIGINS`, `JWT_*` lifetimes (customer and CMS separately), `INTEGRATION_ENCRYPTION_KEY`, `PAYMENT_*` (TBD, environment-only by default), `EMAIL_*` (TBD), `NEXT_PUBLIC_API_BASE_URL` (storefront and CMS; public values only), `SENTRY_DSN` (optional, TBD). Tracking IDs (Pixel/GA) are CMS-managed **public** settings, not env vars.
- **Per-environment hosts:** Development: storefront `localhost:3000`, CMS `localhost:3001`, backend `localhost:8000`. Staging/Production: storefront host, CMS host (e.g. `admin.<domain>`, optionally behind Cloudflare Access / IP allow-list, `noindex`), API under `/api/v1/` on each host via Nginx.
- The CMS is deployed per environment like the other apps; staging CMS manages staging data only; production CMS credentials/roles are separate from staging.
- Differences between environments are limited to configuration, never code paths.

## Release & Rollback (principles)
- Immutable, versioned Docker images; deploy by tag; previous image retained for rollback.
- Migrations are backward-compatible so a rollback of application code is safe.
- Post-deploy smoke test against health endpoints; failure triggers rollback.
