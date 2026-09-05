# Issue 06: Guardian/adult auth — login, sessions, password reset

## Summary

Implement adult authentication: argon2id password verification, DB-backed
sessions delivered as cookie (web) or bearer token (mobile), logout /
logout-everywhere, and email password reset — replacing the issue-05 session
stub.

## Context

DESIGN §6 defines the auth model (ADR-008: custom sessions, Lucia-style
pattern; children authenticate via issue 09). Registration itself happens
inside invite flows (07, 08); this issue covers credentials + session
lifecycle for existing users.

## Scope

In scope: password hashing service, session service, `auth` router (login,
logout, logoutAll, requestPasswordReset, resetPassword, me), reset email
(ja/en), cookie handling, per-route rate limits, audit events, tests.
Out of scope: registration (07/08), child link (09), TOTP (v2).

## Detailed Requirements

1. `src/services/password.ts`: `hashPassword`/`verifyPassword` using
   `@node-rs/argon2` argon2id (m=19456 KiB, t=2, p=1); policy validator
   `assertPasswordAcceptable` (≥ 10 chars AND zxcvbn score ≥ 3 via
   `@zxcvbn-ts/core`) returning `VALIDATION_FAILED` details on failure.
2. `src/services/session.ts`:
   - `createSession(userId, kind, req)` → generates 32 random bytes
     (base64url token), stores `sha256(token)` + kind + expiry from
     `RATE_LIMITS`/TTL constants (web 30d, mobile 90d, device_link 180d,
     sliding), returns raw token once.
   - `resolveSession(req)` (replaces issue-05 stub): reads cookie
     `famchat_session` or `Authorization: Bearer`; looks up by hash;
     rejects expired/revoked; sliding renewal when > 24 h since
     `lastUsedAt`; loads user (status must be `active`) and active
     memberships into context.
   - `revokeSession(id)`, `revokeAllForUser(userId)`; emits WS
     `session.revoked` when issue 15 lands (call site behind the notify/ws
     abstraction — leave a typed hook `onSessionRevoked`).
3. Cookie handling (@fastify/cookie): set/clear helpers with
   `HttpOnly; Secure (production); SameSite=Lax; Path=/`; cookie value is
   the raw token (opaque), signed cookies not required. Login **rotates**:
   always issues a new session row and cookie.
4. `auth` router procedures (all with zod IO in `packages/shared/src/api/auth.ts`):
   - `login({ email, password, client: 'web'|'mobile' })` → session token
     (set-cookie for web; token in payload for mobile). Constant-shape
     failure `AUTH_INVALID_CREDENTIALS` (no user-exists oracle; verify
     against a dummy hash when user missing).
   - `logout()`, `logoutAll()`.
   - `requestPasswordReset({ email })` → always returns ok (no oracle);
     when user exists: token (32 bytes, sha256 stored, 30-min expiry,
     single-use), email via nodemailer/`SMTP_URL` using i18next server
     rendering in the user's locale, link `${APP_BASE_URL}/reset?token=…`.
   - `resetPassword({ token, newPassword })` → validates, then consumes
     the token **atomically** (`UPDATE password_resets SET used_at = now()
     WHERE token_hash = $h AND used_at IS NULL AND expires_at > now()`
     returning-row guard — zero rows ⇒ `INVITE_INVALID_OR_EXPIRED`-class
     failure) so concurrent reuse is impossible; updates hash;
     `revokeAllForUser`.
   - `me()` → `{ user, memberships }` DTO (no hashes/emails of others).
5. Rate limits (constants from `@famchat/shared/limits`, enforced with the
   issue-05-compatible fastify rate-limit plugin scoped to these routes now;
   issue 12 centralizes): login & reset-request 5/15 min/IP + 10/h/account.
6. Audit events: `auth.login`, `auth.login_failed` (with IP, no password),
   `auth.password_reset` — emitted through a thin
   `src/services/audit.ts` **interface created in this issue** whose only
   implementation here is a debug-log stub (issue 11 adds persistence and
   re-asserts these events as DB rows). Tests in THIS issue assert emission
   by spying on the interface, not by querying a table.
7. Email templates: `packages/i18n/locales/{ja,en}/auth.json` keys
   (`reset.subject`, `reset.body`) — plain-text email, no HTML.
8. Integration tests: login success sets cookie flags exactly; wrong
   password → `AUTH_INVALID_CREDENTIALS` and audit emission (spy); unknown email same
   error shape/time-class; session expiry honored; sliding renewal updates
   `expiresAt`; logoutAll invalidates other sessions; reset flow end-to-end
   via Mailpit API (`GET :8025/api/v1/messages`) including token single-use
   and session revocation; parallel double-consume of one reset token
   succeeds exactly once; suspended user cannot log in; rate limit trips on
   6th attempt.

## Acceptance Criteria

- [ ] All procedures behave per DESIGN §6.1/§6.3 including rotation,
      sliding expiry, and no-oracle responses.
- [ ] Cookie attributes verified by test (`HttpOnly`, `SameSite=Lax`,
      `Secure` in production build flag).
- [ ] Reset email arrives in Mailpit in both locales (test both).
- [ ] Tokens/passwords never appear in logs (redaction test).

## Validation

```bash
pnpm --filter @famchat/api test -- --grep auth
# manual: pnpm --filter @famchat/api dev; curl login; check Mailpit :8025
```

## Dependencies

05 (skeleton), 04 (schema). Audit persistence completes in 11.

## Non-goals

Registration, child auth, TOTP, passkeys, account lockout beyond rate
limits, email change flows (v2).

## Design References

- DESIGN §6.1–6.3 (auth/sessions), §8.2 (`auth` router), §13.8 (audit),
  §19.5 (limits); ADR-008
