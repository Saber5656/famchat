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
   30 s grace; on reconnect run the §9.3 contract (invalidate rooms
   list + notifications, refetch open room since newest known id);
   `session.revoked` → 42's wipe flow; expose `useSocketEvent(event,
   handler)` hook mirroring web's API (22) for cache patching.
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
4. Receipts: read-by chips per 22's compact spec; markRead triggers —
   screen focused at bottom, new message while at bottom, screen focus
   event; never while scrolled up.
5. Oversight notice: permanent header banner for child sessions
   (`safety.oversight_notice`), same key as web.
6. Offline UX: banner when disconnected (NetInfo), sends fail fast with
   retry affordance (no outbox queue in v1 — documented limitation
   matching DESIGN §2.2).
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
pnpm --filter @famchat/mobile test -- --grep "rooms|chat|socket"
# manual: two-device checklist incl. background recovery
```

## Dependencies

42 (auth/session), 16 (receipts API), 15. Reference: 22.

## Non-goals

Images (44), board (45), push (46), lock screen/report/guardian surfaces
(47), typing/presence (v2).

## Design References

- DESIGN §17 (mobile), §10 (messaging rules), §9.1/§9.3 (WS lifecycle,
  reconnect), §13.3 (oversight notice)
