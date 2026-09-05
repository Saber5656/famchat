# Issue 43: Mobile rooms + chat

## Summary

Implement the mobile Rooms tab and chat screen: room list with unread
badges, inverted virtualized message list, composer, receipts, socket.io
integration with app-state-aware reconnection, and the child oversight
notice — feature parity with web chat (22) minus images (44).

## Context

DESIGN §17 mirrors §16's chat behaviors; DESIGN §9.3 reconnect contract
matters doubly on mobile (backgrounding kills sockets). 22 is the
reference implementation; reuse its interaction specs verbatim where not
stated here.

## Scope

In scope: rooms list screen, chat screen (text messages), WS client
lifecycle, receipts, markRead, oversight notice, error/quiet-hours error
surfaces (lock screen itself is 47).
Out of scope: images (44), board (45), push (46), report action + lock
screen + guardian tab (47).

## Detailed Requirements

1. Socket module `src/lib/socket.ts`: socket.io-client with
   `auth: { token }` from the token store; connect on app foreground
   with an authed session, disconnect on background (AppState) after
   30 s grace; on reconnect run the §9.3 contract exactly as 22
   specifies it (invalidate rooms list + notifications; for the open
   room fetch the newest page and merge-or-reset on gap);
   `session.revoked` → 42's wipe flow; expose `useSocketEvent(event,
   handler)` hook mirroring web's API (22) for cache patching. **Room
   subscription lifecycle (DESIGN §9.1)**: the chat screen emits
   `subscribeRoom { spaceId, roomId }` on mount/focus and
   `unsubscribeRoom` on unmount/blur, handling the ack — `ok:false`
   with `QUIET_HOURS_ACTIVE` triggers the quiet placeholder (req 6b),
   other ack errors show the localized error state; re-subscribe after
   every reconnect.
2. Rooms tab: sectioned list (family pinned, active by last activity,
   archived collapsed) with the same row anatomy as 21 (name resolution,
   preview redaction rules, unread badge, notify-off icon, observer
   badge for guardians); pull-to-refresh; create: direct-message picker
   (all roles) + group dialog (guardian/adult) per 21's rules; kid mode
   row sizing.
3. Chat screen (`/rooms/[roomId]`): FlashList inverted with upward
   pagination (`messages.list` before-cursor); day separators (space
   TZ); bubbles per 22's rendering rules (text whitespace, system
   messages, tombstones, guardian flag badge, link policy: children
   plain-text, adults confirm-dialog via `expo-web-browser`); composer
   (auto-grow 6 lines, 4000 counter, send button ≥ 44 pt, optimistic
   send with dedupeId + retry/discard, `QUIET_HOURS_ACTIVE` and
   `CONTENT_BLOCKED_NG_WORD` mapped errors); keyboard handling
   (KeyboardAvoidingView, list maintains position).
3b. Delete flows (parity with 22 §4b): long-press action sheet with
   "delete" on own messages (all roles) and on any message for
   guardians incl. observer mode; confirm sheet (child-register variant
   for children); tombstone renders for everyone via `message.deleted`.
4. Receipts: read-by chips per 22's compact spec; markRead triggers —
   screen focused at bottom, new message while at bottom, screen focus
   event; never while scrolled up.
5. Oversight notice: permanent header banner for child sessions
   (`safety.oversight_notice`), same key as web.
6. Offline UX: banner when disconnected (NetInfo), sends fail fast with
   retry affordance (no outbox queue in v1 — documented limitation
   matching DESIGN §2.2).
6b. Quiet-hours interim handling (until 47's full lock screen): any
   `QUIET_HOURS_ACTIVE` error or `quietHours.state {active:true}` event
   hides all room/board content and renders a simple full-screen
   placeholder (moon emoji + localized "おやすみ じかん" + unlock time
   from `until`) that 47 replaces with the final lock screen — content
   must never remain visible behind it (`TODO(issue-47)` marker).
7. Tests: unit — socket lifecycle state machine (foreground/background/
   revoked transitions with mocked AppState), markRead trigger logic;
   component — bubble variants, composer counter + IME-safe submit (RN:
   no Enter-to-send on soft keyboards; send button only — deviation from
   web documented), room row variants; manual device checklist: two
   devices chat live, background→foreground recovers missed messages,
   receipts flow both ways, guardian observer read-only view.

## Acceptance Criteria

- [ ] Live chat between a device and web with reconnect recovery
      (backgrounding test on checklist, evidence in PR).
- [ ] Rendering/interaction rules match 22 wherever this issue doesn't
      explicitly deviate (deviations: send-button-only, FlashList).
- [ ] Child oversight banner + link policy enforced.
- [ ] Kid-mode audit passes (tap targets, type scale) on both tabs.

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/mobile test -- -t rooms
pnpm --filter @famchat/mobile test -- -t chat
pnpm --filter @famchat/mobile test -- -t socket
# manual: two-device checklist incl. background recovery
```

## Dependencies

16 (receipts API), 22 (reference interaction contract — hard dependency;
this issue reuses its specs verbatim), 42 (auth/session).

## Non-goals

Images (44), board (45), push (46), lock screen/report/guardian surfaces
(47), typing/presence (v2).

## Design References

- DESIGN §17 (mobile), §10 (messaging rules), §9.1/§9.3 (WS lifecycle,
  reconnect), §13.3 (oversight notice)
