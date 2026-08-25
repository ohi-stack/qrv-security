# QR-V™ Security — Commercialization Baseline

Security work for the 30-day revenue sprint is limited to production controls that protect issuer authority, canonical registry state, cryptographic integrity, revocation, and privileged administration.

## Production gates

1. Administrator authentication + MFA.
2. Role-based authorization.
3. Issuer authentication and authorization.
4. Admin/issuer audit logs.
5. SHA-256 integrity validation.
6. Ed25519 key lifecycle, signing, persistence, and verification.
7. Revocation checks that cannot be bypassed by stale cache.
8. API rate limiting and security headers.
9. Restricted production database network access.
10. Server-side payment/entitlement validation.
11. Fail-closed behavior when authorization or registry dependencies fail.

## Ed25519 state rule

Do not report Ed25519 as LIVE merely because a library or keypair exists. It becomes LIVE only when:

```text
issuer key established
→ record signed
→ signature persisted
→ public verification loads issuer public key
→ signature verifies successfully
→ invalid signature fails deterministically
→ audit evidence is retained
```

Until then the state is `PENDING`.

## Admin Dashboard signals

The private admin surface should be able to show:

- signing readiness;
- issuer authentication status;
- rate-limit status;
- suspicious verification events;
- integrity/signature failures;
- privileged audit events;
- registry/database connectivity state.

Do not place private keys or unrestricted platform credentials in browser-readable configuration.
