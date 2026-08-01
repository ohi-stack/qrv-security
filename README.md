# QR-V™ Security

Security architecture, threat models, production-hardening requirements, responsible disclosure, audit controls, issuer trust, and cryptographic verification standards for the QR-V™ Global Verification Network.

## Security Objectives

QR-V must preserve:

- authenticity of registry-backed records;
- integrity of canonical payloads;
- authorization of issuers and operators;
- confidentiality of restricted and private records;
- availability of verification services;
- traceability of issuance, verification, mutation, and revocation events.

## Primary Threats

- QR-code cloning or visual replacement;
- malicious URL substitution;
- forged certificates or product records;
- unauthorized issuer creation;
- stolen issuer API keys or sessions;
- unauthorized record mutation or revocation;
- signature or public-key substitution;
- replay of stale verification responses;
- registry enumeration and privacy leakage;
- SQL injection and unsafe query composition;
- denial of service and abusive scanning;
- compromised deployment credentials;
- public/private environment confusion;
- silent fallback to demo or cached data.

## Mandatory Production Controls

### Identity and Access

- JWT or secure server-side sessions for authenticated portals.
- API keys scoped to one issuer and specific operations.
- Role-based access control for platform and issuer users.
- Multi-factor authentication for platform administrators.
- Immediate token and key revocation capability.
- Separate production, staging, and development credentials.

### Registry and API

- Parameterized SQL only.
- Schema validation for every external request.
- Idempotency keys for issuance and other retryable mutations.
- Optimistic or explicit concurrency controls for lifecycle changes.
- Append-oriented audit logs.
- Strict CORS allowlist.
- Rate limiting by IP, issuer, API key, and operation.
- Request IDs propagated across services.

### Cryptography

- SHA-256 for canonical record hashing.
- Ed25519 for issuer signatures.
- Canonical JSON serialization before hashing and signing.
- Private keys stored outside repositories and application logs.
- Public-key versioning and rotation support.
- Verification results must identify the key version used.
- Signature or hash failure must never return `VERIFIED`.

### Availability and Fail-Safe Behavior

- `/healthz` confirms process health without requiring the database.
- `/readyz` confirms required dependencies.
- Dependency failure returns `UNAVAILABLE`, never `NOT_FOUND` or `VERIFIED`.
- Revocation and issuer-status checks must not rely on stale cache.
- Public pages must never expose raw stack traces.

### Data Protection

QRVP-1 privacy modes:

- `public` — approved public fields;
- `restricted` — reduced metadata;
- `private` — validity result and minimum issuer/record context only.

Logs must not contain private payloads, secrets, full tokens, or signing keys.

## Deterministic Security States

```text
VERIFIED
REVOKED
EXPIRED
NOT_FOUND
INVALID_FORMAT
INVALID_SIGNATURE
SUSPENDED_ISSUER
UNAVAILABLE
```

## Critical Infrastructure Correction

The historical deployment record identified temporary database access using `0.0.0.0/0`. This must not remain in production. Restrict database ingress to approved application infrastructure, use TLS, rotate credentials after access changes, and prefer private or controlled connectivity where operationally available.

## Audit Events

At minimum, record:

- `issuer_login`
- `issuer_login_failed`
- `issuer_created`
- `issuer_approved`
- `issuer_suspended`
- `registry_create`
- `registry_verify`
- `registry_update`
- `registry_revoke`
- `api_key_created`
- `api_key_revoked`
- `signing_key_rotated`
- `admin_override`

## Responsible Disclosure

Security reports should include:

- affected service and URL;
- reproduction steps;
- expected and observed behavior;
- potential impact;
- request IDs or timestamps;
- proof that avoids accessing unrelated records.

Do not include live secrets, unnecessary personal data, destructive exploit payloads, or public disclosure before a reasonable remediation window.

## Security Release Gate

A release cannot be marked production-ready until:

- authentication and authorization tests pass;
- invalid signatures fail closed;
- private fields are filtered;
- revoked records cannot be cached as verified;
- rate limiting is active;
- dependency failure returns a safe unavailable state;
- secrets scanning and dependency audit pass;
- backup and rollback procedures are documented;
- the complete issue → verify → revoke lifecycle is audit logged.
