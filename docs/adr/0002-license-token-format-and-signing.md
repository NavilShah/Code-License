# ADR-0002: License token format and signing

| Field | Value |
|---|---|
| Status | Proposed. Confirm with the Phase 1 interop spike and the answer to N1 (oldest .NET Framework version) |
| Date | 2026-10-19 |
| Related | [Requirements](../01-requirements.md) FR-SDK-1, NFR-2, NFR-8 |

## Context

The Frappe app (Python) signs license tokens. They must verify **offline** in apps built on
.NET Framework, .NET Core and Python, and possibly Node later. The private key never leaves
Navil's server.

## Decision

### Format

Use a JWS in compact serialization (the JWT structure). The header carries `alg` = `ES256`
(ECDSA on P-256 with SHA-256) and a `kid`.

### Why ES256 rather than Ed25519

ECDSA P-256 is built into .NET Core, Python's `cryptography` library and Node's `crypto`, and
into .NET Framework. On .NET Framework, importing a key from parameters needs 4.7 or later;
earlier versions need the Windows CNG key import. Confirm this in the spike.

Ed25519 is not built into .NET Framework and would need a third-party library.

### Claims

| Claim | Meaning |
|---|---|
| `ver` | Token schema version |
| `lic` | License ID |
| `act` | Activation ID |
| `cus` | Customer ID |
| `prd` | Product code |
| `fpr` | Array of fingerprint component hashes |
| `feat` | Enabled features |
| `iat` | Issued at (Unix seconds, UTC) |
| `lic_exp` | Expiry instant (Unix seconds, UTC). For a perpetual license, 9999-12-31 at 23:59:59 in the server timezone |
| `grace_days`, `warn_days` | Enforcement thresholds |
| `status` | Active, Suspended or Revoked |

**There is deliberately no standard `exp` claim.** JWT libraries reject an expired token
outright, which would skip the warning and grace states. The SDK computes the state itself from
`lic_exp`.

### Verification rules (SDK)

1. Pin the algorithm to ES256, and reject anything else, including `none` and `HS256`.
2. The `kid` must match a public key **embedded in the SDK build**. The SDK never trusts a key
   fetched from the network, which would let anyone who can intercept the traffic inject their
   own key.
3. Verify the signature before reading any claim.

### Signature encoding between languages

- Python's `cryptography` library produces DER-encoded ECDSA signatures.
- JWS requires the raw `r||s` form (64 bytes).
- .NET's `ECDsa.VerifyData` expects `r||s` (IEEE P1363) by default.

So the server must use a JWS library or convert the encoding explicitly. Cross-language test
vectors (NFR-8) cover this, and they are the first task of the Phase 1 spike.

### Key handling

- There is one active signing key. Its private key is a PEM file outside the database and the
  backups.
- SDK builds embed the current public key and the next one.
- **Planned rotation:**
  1. Generate the new key.
  2. Ship SDK builds containing both public keys.
  3. Switch signing to the new key.
  4. Tokens are reissued as machines refresh, and offline licenses get new files.
  5. Retire the old key once every app has been updated.
- **On compromise:** follow the same steps urgently. In addition, remove the old `kid` from the
  next SDK release, and reissue every license file in the field.

## Alternatives considered

| Option | Why not chosen |
|---|---|
| Ed25519 / EdDSA | Not built into .NET Framework |
| PASETO v4.public | Based on Ed25519, so the same .NET Framework issue, and fewer libraries |
| Standard.Licensing | A .NET-only XML format, so the Python side would need reimplementing |
| RS256 / PS256 (RSA-2048) | Works on every target, including older .NET Framework, but tokens and keys are larger. **This is the fallback if ES256 fails on the N1 version.** |

## Consequences

- One token format across all runtimes, built on standard, widely implemented primitives.
- Cross-language test vectors become a permanent part of CI.
