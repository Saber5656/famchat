# Issue 47: Mobile safety surfaces (lock screen, report, guardian tab)

## Summary

Complete the mobile safety experience: the child quiet-hours lock screen,
the report flow on chat/board content, the Guardian tab (queues, child
overview, on-the-go device linking), and the child settings variant —
reaching safety parity with web (31–34).

## Context

DESIGN §13 is the product's core; mobile is the primary child device, so
lock and report must be native-quality. Web issues 31–34 are the
reference implementations; backend is complete.

## Scope

In scope: lock screen + transitions, report dialog integration, Guardian
tab (dashboard-lite, flag/report queues, child overview + link-code
generation, quiet-hours editor), child settings variant, kid-UI polish
pass.
Out of scope: space admin parity (members/invites/settings stay
web-only in v1 — guardians are pointed to web for those; documented),
operator anything.

## Detailed Requirements

1. Lock screen (per 34's spec adapted native): full-screen takeover on
   `quietHours.state {active}` WS event, `auth.quietState` bootstrap,
   or any `QUIET_HOURS_ACTIVE` error; suppresses all tabs + deep links;
   countdown, unlock time in space TZ, friendly kid copy with furigana;
   auto-unlock (timer + WS + refetch); locale toggle accessible;
   AppState-safe (recompute on foreground).
2. Report flow: port/wrap the shared dialog spec from 33 (child
   icon-variant, adult note-variant, reassurance success) as a native
   component; entry points: message long-press action sheet (43), board
   post/comment menus (45), member profile sheet; identical privacy/
   idempotency behavior; never shown on own content.
3. Guardian tab (guardian memberships only; hidden otherwise — 41's
   gate). Scope per DESIGN §13.2's v1 platform split: monitoring +
   child management only (member/invite/space-settings stay web-only).
   Concrete routes (expo-router group `(app)/guardian/`):
   - `guardian/index` dashboard-lite: `guardian.dashboard` DTO → open
     flags / open reports / children cards with live badges (guardian
     channel events via `useSocketEvent`).
   - `guardian/moderation` + `guardian/reports`: FlashList queues from
     `moderation.flagQueue` / `reports.queue`, detail sheets, actions
     `moderation.resolveHit` / `reports.resolve|dismiss` with
     optimistic rollback per 31's semantics; pull-to-refresh; live
     content DTO / tombstone rendering + `routeFromLink`-based deep
     links into rooms/board.
   - `guardian/children/[childUserId]`: 30's `childOverview` DTO —
     devices (`children.revokeDevice` with confirm), rooms, quiet-hours
     summary, recent items; **link a new device**:
     `children.createLinkCode` → full-screen 6-digit code + QR of the
     backend qrPayload with countdown/regenerate (mirrors 32 — the
     on-the-go "add grandma's old tablet at her house" flow).
   - `guardian/children/[childUserId]/quiet-hours`: native port of 34's
     editor (day chips, native time pickers, overnight badge, ≤ 14
     rules, preview strip, `guardian.setQuietHours`) — validation
     parity via the shared schema + 29's fixtures.
4. Child settings variant (extends 42's settings stub): avatar preset
   grid + locale + the transparency note + read-only quiet-hours card
   (34's spec); adult settings: profile fields, sessions list + revoke
   (25 parity), logout.
5. Kid-UI polish pass (this issue owns the audit): all child-reachable
   screens re-checked for ≥ 44 pt targets, type scale, furigana
   coverage (40's checker covers catalogs; this is the layout audit) —
   record a checklist table in the PR (screen × criteria).
6. Tests: component — lock engage/disengage transitions (fake timers +
   mocked WS), report dialog variants, guardian queue actions with
   optimistic rollback, quiet-hours editor validation parity fixtures
   (shared with 29/34); manual device checklist: guardian sets window
   from the device → child device locks < 3 s → unlock at end; report
   from child device → guardian device dashboard badge increments live;
   device-link generation on mobile links a third device.

## Acceptance Criteria

- [ ] Child lock + report + oversight transparency fully functional on
      device (checklist evidence).
- [ ] Guardian can triage flags/reports and manage a child's devices +
      quiet hours entirely from mobile.
- [ ] Editor/validation parity with web proven by shared fixtures.
- [ ] Kid-UI audit table complete and passing.

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/mobile test -- -t lock
pnpm --filter @famchat/mobile test -- -t report
pnpm --filter @famchat/mobile test -- -t guardian
# manual: two-device guardian/child checklist
```

## Dependencies

28, 29, 30 (APIs), 33 (dialog contract), 34 (editor spec), 43 (chat
surfaces), 45 (board surfaces), 46 (routeFromLink for deep links).

## Non-goals

Space admin (members/invites/space settings) on mobile, export/deletion
UI on mobile, biometric guardian lock (v2).

## Design References

- DESIGN §13.2–13.6 (safety surfaces), §17 (mobile), §15 (child
  register)
