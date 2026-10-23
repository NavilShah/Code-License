# ADR-0004: License server as a standalone Django application

| Field | Value |
|---|---|
| Status | Accepted: Navil's decision of 2026-10-23. The supporting choices below (database, packages, deployment) are proposed and get confirmed in T2 |
| Date | 2026-10-23 |
| Supersedes | [ADR-0001](0001-license-server-on-frappe.md) (license server as a custom Frappe app) |
| Related | [Decision brief](../00-decision-brief.md) D5, [Requirements](../01-requirements.md) |

## Context

On 2026-10-23 Navil decided not to integrate the license server with ERPNext. It will be built
as a standalone Python web application on Django instead. The requirements are unchanged.

Frappe supplied several things for free, and now each must be provided some other way:

- an admin UI;
- a client portal;
- roles and permissions;
- a scheduler;
- rate limiting;
- two-factor login.

## Decision

| Concern | Choice | Why |
|---|---|---|
| Framework | Django 5.2 LTS (5.2.18 on PyPI on 2026-10-23; supported until April 2028). Move to the next LTS, 6.2, after it ships (expected April 2027) | LTS releases get security fixes for about three years |
| Database | PostgreSQL | Django's best-supported database. MariaDB is acceptable if Navil would rather run a single database engine; both store 9999-12-31 in a DATE column |
| Navil's admin screens | The Django admin, with two-factor login (`django-otp`) | Covers issuing, extending and suspending licenses, and support deactivations, without building screens |
| Client API (activate, refresh, deactivate, offline exchange) | Plain Django views that return JSON | About five endpoints. Django REST framework would add a dependency for little gain |
| Client portal | Django's authentication views plus server-rendered templates | A handful of pages; no single-page app needed |
| Rate limiting | `django-ratelimit`, backed by Django's database cache | The counters must be shared by all worker processes; the database cache does this without adding Redis |
| Scheduled jobs | Django management commands run by a systemd timer or cron | One daily job needs neither Celery nor a message broker |
| Signing | Python's `cryptography` library, with the key file's path taken from an environment variable | ADR-0002 |
| Deployment | Docker Compose on Navil's server: nginx (TLS) in front of gunicorn and Django, with PostgreSQL in its own container | Navil already works with Docker. The license server shares no process or OS user with other apps |

**Runtime dependencies:** Django, psycopg, cryptography, django-ratelimit, django-otp and
gunicorn. Any other runtime dependency needs a recorded reason.

## Alternatives considered

| Option | Why not chosen |
|---|---|
| Custom Frappe app (ADR-0001) | Superseded by Navil's decision. Frappe gave the portal, roles and scheduler for free. Django needs those built or configured, but in return the app is smaller and conventionally structured, upgrades independently of ERPNext, and keeps the signing key in its own containers |
| FastAPI | Has no admin UI, so Navil's admin screens would have to be built |
| Flask | The admin, authentication and migrations would all have to be assembled from separate packages |

## Consequences

**Benefits**

- The signing key lives in a small, single-purpose deployment.
- The Django admin gives Navil working management screens from day one.
- Testing (the Django test runner or pytest-django) and migrations follow standard practice.
- Upgrades do not depend on Frappe or ERPNext.

**Costs**

- The portal pages and the scheduled job are built by hand, adding about a day to T7.
- One more stack to run beside Frappe/ERPNext and Node.
- **Django is not in Navil's listed experience.** If it is new to him, budget learning time. The
  estimates do not include it (R-16).
- There is no automatic extension on payment. Extensions are made by hand in the admin.

## Revisit if

- Clients need a richer portal than server-rendered pages can give.
- Extending licenses automatically on payment becomes worth building. Add a webhook endpoint
  then, rather than coupling the server to an ERP.
