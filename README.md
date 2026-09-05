# QR-V™ Security

Security architecture, threat models, production-hardening requirements, issuer trust, and cryptographic verification standards for the QR-V™ Global Verification Network.

## Production trust boundary

QR-V Production Architecture v1.0 uses exactly two active runtime nodes:

```text
qrv.network
  public platform, verification UI, issuer UI, docs, registry/explorer UI
      ↓ authenticated server-to-server calls
api.qrv.network
  trusted API, canonical persistence, registry mutation, verification state,
  issuer authorization, cryptographic processing, audit, rate limiting
      ↓
canonical QR-V registry datastore
```

Legacy QR-V subdomains are compatibility aliases only. They are not separate trust boundaries.

### Mandatory boundary rules

- `qrv.network` must not receive `DATABASE_URL`.
- `qrv.network` must not receive `SUPABASE_SECRET_KEY`.
- `qrv.network` must not receive signing private keys, webhook secrets, payment-provider secrets, or database-admin credentials.
- `api.qrv.network` is the only production application node allowed to own canonical registry credentials.
- `QRV_PLATFORM_API_KEY` is server-to-server and must never be exposed to browser JavaScript.
- Write operations fail closed when authorization is missing.
- CORS on the API must allow only explicitly approved origins, starting with `https://qrv.network`.
- Database secrets, issuer signing keys, webhook secrets, privileged API keys, and server-side billing secrets belong on the API node only.
- Redirect-only legacy hostnames must use canonical HTTP 308 redirects and must not host duplicate writable applications.

## Single-authority datastore rule

QR-V must have exactly one writable canonical registry authority.

The current production contract is PostgreSQL / managed PostgreSQL through `DATABASE_URL` on `api.qrv.network`.

Supabase is permitted only as an intentional replacement persistence adapter. If adopted, `SUPABASE_URL` and `SUPABASE_SECRET_KEY` remain server-side on the API node, and the old datastore must not remain a competing writable source of truth.

Split-brain registry state is a critical integrity failure.

## Primary threats

- QR-code cloning or visual replacement;
- malicious URL substitution;
- forged certificates or product records;
- unauthorized issuer access;
- stolen platform or issuer credentials;
- unauthorized record mutation or revocation;
- signature or public-key substitution;
- replay of stale verification responses;
- registry enumeration and privacy leakage;
- SQL injection;
- denial of service;
- compromised deployment credentials;
- silent fallback to demo data;
- split-brain state between multiple writable datastores;
- legacy-host routing to an incorrect application.

## Current consolidated controls

The current two-node implementation provides:

- HTTPS-ready server separation;
- strict API CORS configuration;
- server-side write authorization;
- HttpOnly issuer session cookies on the platform node;
- issuer writes proxied server-to-server;
- parameterized SQL;
- public verification rate limiting;
- health/readiness separation;
- create, verify, and revoke audit events;
- deterministic public baseline states: `VERIFIED`, `REVOKED`, `EXPIRED`, `NOT_FOUND`;
- fail-closed issuer access when required secrets are missing;
- HTTP 308 legacy-host redirects on the platform node.

## Required hardening before enterprise multi-tenant launch

The consolidated `/issuer` access-code flow is suitable only as a controlled pilot gate. Replace or extend it before multi-tenant commercial deployment with:

- individual issuer accounts;
- Argon2id or bcrypt password hashing;
- MFA for privileged users;
- issuer-scoped RBAC;
- issuer-scoped API keys;
- session rotation and revocation;
- login throttling and lockout policy;
- organization/team boundaries;
- administrative approval workflow;
- audit attribution to individual actors.

## Cryptographic requirements

QRVP-1 requires:

- canonical JSON serialization;
- SHA-256 record hashing;
- Ed25519 issuer signatures;
- private keys stored outside repositories and logs;
- issuer public-key versioning and rotation;
- deterministic signature validation during verification.

SHA-256 integrity support is active in the current consolidated implementation. Full Ed25519 issuer signing remains a production gate.

Do not claim full issuer-signed QRVP-1 cryptographic verification compliance until all of the following are operational end-to-end:

```text
issuer key generation / custody
→ record signing
→ signature persistence
→ issuer public-key lookup
→ signature verification
→ key rotation/versioning
→ INVALID_SIGNATURE fail-closed handling
```

## Production verification-state target

The current baseline implementation returns:

```text
VERIFIED
REVOKED
EXPIRED
NOT_FOUND
```

The production security target must also distinguish, where applicable:

```text
INVALID_FORMAT
INVALID_SIGNATURE
SUSPENDED_ISSUER
UNAVAILABLE
```

Dependency failures must never be mislabeled as `NOT_FOUND` or `VERIFIED`.

## Availability and fail-safe behavior

- `/healthz` confirms process health without requiring the registry datastore.
- `/readyz` confirms required dependencies and canonical registry access.
- dependency failure must not return `VERIFIED` or `NOT_FOUND`;
- revoked state must not be served from stale cache;
- public pages must not expose stack traces or secrets;
- overall public status must not claim full operational readiness when the API node is misrouted, unavailable, or failing acceptance.

## Database security

The historical deployment record identified temporary `0.0.0.0/0` database access. That must not remain as the long-term production rule. Restrict ingress to approved infrastructure where feasible, require TLS, rotate credentials after network changes, maintain tested backups, and test restoration.

## Audit events

At minimum:

```text
CREATE
VERIFY
REVOKE
issuer_login
issuer_login_failed
api_key_used
admin_override
signing_key_rotated
```

Audit records should include request ID, actor, issuer, QRVID, operation, result, UTC timestamp, and structured metadata where applicable.

## Security release gate

Do not claim full production security readiness until:

- `api.qrv.network` is mapped to the canonical API application and passes `/healthz` and `/readyz`;
- multi-tenant issuer authentication is implemented for general enterprise use;
- issuer-scoped authorization tests pass;
- Ed25519 signature validation is active;
- invalid signatures fail closed;
- restricted/private data is filtered;
- dependency failure returns a safe unavailable state;
- secrets scanning and dependency audit pass;
- database ingress is hardened;
- backup/restore is tested;
- issue → verify → revoke is fully audit logged;
- legacy subdomains resolve only through canonical redirect behavior.
