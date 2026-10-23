# ADR-0001: License server as a custom Frappe app

| Field | Value |
|---|---|
| Status | **Superseded** by [ADR-0004](0004-license-server-on-django.md) on 2026-10-23. Navil chose a standalone Django application with no ERPNext integration. Kept for the record |
| Date | 2026-10-19 |
| Related | [Decision brief](../00-decision-brief.md), [Requirements](../01-requirements.md) |

## Context

The license server must:

- run on Navil's own server;
- give clients a self-service portal to activate and deactivate machines, limited to 3 resets
  per month;
- support online activation and offline activation by file;
- serve desktop apps written in .NET and Python, and later Navil's hosted services.

Navil has deep Frappe experience, and Frappe is already part of his web stack.

## Decision

Build `license_server` as a custom Frappe app on a **dedicated Frappe site**. Use a dedicated
bench if the existing bench runs third-party or client code: apps on the same bench run as the
same OS user and could read the signing key file.

Frappe already provides:

- DocTypes and the admin desk UI
- Roles and permissions
- Website portal pages
- Whitelisted API methods
- The scheduler and email
- Document version history

We build:

- Token signing (ADR-0002)
- The activation and reset rules
- The offline file exchange
- The portal pages
- The client SDKs

## Alternatives considered

| Option | Why rejected |
|---|---|
| Keygen CE, self-hosted | It is a separate stack (Rails, PostgreSQL, Redis) to run beside Frappe and MariaDB, and it is not integrated with Navil's Frappe data. Rejected on fit, not on capability, and not evaluated hands-on. |
| Cryptolens | A vendor-hosted SaaS, so license data would sit on the vendor's servers. That conflicts with the requirement to run on Navil's server. |
| Standalone ASP.NET Core or Node service | It would rebuild the admin UI, permissions, portal and scheduler that Frappe already provides. |

## Consequences

**Benefits**

- The admin UI and portal come fast.
- Navil already has the skills.
- A direct path to linking with ERPNext invoices later (Phase 3).

**Costs**

- **The license server is tied to Frappe's upgrade cycle.** Mitigation: pin the version and run
  the test suite before every upgrade.
- **The private key lives on the same host as a public web app.** Mitigation: keep the key file
  outside the database and the backups. If a review requires stronger isolation, move signing
  into a small separate service.
- **The portal needs hardening:** rate limits, permission tests, and two-factor authentication
  where supported.

## Revisit if

- A client or an audit requires the signing key to be isolated from the web host.
- Load grows beyond what one Frappe site can serve. This is not expected at the current scale.

## Sources

- Keygen self-hosting docs: https://keygen.sh/docs/self-hosting/
- Keygen Fair Source announcement: https://keygen.sh/blog/all-your-licensing-are-belong-to-you/
- Cryptolens, offline license verification: https://cryptolens.io/2024/07/offline-license-verifications
