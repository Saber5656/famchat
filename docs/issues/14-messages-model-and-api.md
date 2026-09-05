# Issue 14: Messages model + API

## Summary

Implement the messages table and the `messages` router: send text (with
idempotent dedupe), cursor pagination, soft delete with tombstone rendering,
and system messages — realtime fanout and moderation attach in later issues
through hooks defined here.

## Context

DESIGN §7.2 (schema), §10.2 (lifecycle), §8.4 (mutation pipeline). ULID ids
are the ordering key. Image messages become sendable in 17/18 (attachment
readiness); the `sendImage` procedure ships here gated on attachment status.

## Scope

In scope: migration, `messages` router (sendText, sendImage, list, around,
delete), DTO mapper with redaction, dedupe, hooks for moderation/WS/notify,
rate-limit wiring, tests.
Out of scope: attachments upload (17), WS delivery (15), receipts (16),
moderation behavior (27 — hook passes through), quiet hours (29).

## Detailed Requirements

1. Migration per DESIGN §7.2: `messages` with index (room_id, id DESC),
   `attachment_id text null` **without an FK constraint in this migration**
   (the attachments table arrives in 17, whose migration adds
   `ALTER TABLE messages ADD CONSTRAINT … FOREIGN KEY (attachment_id)`),
   plus `dedupe_id text?` and permanent unique index
   `(sender_id, dedupe_id)` (nullable dedupe ignored; dedupeId is a
   client ULID, so no expiry window — DESIGN §8.4). Body ≤ 4000 (caption
   ≤ 500 for kind=image) enforced in zod + DB check constraints.
2. Send pipeline (`sendText({ spaceId, roomId, body, dedupeId? })`,
   `sendImage({ spaceId, roomId, attachmentId, caption? ≤ 500,
   dedupeId? })`): order exactly per DESIGN §8.4 — session → membership/
   permission (`roomAccess === 'member'`; observers cannot send —
   guardians must join to talk, explicit rule + test) → zod input →
   room-state check (`assertRoomWritable` from 13) → quiet-hours hook
   (`assertNotQuiet(ctx)` no-op until 29) → moderation hook
   (`moderateContent()` returns `clean` until 27) → insert →
   post-commit hooks `onMessageCreated(message)` (WS in 15, notify in 37
   — register-pattern like `onSpaceCreated`). An order-recording test
   double asserts this exact sequence.
   - `sendImage`: the attachments table arrives in issue 17 (which runs
     after this issue), so ship `sendImage` here **disabled** behind an
     `attachmentsEnabled` capability flag (returns `VALIDATION_FAILED
     details.reason='attachments_not_enabled'`), with its validation +
     atomic-claim logic specified for 17's schema (attachment exists, same
     space, uploader = sender, `status='ready'`, unclaimed) and its tests
     written but `skip`-marked. Issue 17 flips the flag, implements the
     claim column, and unskips the tests (`TODO(issue-17)` markers).
3. Dedupe: same `(senderId, dedupeId)` returns the original message
   (idempotent success, no duplicate row) — implement via
   unique-violation catch → fetch existing.
4. `list({ spaceId, roomId, before?, limit≤50 })` — `roomAccess` member
   or observer; descending by id; returns `MessageDTO[]` + `nextCursor`.
   `around({ spaceId, roomId, messageId })` — resolves the message,
   verifies it belongs to `roomId`, applies the same `roomAccess`
   member-or-observer check (cross-tenant/no-access ⇒ `NOT_FOUND`, no
   existence oracle); returns 25 before/after for deep links.
5. Soft delete `delete({ spaceId, roomId, messageId })`: own message
   (`message.deleteOwn`) or guardian (`message.deleteAny`, audited as
   `message.delete_any`); sets deletedAt/By; post-commit
   `onMessageDeleted`.
6. `MessageDTO` mapper: id, roomId, sender {id, displayName, avatarPreset,
   role} | null, kind, body, attachment (null until 17), moderationStatus,
   createdAt, deleted flag — deleted ⇒ body/attachment nulled server-side
   (redaction happens in the mapper, single choke point, test-proven).
   `moderationStatus` exposed only to guardians; others always see `clean`
   (DESIGN §10.2 rendering rule).
7. System messages: internal `postSystemMessage(roomId, key, params)`
   service; wire exactly two call sites in
   `apps/api/src/services/roomMembership.ts` (13): the add-to-family-room
   function posts `chat.system.member_joined`, the remove function posts
   `chat.system.member_left` (keys added to `packages/i18n` chat
   namespace, ja+en); body stores `{ key, params }` JSON; clients render
   localized.
8. Rate limit: register `messageSend` in the 12 policy map (30/min burst
   10/10 s per user).
9. Tests: pagination boundaries (empty room, exact page, before-cursor);
   dedupe under parallel double-send (one row); observer-cannot-send;
   member-of-other-space `NOT_FOUND`; delete permissions matrix; redaction
   (deleted content absent from DTO and from `around`); moderationStatus
   hidden from non-guardians; system messages render key+params; length
   caps; rate limit trips.

## Acceptance Criteria

- [ ] Lifecycle matches DESIGN §10.2 for clean/deleted paths (flag paths in
      27); pipeline hooks in the exact §8.4 order (order-recording test
      double green).
- [ ] Idempotent send proven under parallelism.
- [ ] Redaction single-point and test-proven; no ORM entity leaks (DTO
      snapshot test).
- [ ] Cross-tenant + observer-send denials green.

## Validation

```bash
pnpm --filter @famchat/db db:migrate && pnpm --filter @famchat/api test -- --grep messages
```

## Dependencies

13. (17 flips `sendImage` on; 15 wires WS; 27 wires moderation; 29 wires
quiet hours; 37 wires notifications.)

## Non-goals

Editing (v1 non-goal), reactions, threads, search, link unfurling (never
fetches URLs — DESIGN §19.2), retention purge (36).

## Design References

- DESIGN §7.2 (schema), §8.2/§8.4 (router/pipeline), §10.2 (lifecycle),
  §19.5 (limits)
