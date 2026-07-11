# Issue 15: Realtime socket.io server + event fanout

## Summary

Attach socket.io to the API server with session-authenticated handshakes,
space/room/guardian channels, the Redis adapter, zod-typed event payloads,
and wire the first producers (messages, session revocation).

## Context

DESIGN §9 defines connection auth, the channel scheme, the event catalog,
and best-effort delivery semantics (DB is source of truth; clients refetch
on reconnect). Guardians get a dedicated per-space channel for safety
events (used by 27/28).

## Scope

In scope: socket.io server + redis adapter, handshake auth, channel joins,
`subscribeRoom`/`unsubscribeRoom`, `wsEmitter` service, shared payload
schemas, producers for `message.created/deleted`, `room.updated`,
`member.updated`, `session.revoked`, quiet-hours refusal hook point, tests.
Out of scope: client integration (22/43), receipts events (16), moderation/
notification events (27/37), presence/typing (v2).

## Detailed Requirements

1. `packages/shared/src/ws.ts`: zod schemas + TS types for every DESIGN
   §9.2 event payload and the client→server `subscribeRoom` ack protocol;
   event-name constants (`WS_EVENTS`).
2. Server (`apps/api/src/ws/index.ts`): socket.io v4 attached to the
   Fastify HTTP server at path `/ws`; CORS same allowlist as HTTP;
   `@socket.io/redis-adapter` on the shared Redis (pub/sub duplicates).
3. Handshake auth middleware: cookie (web) or `auth.token` (mobile) →
   `resolveSession`; failure ⇒ `next(Error('UNAUTHORIZED'))`. On connect:
   join `user:<id>`, `space:<id>` per active membership, and
   `space:<id>:guardians` for guardian memberships. Store `{ userId,
   membershipRoles }` on `socket.data`.
4. `subscribeRoom { spaceId, roomId }` (ack `{ ok } | { ok:false, code }`):
   `canAccessRoom` (member or observer) → join `room:<id>`; quiet-hours
   hook `wsQuietGate(socket, spaceId)` (no-op until 29) may refuse with
   `QUIET_HOURS_ACTIVE`. `unsubscribeRoom` leaves. Membership loss ⇒ 16-
   line note: rooms are re-checked on every subscribe; existing
   subscriptions are torn down via `member.updated` handling in step 6.
5. `wsEmitter` service (`src/ws/emitter.ts`): typed
   `emitToRoom/Space/SpaceGuardians/User(event, payload)` — validates
   payload with the shared schema in development, fire-and-forget in
   production. Exported for all later issues.
6. Producers wired now:
   - 14 hooks: `onMessageCreated` → `message.created` to room;
     `onMessageDeleted` → `message.deleted`.
   - 13 flows: room create/archive/membersUpdate → `room.updated` to
     space; membership changes → `member.updated` to space **and** force
     re-evaluation: on `member.updated` the server walks sockets of the
     affected user and makes them leave rooms/spaces they lost access to
     (and `space:*` on removal).
   - Session revocation: implement 06's `onSessionRevoked` → emit
     `session.revoked` to `user:<id>` and `disconnect(true)` the sockets
     carrying that session id (track session id on `socket.data`).
7. Reconnect contract: no replay buffer; document in `ws.ts` JSDoc the
   client duty (on reconnect: refetch rooms list, notifications, and open
   room since last id) — client issues implement it.
8. Integration tests (socket.io-client in vitest): auth success/failure;
   two clients in one room receive `message.created`; cross-tenant
   subscribe denied; observer guardian receives child-room events;
   adult-only room hidden from non-member guardian; removed member's
   socket is ejected from room+space channels; session revoke kicks;
   redis adapter smoke (two server instances in one test process sharing
   Redis, event crosses instances).

## Acceptance Criteria

- [ ] Every DESIGN §9.2 event name/payload has a shared schema; emitter is
      the only emit path (grep-style meta test: no direct `io.emit` outside
      `src/ws/`).
- [ ] Channel security proven: cross-tenant, adult-only-room, and
      post-removal cases.
- [ ] Multi-instance fanout works via Redis adapter (test).
- [ ] `/ws` path proxied cleanly in dev (document Next.js rewrite for 20).

## Validation

```bash
pnpm --filter @famchat/api test -- --grep "ws|realtime"
```

## Dependencies

14 (message hooks), 13, 06 (sessions).

## Non-goals

Client hooks (22/43), receipts (16), safety events (27/28), notification
event (37), presence/typing, message replay.

## Design References

- DESIGN §9 (realtime), §10.2 (lifecycle events), §3.3 (socket.io choice);
  ADR-001
