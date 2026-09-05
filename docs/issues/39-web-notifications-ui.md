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
   the row carries `{ type, payload }` (37's canonical shape) and the
   client renders localized title/body from the `notifications.*`
   catalog (locale-switch-safe; nothing prerendered is stored). Payload
   params (sender names, excerpts) are user-derived: render as plain
   text via normal React escaping — never HTML, never linkified.
   Type icon (message/board/safety/system families), relative time,
   unread dot; click → navigate to `link` **only after validating it is
   an app-internal relative route (same `^/(s/|notifications|settings)`
   rule as 38's SW; else fall back to the feed)** + `markRead([id])`;
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
6. Child rendering: a child's feed contains exactly what the 37 catalog
   addresses to them (`message.new`, board types, `member.joined`,
   `quiet_hours.updated`, `space.deletion_requested`); guardian safety
   types (`moderation.flagged`, `report.new`, `child.device.linked`,
   `export.ready`) never reach child feeds because their recipient sets
   are guardian/owner-only — backend-guaranteed, and the UI additionally
   renders any unknown type as a safe generic row (forward-compat rule).
   Child-visible strings use the child register with furigana where the
   namespace provides it.
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
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/web test -- -t notifications
pnpm --filter @famchat/web exec playwright test --grep @notifcenter
```

## Dependencies

21 (shell), 22 (room deep links + header menu), 24 (board toggle slot),
31 (guardian console routes for safety deep links), 37 (feed/types).

## Non-goals

Push permission UX (38), grouping/digests, per-type mute matrix (v2),
sounds.

## Design References

- DESIGN §14 (channels, catalog), §16 (web), §13.2 (guardian alert
  surfacing)
