# Issue 49: Production compose + Caddy TLS + migrate gate

## Summary

Create the production deployment artifact: `infra/compose.prod.yml`
running the full stack behind Caddy with automatic TLS, a one-shot
migration service gating app startup, healthchecks, and the deploy/
upgrade/rollback runbook.

## Context

ADR-005: self-host = SaaS; one compose file is the product's deployment
story. DESIGN §18.1 fixes the topology and §19.2 the exposure rules
(only 80/443 published). Images come from GHCR (published by 51 — until
then the compose file supports `build:` fallback for local validation).

## Scope

In scope: compose.prod.yml, Caddyfile, `.env.production.example`,
migrate gating, healthchecks, deploy runbook, websocket keepalive (U8),
prod smoke script.
Out of scope: backups (50), CI image publishing (51), monitoring stack
(v2).

## Detailed Requirements

1. `infra/compose.prod.yml` services (DESIGN §18.1): `caddy` (ports
   80/443 only published ports in the file; volumes for caddy data/
   config), `web`, `api`, `worker`, `postgres:16` (volume), `redis:7`
   (AOF, volume), `minio` (volume + console NOT published), `migrate`
   (api image, command `pnpm --filter @famchat/db db:deploy`,
   `restart: "no"`), `minio-init` (bucket, as dev). App images:
   `image: ghcr.io/saber5656/famchat-{api,worker,web}:${FAMCHAT_TAG}`
   with commented `build:` blocks for pre-51 local validation.
2. Startup ordering: postgres/redis/minio healthy → migrate runs to
   completion (`depends_on: condition: service_completed_successfully`)
   → api/worker/web start → caddy last (depends on api healthy via
   `/readyz`-backed container healthcheck).
3. `infra/caddy/Caddyfile`: single site `{$FAMCHAT_DOMAIN}`; routes:
   `/trpc* /ws* /media* /healthz /readyz /admin*` → `api:8080` (with
   `@admin` matcher wrapped in an optional IP-allowlist snippet
   documented inline), everything else → `web:3000`; websocket upgrade
   handled by Caddy's reverse_proxy defaults + explicit long
   `stream_timeout`/keepalive settings for `/ws` (resolves unknown U8 —
   record chosen values + rationale in the file); global: HSTS
   (max-age 1y) on responses, `encode zstd gzip`, access logs JSON to
   stdout; rate limit: the DESIGN §19.5 global `300/min/IP` is
   **mandatory** — build a custom Caddy image (`infra/caddy/Dockerfile`,
   xcaddy pinned by digest, `--with github.com/mholt/caddy-ratelimit`
   pinned to a tagged version) and configure the `rate_limit` directive;
   stock-Caddy fallback is not permitted (no silent deviation from the
   canonical limits table).
4. `.env.production.example`: full DESIGN §3.4 matrix with production
   guidance comments (secret generation one-liners, FAMCHAT_DOMAIN,
   FAMCHAT_TAG) — no defaults for secrets.
5. Resource hygiene: `restart: unless-stopped` on long-running
   services; log rotation via `logging: options: max-size/max-file`;
   memory limits commented with 4 GB-VPS guidance values (DESIGN
   §18.1).
6. `scripts/prod-smoke.sh` — from a clean VM/host with only Docker; the
   script is pure curl+jq against the API (no pnpm/ops-CLI dependency;
   the 35 admin REST endpoints are called directly with
   `OPERATOR_TOKEN`). TLS variant selection: env `FAMCHAT_TLS_MODE`
   (`auto` default / `internal`) consumed by the Caddyfile via an
   `import` snippet — `internal` adds `tls internal` for
   `FAMCHAT_DOMAIN=localhost` runs without touching production config;
   53 reuses the same mechanism. Exact steps (each with an explicit
   jq assertion and expected status):
   1. `docker compose -f infra/compose.prod.yml config -q` (lint).
   2. `up -d` → poll `GET /readyz` ≤ 120 s → `.ok == true`.
   3. `POST /admin/v1/instance-invites` (Bearer $OPERATOR_TOKEN,
      `{"note":"smoke"}`) → 200, capture `.code`.
   4. tRPC `spaces.create` (curl POST `/trpc/spaces.create`) with the
      code + `newUser {email: smoke@example.com, password: <random>,
      displayName: Smoke, locale: en, tosAccepted: true}` → capture
      session cookie + `spaceId`.
   5. `rooms.list` → family room exists; `messages.sendText` → 200;
      `messages.list` → the message round-trips.
   6. `attachments.requestUpload` (fixture jpeg bytes from
      `scripts/fixtures/smoke.jpg`, committed) → PUT → `finalize` →
      poll `attachments.get` until `ready` (≤ 60 s) → GET
      `/media/<id>/thumb` follows 302 → 200 + `image/webp`.
   7. `docker compose down`; exit 0 only if every assertion passed.
7. `docs/ops/deploy.md`: first deploy (DNS, ports, env, up), upgrade
   (edit FAMCHAT_TAG → pull → up -d; migrate auto-runs; observed-safe
   ordering), rollback (previous tag + the migration-rollback caveat:
   prisma migrations are forward-only — rollback = restore from backup,
   pointer to 50), troubleshooting (cert issuance, port 443 busy,
   websocket drops).

## Acceptance Criteria

- [ ] `prod-smoke.sh` passes on a fresh Linux VM (evidence: transcript
      in PR) with locally-built images.
- [ ] Only 80/443 reachable from outside (`docker compose config`
      assertion in smoke script + manual `nmap`-style check documented).
- [ ] Kill-test: `docker compose restart api` mid-chat recovers web
      clients via WS reconnect (manual step in runbook validation).
- [ ] Migrate gate provably blocks app start on failed migration
      (smoke-script negative test with a poisoned migration in a
      scratch branch — documented procedure).

## Validation

```bash
docker compose -f infra/compose.prod.yml config
bash scripts/prod-smoke.sh   # on a scratch VM
```

## Dependencies

18 (worker image contents), 35 (admin REST endpoints the smoke script
calls), 38 (web assets incl. SW). 51 publishes the real images.

## Non-goals

Backups (50), CI (51), Prometheus/alerting (v2), multi-host scaling
(v2), CDN.

## Design References

- DESIGN §18.1 (topology), §19.2 (exposure), §3.4 (env); ADR-005;
  ISSUE_PLAN U8
