# Issue 16: Read receipts + unread counts

## Summary

Implement last-read tracking per room member, unread counts in the rooms
list, per-message read-by derivation, and the `receipt.updated` realtime
event.

## Context

DESIGN §7.7: receipts derive from `room_members.last_read_message_id`
pointers (no per-message rows). ULID ordering makes "read up to X" a simple
id comparison. Family scale keeps the count queries cheap.

## Scope

In scope: `receipts` router (markRead, roomReaders), unread integration in
`rooms.list`, last-message integration, WS event, tests.
Out of scope: client rendering (22/43), notification badge totals (37/39).

## Detailed Requirements

1. `receipts.markRead({ spaceId, roomId, messageId })` — room **member**
   only (observers don't produce receipts — guardian reads must not be
   visible to children as "read", a child-trust decision documented in
   code); validates message belongs to room; **monotonic**: update only if
   `messageId > current` (ULID string compare); emits `receipt.updated`
   to the room on actual change; idempotent otherwise.
2. `receipts.roomReaders({ spaceId, roomId })` — returns
   `{ userId, lastReadMessageId }[]` for room members (observer guardians
   may query); clients derive per-message read-by sets locally.
3. `rooms.list` integration: replace placeholder `unreadCount` with the
   DESIGN §7.7 query (messages newer than my pointer, not mine, not
   deleted), and `lastMessage` preview via the 14 DTO (respecting
   redaction); one query per space using lateral joins (provide the exact
   SQL in a Prisma `$queryRaw` typed helper if the ORM shape is awkward —
   raw allowed here as a reviewed exception, note in code).
4. `messages.list` marks nothing automatically — explicit `markRead` from
   clients only (scroll-position-driven; client issues).
5. Tests: monotonic guard (stale markRead is a no-op, no event); unread
   math excludes own + deleted messages; empty-room zero; observer blocked
   from markRead but allowed roomReaders; WS event received by second
   client; cross-tenant denial; unread performance sanity (≤ 30 rooms ×
   1000 msgs seeded — list completes < 300 ms locally, soft assertion
   logged not failed).

## Acceptance Criteria

- [ ] Unread counts correct across the seeded matrix (own/deleted/foreign
      messages).
- [ ] Monotonicity proven; duplicate markRead emits no event.
- [ ] Observer receipt exclusion enforced and documented.
- [ ] `rooms.list` returns real lastMessage + unreadCount.

## Validation

```bash
pnpm --filter @famchat/api test -- --grep receipt
```

## Dependencies

15 (WS), 14, 13.

## Non-goals

Per-message receipt rows, delivery (vs read) states, badge aggregation
(37), client UI (22/43).

## Design References

- DESIGN §7.7 (read model), §8.2 (`receipts`), §9.2 (`receipt.updated`)
