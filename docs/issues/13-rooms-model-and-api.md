# Issue 13: Rooms model + API (incl. guardian oversight rule)

## Summary

Implement rooms (family/group/direct), room membership maintenance, the
guardian-oversight access rule, and the `rooms` router — the structural core
of chat and of the child-safety model.

## Context

DESIGN §10.1 defines room types and the oversight rule: guardians can read
any room containing at least one child member without being members.
DESIGN §7.2 defines the schema. The family room auto-membership keeps every
member in the space-wide room.

## Scope

In scope: migration (rooms, room_members, direct-dedupe key, family-room
partial unique), family-room lifecycle hooks, `canAccessRoom`, `rooms`
router, membership maintenance, tests.
Out of scope: messages (14), receipts/unread numbers (16 — list returns
structure now, zero counts), WS (15), name moderation (27).

## Detailed Requirements

1. Migration per DESIGN §7.2: `rooms` (+ `directKey text?` unique — the
   sorted pair `"<userIdA>:<userIdB>"` for direct rooms), `room_members`
   (composite PK, `notify` enum all/none default all,
   `lastReadMessageId text?`), partial unique index: one family room per
   space (`WHERE type='family'`).
2. Family room lifecycle: implement `onSpaceCreated` hook (from 07) to
   create the family room; **backfill migration** creates family rooms +
   full room_members for any existing spaces. Membership maintenance: on
   membership create → add to family room; on membership removal → remove
   from all rooms in the space (row delete, messages remain). Implemented as
   application-service functions called by 07/08/09/members flows (no DB
   triggers).
3. `canAccessRoom(userId, room)` in `apps/api/src/services/roomAccess.ts`:
   true iff (a) active room_member, or (b) requester has guardian
   membership in `room.spaceId` AND the room has ≥ 1 member whose
   membership role is child (`room.observe` permission). Single SQL query;
   exported for reuse by 14/15/16/17.
4. `rooms` router (issue-10 builders):
   - `list({ spaceId })` — member rooms + (for guardians) observable child
     rooms marked `{ access: 'member'|'observer' }`; each with last-message
     placeholder (null until 14 integrates) and `unreadCount: 0`
     placeholder (16). Sorted by last activity.
   - `get({ spaceId, roomId })` — via `canAccessRoom`; includes members
     with roles.
   - `createGroup({ spaceId, name, memberUserIds })` — permission
     `room.createGroup` (guardian/adult); members must be active space
     members; creator auto-included; name 1–30 (moderation hook lands in
     27 — leave `moderateOrPass(name)` call site that currently passes).
   - `createDirect({ spaceId, otherUserId })` — permission
     `room.createDirect` (all roles); find-or-create by `directKey`;
     children may only target members of their own space (inherent — input
     validated as space member).
   - `archive({ spaceId, roomId })` — guardians any group room; adults own
     created; family room never (`VALIDATION_FAILED`); direct rooms cannot
     be archived (they can go dormant). Archived rooms are read-only
     (writes rejected `ROOM_ARCHIVED`) but remain listed under an archived
     section.
   - `updateMembers({ spaceId, roomId, add[], remove[] })` — same
     owner-of-room rule as archive; family room refuses; direct refuses;
     cannot remove the last guardian-or-adult from a room containing a
     child? — rule: a room may exist with only children as members; the
     oversight rule already guarantees guardian visibility, so **no
     additional constraint** (documented in code comment).
   - `setNotify({ spaceId, roomId, notify })` — own row.
5. Audit: `room.create`, `room.archive`, `room.members_update`.
6. Integration tests: family room auto-create + backfill; auto-membership
   on join/leave; direct dedupe under parallel create (unique-violation
   retry path returns the existing room); oversight matrix — guardian reads
   child-containing group they're not in; guardian **cannot** read
   adult-only room they're not in (`NOT_FOUND` to avoid existence oracle);
   adult cannot observe anything; archived write rejection; cross-tenant
   denial on every procedure.

## Acceptance Criteria

- [ ] Room type rules, archive rules, and updateMembers rules exactly per
      DESIGN §10.1 table (test per row).
- [ ] Oversight rule provably scoped: child-containing rooms only, guardian
      role only, `NOT_FOUND` for non-observable rooms.
- [ ] Direct-room parallel creation yields exactly one row.
- [ ] Family room invariants hold across join/leave/backfill.

## Validation

```bash
pnpm --filter @famchat/db db:migrate && pnpm --filter @famchat/api test -- --grep rooms
```

## Dependencies

09, 10, 11.

## Non-goals

Messages (14), unread math (16), WS events (15), room-name moderation (27),
typing/presence (v2).

## Design References

- DESIGN §7.2 (schema), §10.1 (room rules), §13.1 (`room.*` permissions),
  §13.2 (oversight)
