# QR-V™ Security

Security architecture, threat models, production-hardening requirements, issuer trust, and cryptographic verification standards for the QR-V™ Global Verification Network.

## Production trust boundary

The consolidated runtime uses two active nodes:

```text
qrv.network
  public platform, verification UI, issuer UI, docs, registry UI
      ↓ server-to-server
api.qrv.network
  API, PostgreSQL access, registry mutation, verification state, audit
      ↓
PostgreSQL / Google Cloud SQL
```

### Mandatory boundary rules

- `qrv.network` must not receive `DATABASE_URL`.
- `api.qrv.network` is the only application node with database credentials.
- `QRV_PLATFORM_API_KEY` is server-to-server and must never be exposed to browser JavaScript.
- Write operations fail closed when authorization is missing.
- CORS on the API should allow only explicitly approved origins, starting with `https://qrv.network`.
- Legacy subdomains are compatibility redirects, not independent trust boundaries.

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
- silent fallback to demo data.

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
- deterministic public states: `VERIFIED`, `REVOKED`, `EXPIRED`, `NOT_FOUND`;
- fail-closed issuer access when required secrets are missing.

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

The consolidated API currently exposes hash presence and does **not** claim successful signature validation unless signing keys are configured. Full Ed25519 issuer signing remains a production-hardening gate and must be completed before claiming full QRVP-1 cryptographic verification compliance.

## Availability and fail-safe behavior

- `/healthz` confirms process health without requiring PostgreSQL.
- `/readyz` confirms required dependencies.
- dependency failure must not return `VERIFIED` or `NOT_FOUND`;
- revoked state must not be served from stale cache;
- public pages must not expose stack traces or secrets.

## Database security

The historical deployment record identified temporary `0.0.0.0/0` database access. That must not remain as the long-term production rule. Restrict ingress to approved infrastructure where feasible, require TLS, rotate credentials after network changes, and maintain tested backups.

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

- multi-tenant issuer authentication is implemented;
- issuer-scoped authorization tests pass;
- Ed25519 signature validation is active;
- invalid signatures fail closed;
- restricted/private data is filtered;
- dependency failure returns a safe unavailable state;
- secrets scanning and dependency audit pass;
- database ingress is hardened;
- issue → verify → revoke is fully audit logged.
