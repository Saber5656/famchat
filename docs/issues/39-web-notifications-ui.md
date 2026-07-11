# Issue 39: Web notifications UI (feed, badges, preferences)

## Summary

Build the in-app notification center on web: bell with live unread badge,
feed panel with read tracking and deep links, and the per-room / board
notification preference controls.

## Context

DESIGN §14 (in-app channel is the reliable baseline; push is best-effort)
and §16. Backend feed/prefs exist since 37/13/19; this issue is pure UI +
cache orchestration.

## Scope

In scope: bell + badge, feed panel/page, mark-read behaviors, deep links,
per-room notify toggle surfacing, board notify toggle, child-appropriate
rendering.
Out of scope: push subscription UX (38), mobile (46), digest/grouping
logic beyond tag collapse (v2).

## Detailed Requirements

1. Bell in the shell header (21): unread count from
   `notifications.unreadCounts` (notifications component), live via WS
   `notification.created` increments; badge caps 99+; child mode: bell
   sized up, count dot simplified.
2. Feed panel (popover ≥ 900 px; full route `/notifications` below):
   infinite scroll of `notifications.feed`; row renderer per type —
   localized title/body (server-composed strings from payload keys/
   params rendered client-side via the same `notifications.*` catalog:
   client re-renders from `{ type, payload }` for locale-switch
   correctness rather than storing prerendered text — matches 37's row
   shape), type icon (message/board/safety/system families), relative
   time, unread dot; click → `link` navigation + `markRead([id])`;
   "mark all read" button; empty state.
3. Safety-type prominence: `moderation.flagged`, `report.new`,
   `child.device.linked` rows get a distinct accent + guardian-console
   deep links (31).
4. Read semantics: row click marks read; panel open does NOT bulk-mark
   (explicit action only — predictability for guardians tracking safety
   items); unread notification count decrements optimistically.
5. Preference surfacing: room header menu (22) gains notify all/none
   toggle (13's `rooms.setNotify`) with state icon in room list (21
   already renders it); board page (24) header gains board notify toggle
   (19's `board.setNotify`); copy explains scope ("この部屋の通知").
6. Child rendering: children receive `message.new`/board types only (37
   catalog); feed strings child-registered with furigana where the
   namespace provides it; no safety types ever reach child feeds
   (backend-guaranteed; UI asserts by rendering unknown-type rows as a
   safe generic — forward-compat rule).
7. Tests: component — per-type renderer snapshot ja/en, unknown-type
   fallback, badge increments on WS event, explicit-read semantics;
   Playwright: B sends message → A's bell increments live → open feed →
   click row → lands in room at message → badge decremented; guardian
   flag event → accented row → guardian console deep link.

## Acceptance Criteria

- [ ] Bell/badge/feed live-update without reload; deep links land
      correctly (room deep link scrolls to message via 22's `around`).
- [ ] Every 37 catalog type has a renderer + ja/en strings (parity test
      over the type table).
- [ ] Read semantics exactly as specified (explicit only).
- [ ] Room/board preference toggles round-trip and reflect everywhere
      they're displayed.

## Validation

```bash
pnpm --filter @famchat/web test -- --grep notifications
pnpm --filter @famchat/web exec playwright test --grep @notifcenter
```

## Dependencies

37 (feed/types), 21 (shell), 22 (room deep links), 24 (board toggle
slot).

## Non-goals

Push permission UX (38), grouping/digests, per-type mute matrix (v2),
sounds.

## Design References

- DESIGN §14 (channels, catalog), §16 (web), §13.2 (guardian alert
  surfacing)
