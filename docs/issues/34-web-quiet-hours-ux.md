# Issue 34: Web quiet hours UX (lock screen + guardian editor)

## Summary

Build the child-facing quiet-hours lock screen (friendly full-screen state
with countdown and auto-unlock) and the guardian schedule editor, wired to
the 29 backend and its realtime state events.

## Context

DESIGN §13.5 client UX: the lock is calm and predictable ("あさ 7:00 に
あえるよ"), never punitive. Guardians edit per-child weekly windows in the
space timezone.

## Scope

In scope: lock screen overlay + routing behavior, countdown/auto-unlock,
guardian editor screen, timezone display, i18n.
Out of scope: backend (29), mobile lock (47), screen-time reports (v2).

## Detailed Requirements

1. Lock detection: on `auth.quietState` at app bootstrap and on WS
   `quietHours.state`; when active, child sessions render a full-screen
   lock route replacing the app shell (all in-app navigation suppressed;
   deep links land on the lock too). Guardians/adults never see it.
2. Lock screen content (kid register, furigana): moon/star illustration
   placeholder (CSS/emoji-based, no binary assets), 「いまは おやすみの
   じかん」, unlock time in space timezone 「あさ 7:00 に あえるよ」,
   live countdown (hh:mm), and a small settings-free footer (avatar +
   display name so the child knows who's signed in). No bypass
   affordances. Locale toggle remains accessible (exempt procedure).
3. Auto-unlock: timer to `until` + `quietHours.state {active:false}`
   handling → full cache invalidation + shell restore (Playwright-proven
   with a 5-second test window via injected schedule).
4. Race handling: any API `QUIET_HOURS_ACTIVE` error while unlocked
   (clock skew) → flip to lock using `details.until` (global error hook
   from 20 extended here).
5. Guardian editor (`/guardian/children/[id]/quiet-hours`): enable
   toggle; rule rows — day-of-week chip group (Mon–Sun localized), start/
   end time inputs (15-min steps), crossing-midnight visual hint (start >
   end renders a "overnight" badge spanning to next day); add/remove rule
   (≤ 14); "copy to all days" helper; live preview strip: 7×24 mini grid
   shading blocked hours incl. wrap; save via `guardian.setQuietHours`
   with zod client validation mirroring the shared schema; space timezone
   caption ("この予定は Asia/Tokyo 基準です" — from space settings, link
   to 32's settings).
6. Child transparency: when quiet hours are enabled, the child's settings
   page (25) shows a read-only "おやすみ じかん" card (schedule summary,
   friendly) — no surprise locks (DESIGN §13.3 spirit).
7. Tests: component — countdown formatting, grid preview correctness for
   wrap windows (fixture schedules → expected shaded cells), editor
   validation parity with shared schema (same fixtures pass/fail both);
   Playwright: guardian sets a window covering now for a child context →
   child locks in-place via WS (< 3 s), report… (not applicable — lock
   has no report), unlock at window end restores chat; deep link during
   lock lands on lock.

## Acceptance Criteria

- [ ] Lock engages/disengages live via WS and timers without reload; deep
      links respect it; adults unaffected.
- [ ] Editor round-trips schedules exactly (fixture → save → reload →
      identical), including overnight windows.
- [ ] Preview grid matches `isQuietNow` semantics (shared fixtures with
      29's unit tests).
- [ ] Full ja/en with furigana on all child-facing lock strings.

## Validation

```bash
pnpm --filter @famchat/web test -- --grep quiet
pnpm --filter @famchat/web exec playwright test --grep @quiet
```

## Dependencies

29 (backend + events), 31 (console navigation), 25 (child settings card).

## Non-goals

Mobile lock (47), bypass/exception grants, usage statistics, bedtime
reminders (v2).

## Design References

- DESIGN §13.5 (UX + enforcement), §13.3 (transparency), §15 (child
  register), §16 (routes)
