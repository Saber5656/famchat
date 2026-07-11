# Issue 09: Child accounts + QR device link

## Summary

Implement guardian-managed child accounts (no email/password) and the
device-link authentication flow: guardian generates a one-time 6-digit/QR
code, the child device redeems it for a long-lived revocable device session;
guardians manage devices and an optional PIN.

## Context

DESIGN §6.2 and ADR-008. Children 6–12 cannot manage credentials; the trust
anchor is the guardian's ability to see and revoke device sessions, with
guardian notification on every successful link.

## Scope

In scope: `children` router (create/update/list/remove/createLinkCode/
listDevices/revokeDevice/setPin), `auth.childLink`, link-code service,
guardian notifications (stub), audit, rate limits, tests.
Out of scope: quiet hours (29), client UIs (20/32/42), PIN verification UX
(25/42).

## Detailed Requirements

1. `children.create({ spaceId, displayName, birthYear (currentYear-15 …
   currentYear-3), avatarPreset, locale })` — guardian-only. Creates `users`
   row (kind child, no email/password) + membership (role child) +
   `child_settings` row in one transaction. v1 rule: a child user has
   exactly one membership ever (enforced here; DESIGN §5.1).
2. `children.update` (displayName/avatarPreset/locale/birthYear),
   `children.list({ spaceId })` (guardian-only; includes settings summary +
   device count), `children.remove({ spaceId, childUserId, purgeContent:
   boolean (default false) })` — membership → removed, child user status
   → **deleted** (canonical rule: guardian removal of a child ends the
   account, since v1 children have exactly one space; `suspended`
   remains operator-only per DESIGN §7.1), all sessions revoked;
   `purgeContent: true` enqueues the child-content-purge job specified
   in 36 (until 36 lands, the flag is accepted and the enqueue is a
   typed no-op with `TODO(issue-36)`).
3. Link codes: `children.createLinkCode({ spaceId, childUserId })` —
   guardian-only; generates 6-digit numeric code (crypto-random,
   zero-padded); stores sha256; TTL 10 min (`CHILD_LINK_CODE_TTL_MIN`);
   one active code per child (creating a new one revokes prior unused
   codes). Response `{ code, qrPayload, expiresAt }` where `qrPayload =
   "${APP_BASE_URL}/link#c=<code>&s=<spaceId>"` per DESIGN §6.2 — the
   code rides in the URL **fragment** so it never reaches server/proxy
   logs; any camera app opens the web page, and the universal link opens
   the mobile app when installed.
4. `auth.childLink({ code, client: 'web'|'mobile' })` — unauthenticated;
   **code alone identifies the link row** (codes are globally unique:
   generation retries on hash collision at insert; the `s=<spaceId>` in
   the QR URL exists only for client routing/display and is never
   required for redemption — a child typing 6 digits manually must
   succeed); verifies code (sha256 lookup with `crypto.timingSafeEqual`
   comparison per DESIGN §19.3, unexpired, unused, child + space
   active); consumes atomically
   (conditional UPDATE guard, same pattern as 06's reset tokens); creates
   session kind `device_link` (180-day sliding); returns token/cookie +
   child user DTO. Rate limit 5/15 min/IP. On success: notify event
   `child.device.linked` to all space guardians — enqueued via the
   issue-05 `NotifyService` interface with the typed `NotifyEvent` union
   member `{ type: 'child.device.linked', spaceId, childUserId,
   sessionId }` (this issue adds the member to the union; 37 delivers) —
   plus audit `child.device_link` with device user-agent.
5. Device management: `children.listDevices({ spaceId, childUserId })` —
   guardian-only; active device_link sessions (created, lastUsed,
   user-agent; never token material) of that child; and
   `children.revokeDevice({ spaceId, childUserId, sessionId })` —
   guardian-only; the session must be kind `device_link`, belong to
   `childUserId`, and that child must have an active membership in
   `spaceId` (explicit ownership check — cross-space or arbitrary-session
   revocation must be impossible, tested). Revocation takes effect on
   next request AND via the `onSessionRevoked` hook (WS kick once 15
   lands).
6. PIN: `children.setPin({ spaceId, childUserId, pin: 4–6 digits | null })`
   — stores argon2id hash in `child_settings.pin_hash`; add
   `auth.verifyChildPin({ pin })` (child-session-only, rate limit 5/15 min/
   session) returning boolean — client app-lock UX consumes it (ADR-008:
   not a security boundary; document in code comment).
7. Audit: `child.create/update/link_code_create/device_link/device_revoke/
   pin_set/remove`.
8. Integration tests: full link flow happy path; expired/used/foreign-space
   codes fail identically; parallel double-redeem of one code admits one
   session; new code revokes old; revoked device session is dead;
   revokeDevice rejects sessions of another child / another space / non-
   device_link kinds; guardians notified (NotifyService spy); child cannot
   call guardian-only procedures; PIN verify rate-limits; child users
   cannot log in via `auth.login` (no credentials) — explicit test.

## Acceptance Criteria

- [ ] Guardian → link code → child device session end-to-end in tests for
      web and mobile client kinds.
- [ ] All guardian notifications + audit rows emitted.
- [ ] Codes single-use, 10-min TTL, hashed at rest, rate-limited, absent
      from logs.
- [ ] Device revocation immediately invalidates the session.

## Validation

```bash
pnpm --filter @famchat/api test -- --grep "children|childLink"
```

## Dependencies

07. (WS kick completes when 15 lands; notification delivery when 37 lands.)

## Non-goals

Quiet hours (29), avatar uploads (v2 — presets only), multi-space children
(v2), client link UIs.

## Design References

- DESIGN §5.2 (child flow), §6.2 (device link), §7.1 (child tables), §13.8
  (audit), §19.3 (abuse cases); ADR-008
