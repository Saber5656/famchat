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
3. Handshake auth middleware — concrete adapter contract: implement
   `resolveSessionFromHandshake(handshake)` in `src/ws/auth.ts` that
   extracts the raw token from either the `famchat_session` cookie
   (parse `handshake.headers.cookie`) or `handshake.auth.token`
   (mobile), then calls the SAME session-service lookup used by HTTP
   (06's hash lookup + expiry/revocation checks — no duplicated logic);
   failure ⇒ `next(Error('UNAUTHORIZED'))`. On connect: join
   `user:<id>`, `space:<id>` per active membership, and
   `space:<id>:guardians` for guardian memberships. Store `{ userId,
   sessionId, membershipRoles }` on `socket.data` (sessionId enables the
   revocation kick in step 6).
4. `subscribeRoom` (ack `{ ok } | { ok:false, code }`): the incoming
   payload is **zod-validated against the shared schema before any
   lookup** (DESIGN §19.2 — malformed payloads ack `VALIDATION_FAILED`);
   then `roomAccess` (member or observer) → join `room:<id>`;
   quiet-hours hook `wsQuietGate(socket, spaceId)` (no-op until 29) may
   refuse with `QUIET_HOURS_ACTIVE`. `unsubscribeRoom` likewise
   validated. Note: access is re-checked on every subscribe; existing
   subscriptions of a user who loses access are torn down by the
   `member.updated` handling in step 6.
5. `wsEmitter` service (`src/ws/emitter.ts`): typed
   `emitToRoom/Space/SpaceGuardians/User(event, payload)` — validates
   payload with the shared schema in development, fire-and-forget in
   production. Exported for all later issues.
6. Producers wired now:
   - 14 hooks: `onMessageCreated` → `message.created` to room;
     `onMessageDeleted` → `message.deleted`.
   - 13 flows: room create/archive/updateMembers → `room.updated` to
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
      post-removal cases; malformed subscribe payloads rejected by schema
      validation before any DB lookup (test).
- [ ] Multi-instance fanout works via Redis adapter (test).
- [ ] `apps/api/README.md` gains a "WS behind proxies" note (path `/ws`,
      upgrade requirements, the dev-rewrite rule that issue 20
      implements and issue 49 mirrors in Caddy).

## Validation

```bash
pnpm -w typecheck
pnpm --filter @famchat/shared test
pnpm --filter @famchat/api test -- -t ws
```

## Dependencies

14 (message hooks), 13, 06 (sessions).

## Non-goals

Client hooks (22/43), receipts (16), safety events (27/28), notification
event (37), presence/typing, message replay.

## Design References

- DESIGN §9 (realtime), §10.2 (lifecycle events), §3.3 (socket.io choice);
  ADR-001
