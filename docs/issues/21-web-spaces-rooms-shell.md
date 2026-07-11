# Issue 21: Web shell — space switcher, room list, unread badges

## Summary

Build the authenticated web shell: minimal space switcher, room list with
unread badges and last-message previews, create-room dialogs, and the
responsive split layout that chat (22) and board (24) plug into.

## Context

DESIGN §16 route map (`/s/[spaceId]`, split view on desktop) and §5/§10
tenancy semantics. Multi-space membership exists (ADR-002) but the v1
switcher stays minimal.

## Scope

In scope: `/s/[spaceId]` shell + room list, space switcher, create
group/direct dialogs, archived section, WS-driven list updates, empty
states.
Out of scope: chat pane content (22), board pages (24), guardian routes
(31/32), notifications bell behavior (39 — placeholder icon only).

## Detailed Requirements

1. Space switcher in the shell header: current space name; dropdown of my
   memberships (`spaces.list`); last-visited space remembered
   (localStorage) and used by the root redirect `/` → `/s/<lastOrFirst>`.
2. Room list pane (`rooms.list`): sections — pinned family room first,
   active rooms by last activity, archived collapsed at bottom; each row:
   localized room display name (family room name from i18n; direct rooms
   show the other member's displayName + avatar preset icon; group rooms
   their name), last-message preview (redaction-aware: deleted → tombstone
   text, image → 📷 + localized "photo"), unread badge (count from 16,
   99+ cap), notify-off icon when `notify=none`, `observer` badge for
   guardian-observable rooms the guardian isn't a member of (distinct
   visual per DESIGN §13.2 transparency).
3. Live updates: WS client (socket.io-client wrapper hook
   `useFamchatSocket` created here, reused by 22/39): on
   `message.created/deleted`, `room.updated`, `member.updated`,
   `receipt.updated` → patch TanStack Query caches (list reorder, unread
   increments — no full refetch); on reconnect → invalidate rooms list
   (the DESIGN §9.3 reconnect contract; document in the hook).
4. Create room: FAB/button gated by `room.createGroup` permission
   (children see a "new direct message" picker only — `room.createDirect`);
   group dialog: name field (30 cap + counter), member multi-select of
   active space members; direct picker: member list → find-or-create then
   navigate.
5. Responsive: < 900 px = list OR room full-screen (route-driven); ≥ 900 px
   = split (list 320 px + outlet). Child mode: larger rows, furigana
   labels.
6. Empty states (ja/en, furigana for child): no rooms yet, archived-empty,
   switcher-single-space (hide dropdown chrome).
7. Tests: component tests for row rendering variants (unread, tombstone,
   observer badge, notify-off); switcher persistence; Playwright: two-user
   scenario — user B sends via API, user A's list reorders + badge
   increments without reload (WS path), then A opens room (badge clears
   after 22 lands — here assert reorder only).

## Acceptance Criteria

- [ ] Shell navigates spaces/rooms per DESIGN §16 routes; last-visited
      space restored.
- [ ] Unread/preview/observer/archived visuals all present and localized
      ja+en.
- [ ] WS cache-patching works (Playwright-proven, no reload).
- [ ] Child mode shows simplified creation options and kid styling.

## Validation

```bash
pnpm --filter @famchat/web test && pnpm --filter @famchat/web exec playwright test --grep @shell
```

## Dependencies

16, 20.

## Non-goals

Message pane (22), board (24), guardian console (31/32), notification
center (39), presence.

## Design References

- DESIGN §16 (web), §10.1 (room rules), §9.3 (reconnect contract), §13.2
  (observer transparency), §7.7 (unread)
