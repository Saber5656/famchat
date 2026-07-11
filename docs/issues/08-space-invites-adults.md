# Issue 08: Space invites for adults/guardians

## Summary

Implement guardian-issued space invites (one-time, 72 h, role-scoped codes
rendered as link + QR payload) and the accept flow that signs up or logs in
the recipient and creates their membership.

## Context

DESIGN §5.2 row 2. This is the only way adults join an existing space
(ADR-006). Invite codes are credential-equivalent secrets: hashed at rest,
short-lived, single-use by default (DESIGN §19.3, §19.6).

## Scope

In scope: `invites` router (create/list/revoke/preview/accept), code format,
membership creation, `member.joined` notification event, audit, tests.
Out of scope: child accounts (09), instance invites (07), invite email
sending (links are shared via existing family channels by design), UI (32).

## Detailed Requirements

1. Code format `si_<base64url 16 bytes>`; store `sha256`; TTL fixed 72 h
   (`INVITE_TTL_HOURS`); `max_uses` fixed 1 in v1 (schema supports more).
2. `invites.createSpaceInvite({ spaceId, role: 'adult'|'guardian' })` —
   guardian-only (local check until 10). Returns `{ inviteId, code, url:
   `${APP_BASE_URL}/invite/<code>`, expiresAt }` — the raw code is returned
   exactly once. Cap: ≤ 10 active invites per space
   (`VALIDATION_FAILED` beyond).
3. `invites.listSpaceInvites({ spaceId })` — guardian-only; shows status
   (active/used/expired/revoked), role, creator, expiry — never the code.
4. `invites.revokeSpaceInvite({ spaceId, inviteId })` — guardian-only.
5. `invites.previewInvite({ code })` — **unauthenticated**; returns
   `{ spaceName, role, valid: true }` or `INVITE_INVALID_OR_EXPIRED`; rate
   limited 5/15 min/IP; no other space metadata leaks.
6. `invites.acceptSpaceInvite({ code, newUser?: { email, password,
   displayName, locale, tosAccepted: true } })` — logged-out requires
   `newUser` (same registration rules as 07 incl. consent timestamp);
   logged-in adult joins directly. Atomic consume (same race guard pattern
   as 07). Rejections: already a member (`VALIDATION_FAILED` with
   `already_member` detail), child accounts cannot accept, suspended space.
   Creates membership with the invite's role; emits notify event
   `member.joined` (stub) to space members; audit `invite.create/revoke/
   accept`, `member` metadata.
7. Guardian-invite hygiene: accepting a `guardian` invite grants guardian
   role but **not** ownership.
8. Integration tests: full accept happy paths (new + existing user); expiry
   honored (time-travel via injected clock or direct row edit); one-time
   enforcement under parallel accepts; preview leaks nothing on invalid;
   revoked cannot be accepted; already-member rejection; cross-space code
   cannot be replayed elsewhere (code is bound to its space row); audit rows.

## Acceptance Criteria

- [ ] Grandparent flow works end-to-end: guardian creates invite → new user
      accepts with signup → membership role `adult` active.
- [ ] Codes appear only once in any API response; stored hashed; absent
      from logs.
- [ ] Parallel double-accept admits exactly one membership.
- [ ] Preview is rate-limited and oracle-free.

## Validation

```bash
pnpm --filter @famchat/api test -- --grep invites
```

## Dependencies

07.

## Non-goals

Multi-use invites, invite email delivery, QR rendering (client issues 32/
20), child links (09).

## Design References

- DESIGN §5.2 (flows), §7.1 (space_invites), §8.2 (`invites` router),
  §19.3 (brute-force mitigations); ADR-006
