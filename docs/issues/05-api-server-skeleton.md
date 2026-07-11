# Issue 05: `apps/api` — Fastify + tRPC server skeleton

## Summary

Create the API server process: Fastify with a mounted tRPC router, health/
readiness endpoints, pino logging with redaction, fail-fast env loading,
error mapping, CORS + CSRF-header plumbing, a no-op notification service
interface, an integration-test harness, and a production Dockerfile.

## Context

DESIGN §3.2 defines the `api` process; §8 the API surfaces; §19 the security
posture. Every feature router from issue 06 onward plugs into this skeleton,
so its context object, error mapping, and test harness must be right first.

## Scope

In scope: server bootstrap, tRPC wiring + context, `/healthz` `/readyz`,
logging, CORS/CSRF plumbing, error mapper, notifyService stub, Dockerfile,
test harness, graceful shutdown.
Out of scope: any business router (06+), socket.io (15), rate limits (12).

## Detailed Requirements

1. Scaffold `apps/api` as `@famchat/api`; deps: fastify, @fastify/cors,
   @fastify/cookie, @trpc/server, zod, pino, ioredis (readiness ping +
   future rate-limit store), @aws-sdk/client-s3 (readiness ping + future
   presigning), @famchat/{shared,db}. Entry `src/index.ts` →
   `buildServer()` in `src/server.ts` (exported for tests).
2. Boot: `loadEnv(apiEnvSchema)` before anything; crash with readable output
   on invalid env (from issue 03).
3. Logging: pino via Fastify logger; redaction list per DESIGN §19.6 in
   full: header paths `req.headers.authorization`, `req.headers.cookie`,
   and key patterns `password`, `token`, `code`, `secret`, and any key
   matching `*_key` / `*Key` (implement as pino `redact.paths` plus a
   custom serializer for wildcard key patterns; unit-test with a fixture
   object containing `s3SecretAccessKey`, `vapid_private_key`); request-id
   (`x-request-id` honored or generated); no message-content logging
   anywhere.
4. tRPC: `src/trpc.ts` defines `initTRPC.context<Context>()`, base router in
   `src/routers/index.ts` (`export type AppRouter`), fastify adapter mounted
   at `/trpc`. `Context = { db, env, log, notify: NotifyService, ip,
   session: null | { user: UserCtx, session: SessionCtx, memberships:
   MembershipCtx[] } }` — the full shape (including `memberships`, per
   DESIGN §6.3) is declared NOW as types in `packages/shared`; the issue-05
   stub `resolveSession(req)` always returns null, and issue 06 replaces
   only that function without touching the type.
5. Error mapping: a single `errorFormatter` + helper `throwFamchat(code,
   msg?, details?)` mapping `FamchatErrorCode` → TRPCError codes
   (`PERMISSION_DENIED→FORBIDDEN`, `NOT_FOUND→NOT_FOUND`,
   `RATE_LIMITED→TOO_MANY_REQUESTS`, `VALIDATION_FAILED→BAD_REQUEST`,
   default `BAD_REQUEST`; unexpected errors → `INTERNAL_SERVER_ERROR` with
   generic message, full error logged). **Canonical wire shape** (this is
   the concrete form of DESIGN §8.3, and DESIGN has been aligned to it):
   the tRPC error object carries `message` plus
   `data: { cause: FamchatErrorCode, details?: Record<string, unknown> }`;
   zod issues surface as `cause: 'VALIDATION_FAILED'` with flattened field
   errors in `data.details.fieldErrors`. A snapshot test pins this shape;
   clients (20/41) rely on `error.data.cause`.
6. CORS: @fastify/cors with origin allowlist from `WEB_ORIGINS` exactly
   (no wildcard), `credentials: true`, allowed headers include
   `x-famchat-csrf`, methods GET/POST. CSRF plugin: reject any mutating
   HTTP request that carries a session **cookie** but lacks header
   `x-famchat-csrf: 1` (DESIGN §6.3, §19.4) — bearer requests exempt.
   Wire now, meaningful once 06 lands; unit-test the plugin directly.
7. REST: `GET /healthz` → `{ ok: true }` (no deps); `GET /readyz` → checks
   `db.$queryRaw SELECT 1`, Redis `PING` (ioredis), S3 `HeadBucket`; 503
   with per-check booleans on failure.
8. `NotifyService` interface in `src/services/notify.ts`:
   `enqueue(event: NotifyEvent): Promise<void>` where `NotifyEvent` is a
   discriminated union (typed now with the DESIGN §14.2 type names); v1 stub
   implementation logs at debug level. Issue 37 swaps in BullMQ.
9. Graceful shutdown: SIGTERM/SIGINT → close fastify, disconnect db/redis;
   10 s deadline.
10. Test harness `test/helpers.ts`: builds the server against the dev
    compose services with a random-suffix schema or truncation between
    tests; exports `createCaller()` for direct tRPC invocation and
    `injectHttp()` for REST. One passing test: `/healthz`, `/readyz`, CORS
    denial for a disallowed origin, CSRF plugin rejection.
11. `Dockerfile` (multi-stage: pnpm fetch → build → `node:22-slim` runtime,
    non-root user, `NODE_ENV=production`, `CMD node dist/index.js`,
    `HEALTHCHECK` hitting `/healthz`).

## Acceptance Criteria

- [ ] `pnpm --filter @famchat/api dev` boots against issue-02 services;
      `/healthz` and `/readyz` return 200.
- [ ] Invalid env aborts startup listing every offending variable.
- [ ] Error mapper tests cover all mapped codes; secrets absent from logs
      (test asserts redaction of `authorization`/`cookie`).
- [ ] CSRF/CORS behavior tests pass; docker image builds and runs.

## Validation

```bash
pnpm --filter @famchat/api typecheck && pnpm --filter @famchat/api test
# image boot check: build, run with the dev env, poll the real healthcheck
docker build -f apps/api/Dockerfile -t famchat-api:dev .
docker run -d --rm --name famchat-api-check --env-file .env --network host famchat-api:dev
for i in $(seq 1 30); do curl -sf http://127.0.0.1:8080/healthz && break; sleep 1; done
curl -sf http://127.0.0.1:8080/readyz          # 200 with dev services up
docker stop famchat-api-check
```

## Dependencies

03, 04.

## Non-goals

Business logic, sessions (06), rate limiting (12), socket.io (15), metrics.

## Design References

- DESIGN §3.2 (processes), §8 (surfaces, error model), §19.4/§19.6
  (hardening, secrets), §20 (logging)
