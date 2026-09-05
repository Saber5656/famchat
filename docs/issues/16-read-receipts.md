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
2. `receipts.roomReaders({ spaceId, roomId })` — authorization via
   `roomAccess` exactly like `messages.list`: members and observer
   guardians may query; adult-only rooms return `NOT_FOUND` to
   non-member guardians; cross-tenant denied (tests restate all three).
   Returns `{ userId, lastReadMessageId }[]` for room members; clients
   derive per-message read-by sets locally.
3. `rooms.list` integration: replace placeholder `unreadCount` with the
   DESIGN §7.7 query (messages newer than my pointer, not mine, not
   deleted), and `lastMessage` preview via the 14 DTO (respecting
   redaction); one query per space using lateral joins. Observer-access
   rooms (guardian, no room_members row) return `unreadCount: null` —
   observers have no read pointer by design (no receipts, no unread
   badge; DESIGN §10.1 "no notifications unless member"); clients render
   no badge for null. If the ORM shape is awkward, a Prisma `$queryRaw`
   typed helper is allowed as a **documented reviewed exception** to
   DESIGN §19.2: parameterized placeholders only, zero string
   interpolation, output mapped to DTOs, a code comment citing this
   issue, and an injection regression test.
4. `messages.list` marks nothing automatically — explicit `markRead` from
   clients only (scroll-position-driven; client issues).
5. Tests: monotonic guard (stale markRead is a no-op, no event); unread
   math excludes own + deleted messages; empty-room zero; observer rooms
   report `unreadCount: null`; observer blocked from markRead but allowed
   roomReaders; adult-only room roomReaders → `NOT_FOUND` for non-member
   guardian; WS event received by second client; cross-tenant denial;
   unread performance sanity (≤ 30 rooms × 1000 msgs seeded — list
   completes < 300 ms locally, soft assertion logged not failed).

## Acceptance Criteria

- [ ] Unread counts correct across the seeded matrix (own/deleted/foreign
      messages).
- [ ] Monotonicity proven; duplicate markRead emits no event.
- [ ] Observer receipt exclusion enforced and documented.
- [ ] `rooms.list` returns real lastMessage + unreadCount.

## Validation

```bash
pnpm -w typecheck
pnpm --filter @famchat/api test -- -t receipt
```

## Dependencies

15 (WS), 14, 13.

## Non-goals

Per-message receipt rows, delivery (vs read) states, badge aggregation
(37), client UI (22/43).

## Design References

- DESIGN §7.7 (read model), §8.2 (`receipts`), §9.2 (`receipt.updated`)
