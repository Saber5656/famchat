# Issue 07: Instance invite gate + space creation

## Summary

Implement the closed-beta gate: operator-issued instance invite codes and the
`spaces.create` flow that (optionally) registers the founding guardian,
creates the space, and grants the owner-guardian membership.

## Context

ADR-006: there is **no public signup surface at all** — the only way a user
account is born is (a) founding a space with an instance invite (this
issue) or (b) accepting a space invite (issue 08) or (c) being created as a
child (issue 09). DESIGN §5.2 row 1.

## Scope

In scope: instance-invite validation, combined signup+space-create
procedure, `spaces.list/get`, minimal `spaces.updateSettings` (name only —
safety settings arrive in 30), consent timestamp capture, audit, tests.
Out of scope: invite issuance API/CLI (35), space invites (08), deletion
lifecycle (36), safety settings (30).

## Detailed Requirements

1. Instance invite verification service: input code (format
   `fi_<base64url 16 bytes>`); lookup by `sha256(code)`; valid iff not
   revoked, not expired, `used_count < max_uses`; consume atomically
   (`UPDATE … WHERE used_count < max_uses RETURNING` guard). Errors map to
   `INVITE_INVALID_OR_EXPIRED` (single code for all failure modes — no
   validity oracle).
2. `spaces.create` (public procedure, works logged-in or logged-out;
   rate-limited 5/15 min/IP route-locally in this issue — it validates
   invite codes, so it gets the invite-class limit from DESIGN §19.5;
   issue 12 centralizes):
   input `{ instanceInviteCode, spaceName (1–40), timezone? (IANA,
   validated against `Intl.supportedValuesOf('timeZone')`, default env),
   locale?: 'ja'|'en', newUser?: { email, password, displayName,
   locale: 'ja'|'en', tosAccepted: true } }` (locale enums pinned to
   DESIGN §7.1's `ja|en` everywhere).
   - **Atomicity: invite consume + user create (when logged-out) + space
     create + owner membership + session row commit in ONE transaction**;
     any failure rolls back all of it (the invite is not burned on a
     failed signup).
   - Logged-out ⇒ `newUser` required: validate password policy (06),
     unique email → create `users` row (kind adult) with
     `tosAcceptedAt = now()` (column added by this issue's migration).
   - Logged-in adult ⇒ ignore `newUser`.
   - Create space (status active, defaults per DESIGN §7.1) + membership
     (role guardian, `isOwner: true`).
   - Output DTO (schema in `packages/shared/src/api/spaces.ts`):
     `{ space: SpaceDTO, membership: MembershipDTO, sessionToken? }` where
     `SpaceDTO = { id, name, timezone, defaultLocale, moderationMode,
     ngBuiltinJa, ngBuiltinEn, status, createdAt }` and `MembershipDTO =
     { spaceId, userId, role, isOwner, status }` — list/get reuse these.
   - Family-room creation is issue 13; call a `onSpaceCreated(space)` hook
     (no-op now) so 13 plugs in without editing this flow.
3. Migration: add `tosAcceptedAt DateTime?` to `User` (used by 56's consent
   flow; captured from day one).
4. `spaces.list` → my active memberships with space summaries + my role;
   `spaces.get({ spaceId })` → member-only detail (settings visible to all
   members; mutable only per matrix later).
5. `spaces.updateSettings({ spaceId, name })` — guardian-gated by a local
   check until issue 10's middleware replaces it (leave `TODO(issue-10)`
   marker the 10 issue removes).
6. Audit: `space.create`, `invite.accept` (instance kind, metadata
   `{ inviteId }`) — emitted via the audit interface from 06 (stub until
   11 persists; tests spy on the interface).
7. Suspended space guard: a shared `assertSpaceActive(space)` helper
   returning `SPACE_SUSPENDED` — applied in `spaces.get` now and reused by
   every space-scoped procedure later.
8. Integration tests: happy path logged-out (user+space+owner membership+
   session in one call); happy path logged-in existing adult; exhausted /
   expired / revoked / malformed codes all return
   `INVITE_INVALID_OR_EXPIRED`; concurrent double-spend of a max_uses=1
   code admits exactly one (parallel test); rollback test (forced failure
   after invite consume leaves the invite unburned); child users cannot
   call `spaces.create` (kind check); consent timestamp stored;
   `spaces.get` denied for non-members (cross-tenant helper pattern —
   formalized in 10 but written here directly); `updateSettings` denied
   for non-guardian member; suspended space returns `SPACE_SUSPENDED` on
   `spaces.get`; rate limit trips on the 6th create attempt per IP.

## Acceptance Criteria

- [ ] A fresh user can go from nothing → account + space + owner-guardian
      membership + session with one procedure call and a valid code.
- [ ] All invalid-code paths return the single opaque error.
- [ ] Double-spend race admits exactly one creation (proven by test).
- [ ] Audit rows written with actor + metadata.

## Validation

```bash
pnpm --filter @famchat/db db:migrate && pnpm --filter @famchat/api test -- --grep "spaces.create|instance"
```

## Dependencies

06. (Invite issuance for real operators arrives in 35; tests insert invite
rows directly via db.)

## Non-goals

Operator CLI (35), space invites (08), family room (13), full settings (30),
deletion (36), billing (never in v1).

## Design References

- DESIGN §5.1–5.2 (tenancy flows), §7.1 (schema), §8.2 (`spaces` router);
  ADR-002, ADR-006
