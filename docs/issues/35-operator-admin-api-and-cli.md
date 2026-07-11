# Issue 35: Operator admin REST API + `pnpm ops` CLI

## Summary

Implement the instance operator surface: the token-guarded `/admin/v1`
REST API (metadata-only, fully audited) and the `scripts/ops.mjs` CLI
wrapper for instance invites, suspension, stats, and audit tailing.

## Context

DESIGN §13.7: operators administer the instance without any content
access — the admin API returns metadata only; the governance policy for
emergency DB access lives in 56's policy doc. ADR-006: instance invites
are the closed-beta gate (07 consumes them; this issue finally lets a
real operator mint them).

## Scope

In scope: admin fastify plugin (auth, no-CORS, audit), endpoints per
§13.7, suspension semantics, CLI, ops docs stub.
Out of scope: content endpoints (never), admin web console (v2), stats
dashboards (v2), user data export (36 handles space export via owner).

## Detailed Requirements

1. Plugin `apps/api/src/rest/admin.ts`, prefix `/admin/v1`:
   - Auth: `Authorization: Bearer <OPERATOR_TOKEN>`; constant-time
     comparison (`crypto.timingSafeEqual` over sha256 digests).
     `OPERATOR_TOKEN` is **optional** in the env schema (canonical:
     DESIGN §3.4; this issue also relaxes it in 03's `apiEnvSchema` —
     min-32-bytes validated only when present): unset ⇒ the plugin
     registers a 404 catch-all (surface absent, not 401 — probing
     yields nothing; self-hosts may run without operator tooling).
   - No CORS headers ever on this prefix; `Cache-Control: no-store`;
     rate limit `adminApi` 60/min/token (12 map).
   - Every request audited (`actor_kind='operator'`, action
     `admin.<method>_<path>` normalized, metadata: params minus body
     secrets, ip).
2. Endpoints (zod-validated, JSON):
   - `POST /instance-invites { note? ≤ 200, maxUses? 1–100 (default 1),
     expiresInDays? 1–90 (default 30) }` → `{ id, code }` (code
     returned exactly once; stored hashed; format from 07).
   - `GET /instance-invites` → `[{ id, status, note, maxUses,
     usedCount, expiresAt, createdAt }]` — never codes.
   - `POST /instance-invites/:id/revoke`.
   - `GET /spaces?cursor=` → id, name, status, memberCounts by role,
     createdAt, mediaBytesUsed — **no names of members, no content**.
   - `POST /spaces/:id/suspend { reason }` / `POST /spaces/:id/unsuspend`
     — sets space status; suspended semantics: every space-scoped
     procedure returns `SPACE_SUSPENDED` (guard from 07), WS space
     channels closed (emit `member.updated` + force-leave via 15's
     machinery), push/notify fanout for the space halted (37 checks
     status).
   - `POST /users/:id/suspend { reason }` / `unsuspend` — user status;
     suspension revokes all sessions (06 service) and blocks login.
   - `GET /stats` → counts: spaces by status, users by kind/status,
     messages last 24h/total, attachments bytes, queue depths (BullMQ
     counts via worker-shared Redis).
   - `GET /audit?spaceId=&cursor=` — instance-level audit reader (full
     catalog incl. `admin.*`; this is the operator view, distinct from
     the guardian-visible subset). Metadata in responses passes through
     the shared `AUDIT_METADATA_ALLOWLIST` filter (11) — the
     content-free invariant applies to audit metadata exactly as to
     every other admin DTO.
3. CLI `scripts/ops.mjs` (Node ≥ 22, no deps beyond `node:` builtins;
   reads `FAMCHAT_API_URL` + `OPERATOR_TOKEN` from env or `.env`):
   subcommands `instance-invite create|list|revoke`, `spaces list`,
   `space suspend|unsuspend <id>`, `user suspend|unsuspend <id>`,
   `stats`, `audit [--space <id>] [--follow]`; table output; non-zero
   exit on API error; `pnpm ops -- <args>` root script.
4. Docs stub `docs/ops/admin.md`: token generation, first invite, CLI
   examples, the no-content-access policy pointer.
5. Tests: 404-when-unset; wrong token constant-time 401 (behavioral: both
   wrong-length and right-length-wrong-value take same path); every
   endpoint happy + validation error; suspension effects (space: member
   API calls fail `SPACE_SUSPENDED`, WS kicked; user: sessions dead,
   login blocked; unsuspend restores); audit rows for each call; stats
   shape; **meta-test: no admin endpoint response contains message/post
   bodies or media keys** (snapshot deny-list scan of all admin DTOs).

## Acceptance Criteria

- [ ] Full operator loop works via CLI on the dev stack: mint invite →
      (07) create space → suspend → verify lockout → unsuspend.
- [ ] Content-free invariant machine-checked over every admin response.
- [ ] Token handling: constant-time, absent-env = absent surface, never
      logged.
- [ ] All calls audited and visible via `GET /audit`.

## Validation

```bash
pnpm --filter @famchat/api test -- --grep admin
pnpm ops -- stats   # against dev stack
```

## Dependencies

06 (session revocation), 07 (invites consumed), 10 (suspension guard),
11 (audit), 12 (rate limit), 15 (WS ejection machinery). Fanout halt
checked again in 37.

## Non-goals

Content access (permanent non-goal per ADR-003/§13.7), admin web UI (v2),
metrics dashboards (v2), multi-operator RBAC (v2).

## Design References

- DESIGN §13.7 (admin), §13.8 (audit), §19.2 (admin boundary), §19.5
  (limits); ADR-006
