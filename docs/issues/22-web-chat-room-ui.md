# Issue 22: Web chat room UI

## Summary

Build the chat room screen: virtualized message list with upward
pagination, composer with optimistic send, delete flows, read receipts,
realtime updates, the child oversight notice, and the link-rendering
safety policy.

## Context

DESIGN §16 (web), §10 (messaging rules), §13.3 (transparency), §9.3
(reconnect). This is the highest-traffic screen and the reference
implementation for the mobile chat (43).

## Scope

In scope: `/s/[spaceId]/r/[roomId]` screen, message list/composer/receipts,
WS integration, markRead behavior, oversight notice, link policy, a11y.
Out of scope: image attach UI (23 — composer exposes a disabled slot),
report action (33), quiet-hours lock (34), moderation flag badge (31 shows
queue; the in-room guardian flag badge ships here reading
`moderationStatus`).

## Detailed Requirements

1. Message list: `@tanstack/react-virtual`; initial `messages.list` newest
   50 anchored bottom; upward infinite scroll via `before` cursor; day
   separators (space timezone via `Intl`); sender grouping (consecutive
   same-sender within 5 min collapse header); message bubbles: text
   (whitespace-preserving, no HTML), system messages centered localized
   (key+params from 14), deleted tombstones, guardian-only flag badge when
   `moderationStatus='flagged'` (DESIGN §10.2 rendering rule).
2. Link policy (DESIGN §19.3 XSS row): URLs in message text render as
   **plain text for child viewers**; for adults/guardians, linkified with
   `rel="noopener noreferrer" target="_blank"` behind a confirm dialog
   showing the full URL. No other markup ever interpreted.
3. Composer: textarea auto-grow (max 6 rows), 4000-char counter past 3500,
   Enter=send / Shift+Enter=newline (IME-composition-safe: ignore Enter
   during `compositionend` handling — test with Japanese input), optimistic
   append with `dedupeId` (ULID) + pending style; failure → inline retry/
   discard affordances (`QUIET_HOURS_ACTIVE` and `CONTENT_BLOCKED_NG_WORD`
   render their specific friendly messages once 27/29 land — the error
   mapping exists now via `errors.json`); send button ≥ 44 px in kid mode.
4. Receipts: `receipts.roomReaders` + `receipt.updated` events → per-
   message compact read-by row of avatar chips under the newest read
   boundary (family scale: show up to 5 + "+n"); own-message "unread by
   all" subtle state.
5. markRead: fire `receipts.markRead` with the newest visible message id
   when (a) room opened focused, (b) window refocus, (c) new message
   arrives while scrolled to bottom; never while scrolled up (preserve
   the user's unread boundary).
6. WS: subscribeRoom on mount / unsubscribe on unmount; `message.created`
   appends (dedupe against optimistic by dedupeId), `message.deleted`
   tombstones in place; reconnect → refetch since newest known id (the
   §9.3 contract via `around`/`list`).
7. Oversight notice (child sessions): permanent slim banner in the room
   header — `safety.oversight_notice` ja: 「おうちの ひとも よめます」 with
   furigana, en: "Your family grown-ups can read this" (DESIGN §13.3 hard
   requirement).
8. Observer mode (guardian, non-member): read-only pane; composer replaced
   by localized "observer mode" note + "join conversation" hint if
   applicable (v1: read-only only); no markRead calls (16 exclusion).
9. A11y: list is `role="log"` with `aria-live="polite"` for new messages;
   composer labeled; dialogs trap focus; contrast AA (DESIGN §16).
10. Tests: component — IME Enter safety, counter, optimistic dedupe
    reconciliation, tombstone/system/flag-badge rendering, link policy per
    role; Playwright two-browser: A sends, B receives < 2 s, B's receipt
    chip appears for A, delete propagates, reload restores history +
    scroll-to-unread; observer read-only path.

## Acceptance Criteria

- [ ] Golden chat loop (send/receive/receipt/delete) works live between
      two browsers with no reload; survives reconnect (kill WS in test).
- [ ] Child banner always visible in child sessions; link policy enforced
      per role.
- [ ] IME-safe composer verified with Japanese composition events.
- [ ] All strings ja+en; kid mode passes tap-target audit (≥ 44 px, test
      via computed styles on key controls).

## Validation

```bash
pnpm --filter @famchat/web test -- --grep chat
pnpm --filter @famchat/web exec playwright test --grep @chat
```

## Dependencies

21 (shell/WS hook), 16, 14.

## Non-goals

Image UI (23), report dialog (33), quiet-hours lock (34), editing,
reactions, typing indicators (v1 non-goals).

## Design References

- DESIGN §10 (messaging), §16 (web), §13.3 (transparency), §9.3
  (reconnect), §19.3 (link/XSS policy)
