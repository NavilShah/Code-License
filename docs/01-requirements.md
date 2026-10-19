# Code-License: Requirements

| Field | Value |
|---|---|
| Version | 0.1 (draft) |
| Date | 2026-10-19 |
| Status | Draft. FR-ENF-4 waits on D1; grace defaults on D2; reset rules on D3 (see decision brief §7) |
| Scope | Phase 1 unless a requirement is marked otherwise |

## 1. Scope

| In scope for Phase 1 | Later phases | Out of scope |
|---|---|---|
| License server (Frappe app) | Python SDK (Phase 2) | Billing and payments |
| Client portal | Entitlement checks for hosted services (Phase 2) | Floating or concurrent seats |
| Online and offline activation | Expiry reminder emails (Phase 2) | On-prem license server |
| .NET SDK | Licensing for third-party extensions (Phase 3) | Anti-debugging and DRM |
| Enforcement policy | Extending licenses on payment via ERPNext (Phase 3) | Any change to safety functions |

## 2. Actors

| Actor | Responsibilities |
|---|---|
| Navil (vendor admin) | Issues, extends, suspends and revokes licenses; manages products and signing keys |
| Client admin (portal user) | Views their licenses; activates and deactivates machines; exchanges offline files |
| Desktop app with SDK | Activates, refreshes, verifies the license, and enforces the policy |
| Hosted service (Phase 2) | Checks whether a client is entitled to use it |

## 3. Domain model (Frappe DocTypes)

```mermaid
erDiagram
  LICENSE_CUSTOMER ||--o{ LICENSE : holds
  LICENSE_PRODUCT ||--o{ LICENSE : "is licensed by"
  LICENSE ||--o{ LICENSE_ACTIVATION : "is bound to"
  LICENSE ||--o{ LICENSE_EVENT : "is audited by"
  LICENSE_SIGNING_KEY ||--o{ LICENSE_ACTIVATION : signs
```

| DocType | Purpose | Key fields |
|---|---|---|
| License Customer | A client | `customer_name`, `email`, `portal_users` (child table of User links); optional link to an ERPNext Customer (Phase 3) |
| License Product | A product that can be licensed | `product_code` (unique; used in tokens), `product_name`, `platform` (.NET desktop / Python desktop / Hosted service / Extension), `is_industrial` (selects the FR-ENF-4 behaviour), `default_grace_days`, `default_warn_days`, `features` (child table) |
| License | The right to run one product on up to N machines | `license_key` (stored hashed; shown once), `customer`, `product`, `max_activations` (default 1), `expires_on` (9999-12-31 means perpetual), `grace_days`, `warn_days`, `status` (Active / Suspended / Revoked), `features`, `notes` |
| License Activation | One machine bound to a license | `license`, `fingerprint_hashes`, `machine_label`, `mode` (Online / Offline), `status` (Active / Deactivated), `activated_on`, `deactivated_on`, `deactivation_source` (App / Portal / Offline code / Admin), `last_refresh_on`, `app_version`, `kid` |
| License Signing Key | Metadata and public half of a signing key | `kid`, `algorithm` (ES256), `public_key_pem`, `status` (Active / Retired). **The private key is never stored in the database.** |
| License Event | Append-only audit log | `license`, `activation`, `event_type`, `actor`, `source` (Portal / API / Desk), `payload` (JSON), `creation` |

Resets are counted from deactivations recorded on License Activation within the reset window,
rather than kept in a separate counter. That keeps a single source of truth.

## 4. Functional requirements

Each requirement is followed by its acceptance criteria (AC).

### Issuing licenses (FR-LIC)

- **FR-LIC-1** Navil can issue a license for a customer and a product, setting
  `max_activations`, `expires_on`, `grace_days` and `warn_days`.
  - AC: the license key is displayed once, when the license is created. It is stored as a hash.
    An `Issued` event is logged.
- **FR-LIC-2** A perpetual license has `expires_on` = 9999-12-31.
  - AC: a 9999-12-31 license verifies as Valid on the server and in the .NET SDK, with no
    overflow. It is stored in a DATE or DATETIME column, never TIMESTAMP.
- **FR-LIC-3** Navil can extend a license by changing `expires_on`.
  - AC: the next refresh from an online machine returns a token with the new expiry. The portal
    offers an updated license file for each active offline activation. An `Extended` event is
    logged with the old and new dates.
- **FR-LIC-4** Navil can suspend or revoke a license.
  - AC: new activations are rejected. A refresh returns a revoked status, and the SDK moves to
    Invalid.
  - Known limitation (accepted): a machine that never refreshes keeps running until its expiry.

### Activation (FR-ACT)

- **FR-ACT-1** Online activation. The SDK sends the license key, product code, hashed
  fingerprint components, machine label and app version.
  - AC: if the license is Active and not expired, the product matches, and the active
    activations are fewer than `max_activations`, the server returns a signed token bound to
    the fingerprint and records the activation.
  - AC: otherwise it returns a structured error code: `LICENSE_NOT_FOUND`, `PRODUCT_MISMATCH`,
    `EXPIRED`, `LIMIT_REACHED` or `SUSPENDED`.
  - AC: either way, an event is logged.
- **FR-ACT-2** Re-activating the same machine is idempotent.
  - AC: the same fingerprint gets a fresh token and uses up neither a seat nor a reset.
- **FR-ACT-3** Deactivating a machine, whether from the app or the portal, frees its seat and
  counts as one reset.
  - AC: a deactivation beyond the limit (3 per window, D3) is rejected with
    `RESET_LIMIT_REACHED`, together with the remaining count and the date the next reset becomes
    available.
  - Known limitation: after a portal deactivation, an online machine only learns of it at its
    next refresh.
- **FR-ACT-4** Activation, refresh and deactivation endpoints are rate-limited per IP address and
  per license key, using Frappe's `rate_limit` decorator (verify it exists in the target Frappe
  version).
  - AC: too many failed attempts on one key in one hour temporarily block that key, and the
    block is logged.
- **FR-ACT-5** When the server is reachable, an online app refreshes at startup and every
  24 hours (configurable).
  - AC: an unreachable server causes no change in license state.

### Offline activation (FR-OFF)

- **FR-OFF-1** The SDK writes an offline request file containing the license key, product code,
  fingerprint hashes, a nonce and the app version.
- **FR-OFF-2** The client admin uploads the request in the portal. The server applies the
  FR-ACT-1 rules and returns a license file to download.
  - AC: imported on the requesting machine, the file is Valid. The same file on another machine
    is Invalid because of the fingerprint mismatch.
- **FR-OFF-3** After an extension, the portal offers an updated license file for each active
  offline activation.
- **FR-OFF-4** For offline deactivation, the SDK removes the license and displays a deactivation
  code, which the client admin enters in the portal.
  - AC: the server confirms the code belongs to that activation (license ID, fingerprint, nonce,
    checksum) and counts it under the D3 rules.
  - Known limitation: the server cannot prove the machine stopped using the license.

### Verification in the SDK (FR-SDK)

- **FR-SDK-1** The SDK verifies the signature using public keys embedded in the SDK, selected by
  `kid`. Only ES256 is accepted (ADR-0002).
  - AC: a token with one byte changed is Invalid. A token with `alg` none or HS256 is Invalid. A
    token with an unknown `kid` is Invalid.
- **FR-SDK-2** The SDK checks that the product code matches, and checks the fingerprint with a
  tolerant rule: 2 of 3 components must match (details in ADR-0003, pending).
  - AC: with one component changed the token is still Valid; with two changed it is Invalid.
- **FR-SDK-3** The SDK computes the license state (§5) from the token's claims and the local
  UTC clock.
- **FR-SDK-4** The SDK detects the clock being set back. It stores the last time it saw; if the
  current UTC time is earlier than that by more than the tolerance (default 2 hours), the
  license is Invalid until the next online refresh or until a newer license file is imported.
- **FR-SDK-5** Apps use one interface (for example `ILicenseProvider`, with `GetState`,
  `Activate`, `Refresh`, `CreateOfflineRequest`, `ImportLicenseFile`, `Deactivate` and a
  `StateChanged` event). Apps never call the server directly.
- **FR-SDK-6** The .NET SDK targets netstandard2.0, so one package serves both .NET Framework and
  .NET Core. It uses no third-party crypto library unless N1 forces it.

### Enforcement (FR-ENF)

- **FR-ENF-1** At startup, if the state is Unlicensed, Expired or Invalid, the app shows a popup
  offering **Activate**, **Import license file** and **Exit**. The app exits if the license is
  not resolved. (Navil's rule.)
- **FR-ENF-2** From `warn_days` before expiry (default 7), the app shows a warning at startup and
  once a day.
- **FR-ENF-3** If the license expires while the app is running and the product is not
  industrial: show the popup, save the user's work, then exit.
- **FR-ENF-4** If the license expires while the app is running and the product is industrial:
  **pending D1.** Recommended: show the popup, block new cycles and jobs, and exit when idle.
- **FR-ENF-5** License logic never runs inside a motion or cycle loop, and never stops or alters
  a safety function.
- **FR-ENF-6** An unreachable server never changes the license state.

### Portal (FR-PRT)

- **FR-PRT-1** A client admin sees only their own customer's licenses: product, expiry,
  activations, and resets used and remaining.
  - AC: a permission test shows a user of customer A cannot read or act on customer B's records.
- **FR-PRT-2** A client admin can deactivate a machine, upload an offline request, download
  license files and enter offline deactivation codes.
- **FR-PRT-3** Portal logins are protected. Use two-factor authentication if the target Frappe
  version supports it for portal users (to verify).

### Audit (FR-AUD)

- **FR-AUD-1** Every issue, extension, suspension, activation, deactivation, refresh, and every
  rejected attempt writes a License Event.
  - AC: portal users cannot edit or delete events.

### Hosted services (FR-HST, Phase 2)

- **FR-HST-1** A hosted service checks the client's entitlement (customer and service) by API
  key against the license server. It caches the result locally for about 10 minutes. If the
  server is down, cached entries stay valid for up to 24 hours.
  - AC: an expired entitlement returns HTTP 403 with `LICENSE_EXPIRED`, and the web app shows a
    license-expired page.

## 5. License state machine

```mermaid
stateDiagram-v2
    [*] --> Unlicensed
    Unlicensed --> Valid: activated / license file imported
    Valid --> Expiring: now >= expiry - warn_days
    Expiring --> Grace: now > expiry
    Grace --> Expired: now > expiry + grace_days
    Expiring --> Valid: extended
    Grace --> Valid: extended
    Expired --> Valid: extended
    Valid --> Invalid: tampered / wrong machine / revoked / clock set back
    Invalid --> Valid: valid license re-applied
```

| State | App behaviour |
|---|---|
| Valid | Full function |
| Expiring | Full function, plus a warning (FR-ENF-2) |
| Grace | Full function, plus a prominent warning |
| Expired / Invalid / Unlicensed | Popup, then exit (FR-ENF-1/3/4) |

## 6. Non-functional requirements

- **NFR-1 Availability.** If the license server is down, only new activations, refreshes and the
  portal are affected. Running apps and apps already activated are not.
- **NFR-2 Security.**
  - The private key is a file outside the database and outside the backup paths (Frappe's
    backups include `site_config.json`; verify in the target version). It has permissions 600
    and is owned by the bench user.
  - The key is never in git, CI or logs.
  - HTTPS only. Only ES256 is accepted. Rate limits as in FR-ACT-4.
- **NFR-3 Privacy.** Collect only hashed fingerprint components, the machine label, the app
  version, timestamps, and the IP address of online calls. Set a retention period for events (to
  be decided).
- **NFR-4 Backup.** Back up the database daily, and test a restore once before the pilot.
- **NFR-5 Observability.** A daily scheduled job reports licenses expiring within N days and any
  rejected attempts. Frappe's Error Log is monitored.
- **NFR-6 Compatibility.** The token schema carries a version (`ver`). The SDK rejects an unknown
  major version and tells the user to update the application.
- **NFR-7 Dates.**
  - All token times are UTC.
  - A license expires at 23:59:59 on `expires_on` in the server's configured timezone, converted
    to UTC in the token.
  - Tests cover 9999-12-31 and the moment of expiry at the day boundary.
- **NFR-8 Interoperability.** Test vectors signed by the server (Python) are verified in CI by
  the .NET SDK on .NET Framework (oldest version from N1) and on .NET 8, and by the Python SDK in
  Phase 2.

## 7. Next artifacts

ADR-0003 (machine fingerprint), the test plan, and the Phase 1 task graph.
