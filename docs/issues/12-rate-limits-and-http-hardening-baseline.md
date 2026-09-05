# Issue 12: Rate limits + HTTP hardening baseline

## Summary

Centralize rate limiting on the API using Redis-backed @fastify/rate-limit
driven by the `@famchat/shared/limits` policy table, and apply the HTTP
hardening baseline (body limits, security headers, `retryAfter` semantics).

## Context

DESIGN §19.5 defines the canonical limit table; issues 06–09 shipped
route-local limits that this issue unifies so drift is impossible. The API
serves only JSON (uploads are presigned direct-to-S3), so strict body limits
are safe.

## Scope

In scope: rate-limit plugin config, policy application map, `RATE_LIMITED`
error shape, API security headers, body/timeout limits, tests, docs sync
check.
Out of scope: Caddy-level global IP limit (49), web CSP (38/54), upload
quotas (17).

## Detailed Requirements

1. Add `@fastify/rate-limit` with Redis store (shared ioredis instance from
   05). Build `applyRateLimits(app)` in `apps/api/src/plugins/rate-limit.ts`
   mapping policy keys → route matchers:
   - `authLogin` → `auth.login`; `authReset` → `requestPasswordReset`;
     `invitePreviewAccept` → invite preview/accept + `auth.childLink` +
     `auth.verifyChildPin`; `messageSend` (wired when 14 lands — the map is
     data, adding a row is the only change); `boardWrite`, `uploadRequest`,
     `reportCreate`, `adminApi` similarly reserved.
   - Scopes: `ip` (default keyGenerator), `account` (session user id),
     `session`. Composite policies (login: IP AND account) implemented as
     two limiter checks.
2. tRPC procedures opt in via `.meta({ rateLimit: 'authLogin' })`; a single
   tRPC middleware reads meta and consults the limiter — replace the
   route-local implementations from 06–09 (delete them; tests must still
   pass).
3. `RATE_LIMITED` responses include `details.retryAfterSec` (canonical
   field per DESIGN §19.5) and HTTP 429 with `Retry-After` header on REST
   paths.
4. Hardening baseline on the API server: `bodyLimit: 1_048_576` (1 MiB),
   Fastify options `connectionTimeout: 30_000` and
   `requestTimeout: 30_000`, headers on all API responses:
   `X-Content-Type-Options: nosniff`, `Referrer-Policy: same-origin`,
   `Cache-Control: no-store` (API responses are personal data). Reject
   `Content-Type` other than `application/json` on mutating REST routes.
5. Limiter availability policy per DESIGN §19.5 (canonical there): Redis
   down ⇒ fail **closed** for credential/invite-class policies (login,
   reset, invite preview/accept, child link, PIN) and fail **open** for
   content-class policies — implement + test both branches; log loudly.
6. Docs/limits sync: unit test asserts every key in `RATE_LIMITS` whose
   `scope` is NOT `'edge'` is registered in the application map (edge
   keys like `globalHttp` belong to Caddy, issue 49, and are excluded
   here); no dead policies, no unmapped API keys — forward-declared keys
   like `messageSend` map to a placeholder matcher list that later
   issues extend; the test asserts list membership explicitly so
   14/17/19/28 must update it consciously.
7. Tests: 6th login within window → 429/`RATE_LIMITED` with retryAfter;
   per-account limit isolates users behind one IP; child-link brute force
   trips at 5; headers present on `/healthz`; body > 1 MiB rejected 413.

## Acceptance Criteria

- [ ] All DESIGN §19.5 API-side rows enforceable via the single policy map;
      06–09 local limiters removed.
- [ ] Redis-down behavior matches the fail-closed/fail-open spec with tests.
- [ ] Headers + body limits verified by integration tests.

## Validation

```bash
pnpm --filter @famchat/api test -- -t rate
pnpm --filter @famchat/api test -- -t harden
```

## Dependencies

05, 06, 08, 09 (this issue replaces the route-local limiters those
issues shipped; all their routes must exist).

## Non-goals

Caddy global limits (49), WAF, upload quota accounting (17), web headers
(38), penetration checklist (54).

## Design References

- DESIGN §19.4–19.5 (hardening, limits), §8.3 (error model)
