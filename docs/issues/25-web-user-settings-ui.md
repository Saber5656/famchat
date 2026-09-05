# Issue 25: Web user settings UI

## Summary

Build `/settings`: profile (display name, preset avatar, locale), password
change, active-session list with revocation and logout-everywhere, and the
child-appropriate reduced variant.

## Context

DESIGN §16 (routes) and §6.3 (session management is a user-facing security
control — ADR-003 compensating transparency). Avatars are presets only
(DESIGN §2.2 non-goal: uploads).

## Scope

In scope: settings routes + forms above, `auth.updateProfile` +
`auth.changePassword` + `auth.listSessions` API additions, child variant.
Out of scope: push toggles (38/39), space-level settings (32), account
deletion (36 handles lifecycle; no self-serve delete in v1 — guardians/
operator mediate; document).

## Detailed Requirements

1. API additions (this issue, in `apps/api` — four procedures plus one
   micro-procedure):
   - `auth.updateProfile({ displayName?, avatarPreset?, locale? })` —
     **role-branched**: adults may change all three; child sessions may
     change only `avatarPreset`/`locale` — a `displayName` field from a
     child is rejected `PERMISSION_DENIED` (child display names are
     guardian-managed via `children.update`, DESIGN §13.1). Adult
     displayName runs the moderation hook (pass-through until 27;
     blocked names surface `CONTENT_BLOCKED_NG_WORD` once 27 lands — UI
     handles the code now); length/preset validation from shared
     constants.
   - `auth.updateLocale({ locale })` — any authed user; the minimal
     subset needed by quiet-hours lock screens; listed in 29's
     quiet-exempt allowlist (coordination note in both issues).
   - `auth.changePassword({ currentPassword, newPassword })` — adults;
     verifies current (failures rate-limited 10/h/account reusing the
     login-class policy — stolen-session hardening), applies policy
     (06), revokes **other** sessions, audit `auth.password_reset`
     metadata `{ via: 'change' }`.
   - `auth.listSessions()` → own sessions: **id**, kind, createdAt,
     lastUsedAt, truncated user-agent, `current` flag (id is required by
     revocation; never token material); `auth.revokeSession({ sessionId })`
     (own, not current — use logout for that); both `revokeSession` and
     `logoutAll` emit audit `session.revoke` with metadata
     `{ sessionId, kind }` only.
2. `/settings` UI (adult): profile card (display name inline edit with
   30-char counter; avatar preset grid from `AVATAR_PRESETS` with selected
   ring; locale radio ja/en applying instantly via i18next + persisted);
   security card (change-password form with policy meter reusing zxcvbn
   feedback; sessions table with per-row revoke + logout-all confirm;
   current session highlighted); all mutations optimistic-with-rollback.
3. Child variant (device-link sessions): avatar preset grid + locale only;
   friendly copy with furigana; no password/session cards (device
   management is the guardian's, per ADR-008) — but show "この きき は
   おうちの人が かんり しています" transparency note.
4. Locale change propagates: i18next switches immediately; server stores
   `users.locale` (notification language per DESIGN §14.1).
5. Tests: API — profile validation, password-change session revocation
   semantics (current survives, others die), session list shape excludes
   token material; UI component — avatar grid selection, locale instant
   switch, child variant gating; Playwright: change locale → UI flips →
   reload persists; revoke other session → that browser context gets 401
   on next call.

## Acceptance Criteria

- [ ] All five API procedures implemented with the annotation/permission
      registry (10) and audit where specified (incl. `session.revoke`
      rows).
- [ ] Session revocation UX verifiably kills the other session
      (Playwright two-context proof).
- [ ] Child variant shows exactly avatar+locale plus the transparency
      note; adult variant complete.
- [ ] ja/en complete; no token/hash data in any DTO (snapshot test).

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/api test -- -t "updateProfile"
pnpm --filter @famchat/api test -- -t "changePassword"
pnpm --filter @famchat/api test -- -t "listSessions"
pnpm --filter @famchat/api test -- -t "revokeSession"
pnpm --filter @famchat/web exec playwright test --grep @settings
```

## Dependencies

20 (web shell), 10 (authz annotations), 11 (audit persistence).
(Moderation of display names activates with 27.)

## Non-goals

Push preferences (38/39), email change, account self-deletion, TOTP (v2),
avatar uploads (v2).

## Design References

- DESIGN §16 (web), §6.3 (sessions), §5.3/§13.1 (roles), §14.1 (locale
  usage); ADR-008
