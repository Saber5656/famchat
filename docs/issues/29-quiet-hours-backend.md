# Issue 29: Quiet hours backend + enforcement

## Summary

Implement per-child quiet-hours schedules: the shared schema and pure
evaluator, guardian schedule management, enforcement across HTTP reads/
writes and WebSocket, the report-creation exemption, and notification
suppression hooks.

## Context

DESIGN §13.5 defines the jsonb schema, space-timezone evaluation, the four
enforcement points, and the safety exemption (children can always report).
This activates the `assertNotQuiet` and `wsQuietGate` hooks stubbed in
14/15/19.

## Scope

In scope: shared schema + `isQuietNow`, `guardian.setQuietHours`, child
"state" endpoint, HTTP + WS enforcement, exemption list, suppression flag
for 37, audit, tests.
Out of scope: lock-screen UIs (34/47), notification delivery (37 consumes
the suppression helper).

## Detailed Requirements

1. `packages/shared/src/quietHours.ts`:
   - zod `quietHoursSchema`: `{ enabled: boolean, rules: Array<{ days:
     number[] (1–7 ISO, unique, non-empty), start: "HH:MM", end:
     "HH:MM" }> }`; max 14 rules; start ≠ end.
   - `isQuietNow(qh, timeZone, now: Date): { active: boolean, until?:
     Date }` — computes local weekday + minutes via
     `Intl.DateTimeFormat(…, { timeZone })` parts; window matching where
     `start > end` wraps past midnight (window belongs to its start day);
     `until` = earliest end of any currently-active window (merging
     overlapping/adjacent windows correctly);
   - unit tests: JST evening window, midnight wrap (21:00–07:00 checked at
     23:59, 00:01, 06:59, 07:00), multi-rule overlap merge, disabled,
     DST transitions using `Europe/London` on change dates, invalid-zone
     guarded upstream by settings validation.
2. `guardian.setQuietHours({ spaceId, childUserId, quietHours })` —
   guardian; validates schema + space timezone exists; writes
   `child_settings.quiet_hours`; audit `quiet_hours.update`; WS
   `quietHours.state` recomputed and pushed to the child's sockets; notify
   event `quiet_hours.updated` to the child (delivered post-lock per §14.2).
3. `auth.quietState()` (child sessions) → `{ active, until? }` for the
   child's space — used by lock screens (34/47) and cheap to poll.
4. HTTP enforcement — implement `assertNotQuiet(ctx)` consulted by the
   `spaceProcedure` pipeline for **child memberships only**:
   - Blocked when active: `messages.*` (list/around/send*, delete),
     `rooms.*` reads and writes, `board.*` all, `receipts.*`,
     `attachments.*`, `moderation`/`guardian` n/a (children lack
     permission anyway).
   - **Exempt allowlist** (explicit constant `QUIET_EXEMPT_PROCEDURES`):
     `reports.create`, `auth.me`, `auth.quietState`, `auth.logout`,
     `auth.verifyChildPin`, `notifications.feed/markRead/unreadCounts`,
     `auth.updateProfile` (locale/avatar remain usable — low-risk,
     documented), `spaces.list/get`.
   - Error: `QUIET_HOURS_ACTIVE` with `details.until` (ISO string).
   - Unskip/activate the 28 exemption test.
5. WS enforcement (activate 15's `wsQuietGate`): on subscribeRoom during
   quiet → refuse with `QUIET_HOURS_ACTIVE`; on transition into quiet
   (see scheduling below) → server emits `quietHours.state {active:true,
   until}` to the child's sockets and forcibly unsubscribes their room
   channels; on transition out → emit `{active:false}` (clients refetch).
6. Transition scheduling: a lightweight in-process scheduler in the api
   (`setTimeout` per connected child socket set, re-armed on connect/
   settings change, cluster-safe because each instance handles its own
   sockets); plus a safety net: every enforcement check recomputes from
   the source of truth (no cached "unlocked" state).
7. Suppression helper for 37: `isChildQuietNow(userId)` service used by
   the notification fanout to skip push (in-app rows still written) —
   export + unit-test now.
8. Tests: enforcement matrix (blocked procedure sample × child ⇒
   `QUIET_HOURS_ACTIVE` with until; same procedures fine for adult/
   guardian and for the same child outside the window — injected clock);
   full exemption allowlist verified procedure-by-procedure (table test);
   WS refusal + transition kick (fake timers); settings update mid-
   connection re-arms; schedule editor validation rejects malformed rules;
   cross-tenant (guardian of space A cannot set for child of space B).

## Acceptance Criteria

- [ ] `isQuietNow` passes all timezone/midnight/DST unit cases.
- [ ] Enforcement + exemption tables enforced exactly (table tests mirror
      the constants).
- [ ] WS transitions push state and tear down subscriptions.
- [ ] Reports remain possible during quiet hours (proven).

## Validation

```bash
pnpm --filter @famchat/shared test -- --grep quiet
pnpm --filter @famchat/api test -- --grep quiet
```

## Dependencies

09 (child_settings), 13 (procedures to gate), 15 (WS), 28 (exemption
cross-ref).

## Non-goals

Lock-screen UIs (34/47), per-room schedules, screen-time totals/limits
(v2), guardian bypass grants (v2).

## Design References

- DESIGN §13.5 (schema, evaluation, enforcement points), §9.1 (WS quiet
  behavior), §14.2 (suppression)
