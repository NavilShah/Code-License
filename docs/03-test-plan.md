# Code-License: Test Plan

| Field | Value |
|---|---|
| Version | 0.1 (draft) |
| Date | 2026-10-23 |
| Status | Draft |
| Scope | Phase 1: the Django license server, the .NET SDK, and the sample Caliburn.Micro app |
| Related | [Requirements](01-requirements.md) · [Task graph](04-phase1-task-graph.md) · ADR-0002, ADR-0003, ADR-0004 |

## 1. Test levels

| Level | What it covers | Tooling | Where it runs |
|---|---|---|---|
| Server unit | Models, activation rules, reset window, expiry computation, permissions, rate limits | The Django test runner or pytest-django (chosen in T2) | CI, Linux |
| SDK unit | Signature checks, the state machine with an injected clock, clock rollback, fingerprint matching | A .NET test framework (chosen in T2), run on net47 and net8.0 | CI, Windows runner |
| Test vectors | The contract between the server and the SDKs (§2) | JSON fixtures in `test-vectors/` | Both of the above |
| Integration | The SDK against a running server | The net8.0 SDK build, with a fake fingerprint provider, against the server running in Docker Compose | CI, Linux |
| .NET Framework 4.7 runtime | That built-in ECDSA works on the oldest target | A test run on a Windows VM whose newest .NET Framework is 4.7 or 4.7.1 | Manual, in T1 and again before the pilot |
| Fingerprint | Reading the components, and the matching rule, on real machines | A physical PC, a VM and a VM clone | Manual, in T9 |
| Enforcement | Popup, warnings, cycle blocking, exit at idle | The sample Caliburn.Micro app with a simulated cycle | Manual, in T10 |
| Commissioning | Enforcement in a real industrial app | A non-production cell or a simulator. **Never first on a production cell** | Manual, before the pilot |
| Security and operations | Key handling, backups, permissions | Checklist TS-22 to TS-25 | Manual on the production host, plus CI checks |

**Why the manual 4.7 run is needed:** .NET Framework 4.x versions are in-place updates, so on a
CI runner with a newer 4.x installed, a `net47` build runs on that newer runtime. CI therefore
cannot prove the behaviour on 4.7 itself.

## 2. Test vectors

The test vectors are the shared contract between the server and every SDK.

- They are signed with a separate test key pair whose `kid` starts with `test-`. Its private key
  is committed under `test-vectors/keys/` and labelled test-only. It is never used in production.
- SDK release builds must not embed any `test-` key (TS-23). Otherwise anyone could sign valid
  licenses with the committed test key.
- Each vector records: the token, the fingerprint input, the simulated current time, and the
  expected state.

| Case | Expected state |
|---|---|
| Valid token | Valid |
| One byte changed in the payload | Invalid |
| One byte changed in the signature | Invalid |
| `alg` set to `none` | Invalid |
| `alg` set to `HS256`, signed using the public key as the HMAC secret | Invalid |
| Unknown `kid` | Invalid |
| Wrong product code | Invalid |
| Inside the warning window | Expiring |
| Past expiry, within grace | Grace |
| Past expiry and grace | Expired |
| Perpetual license (9999-12-31) | Valid |
| One second before, then one second after, an expiry at 23:59:59 IST (18:29:59 UTC) | Expiring, then Grace |
| Schema major version one higher than the SDK knows | Invalid, with an "update application" message |
| Fingerprint with only `mg` or only `ds` changed | Valid |
| Fingerprint with `su` changed | Invalid |
| Fingerprint with two components changed | Invalid |
| Activation that recorded a placeholder `su` | All recorded components must match |
| Raw fingerprint values hashed | Identical hashes in every SDK |

## 3. Scenarios and traceability

| ID | Scenario | Requirement | Level |
|---|---|---|---|
| TS-01 | Issue a license: the key is shown once and stored hashed; the effective end date is displayed | FR-LIC-1 | Server unit |
| TS-02 | A 9999-12-31 license is Valid everywhere and is stored in a DATE column | FR-LIC-2, NFR-7 | Server unit, vectors |
| TS-03 | Extend a license: the next refresh returns the new expiry, and an updated offline file is offered | FR-LIC-3, FR-OFF-3 | Integration |
| TS-04 | Revoke a license: the next refresh moves the app to Invalid | FR-LIC-4 | Integration |
| TS-05 | Activate within the limit; activating one machine too many returns `LIMIT_REACHED` | FR-ACT-1 | Server unit, integration |
| TS-06 | Re-activating the same machine uses neither a seat nor a reset | FR-ACT-2 | Server unit |
| TS-07 | A 4th deactivation within 30 days returns `RESET_LIMIT_REACHED`; an admin deactivation does not count | FR-ACT-3, FR-ACT-6 | Server unit |
| TS-08 | Repeated failures temporarily block a key, and the block is logged | FR-ACT-4 | Server unit |
| TS-09 | With the server down, the app's state is unchanged and it keeps running | FR-ACT-5, FR-ENF-6, NFR-1 | Integration |
| TS-10 | An offline request yields a license file that is Valid on that machine and Invalid on another | FR-OFF-1, FR-OFF-2 | Integration, manual |
| TS-11 | Replaying an offline request (same nonce) is rejected | FR-OFF-2 | Server unit |
| TS-12 | An offline deactivation code is accepted once and counts as a reset | FR-OFF-4 | Server unit |
| TS-13 | The signature, `alg` and `kid` vectors pass | FR-SDK-1 | Vectors |
| TS-14 | Fingerprint tolerance follows ADR-0003 | FR-SDK-2 | SDK unit, manual |
| TS-15 | Setting the clock back by more than 2 hours makes the license Invalid; normal forward drift does not | FR-SDK-4 | SDK unit |
| TS-16 | Starting without a license shows the popup, then exits | FR-ENF-1 | Sample app |
| TS-17 | Warnings appear from 7 days before expiry | FR-ENF-2 | Sample app |
| TS-18 | Industrial app: expiry mid-cycle lets that cycle finish, refuses the next one, and exits at idle | FR-ENF-4 | Sample app, commissioning |
| TS-19 | Non-industrial app: expiry while running saves the user's work, then exits | FR-ENF-3 | Sample app |
| TS-20 | Portal users of customer A cannot see or act on customer B's licenses | FR-PRT-1 | Server unit |
| TS-21 | Every action writes an event, and portal users cannot edit or delete events | FR-AUD-1 | Server unit |
| TS-22 | The private key is absent from the backup output, the Docker image and the repository (secret scan) | NFR-2 | Operations check, CI |
| TS-23 | SDK release builds contain no `test-` key | NFR-2 | CI |
| TS-24 | Restoring from a backup works | NFR-4 | Operations check |
| TS-25 | The .NET SDK package has no third-party runtime dependencies | FR-SDK-6 | CI |
| TS-26 | Navil's admin login requires a second factor | FR-PRT-3 | Operations check |

## 4. Pilot entry criteria

- All automated tests pass: server, SDK on net47 and net8.0, vectors and integration.
- The manual .NET Framework 4.7 run passes.
- TS-16 to TS-19 pass on the sample app, and TS-18 also passes on a non-production cell or a
  simulator.
- TS-22, TS-24 and TS-26 are done on the production host.
- The test report is written as `docs/reports/phase1-test-report.md`.
- **Navil approves the pilot.**

## 5. Not tested in Phase 1

- The Python SDK and the hosted-service entitlement check (Phase 2).
- Load. The scale is small (N5 is still open); revisit if it grows.
