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
   stdout; rate limit: Caddy's built-in per-IP connection limits
   documented (the 300/min/IP from DESIGN §19.5 implemented via the
   `rate_limit` directive if the plugin is available in the chosen
   image — otherwise document Fastify-only enforcement and mark the
   deviation).
4. `.env.production.example`: full DESIGN §3.4 matrix with production
   guidance comments (secret generation one-liners, FAMCHAT_DOMAIN,
   FAMCHAT_TAG) — no defaults for secrets.
5. Resource hygiene: `restart: unless-stopped` on long-running
   services; log rotation via `logging: options: max-size/max-file`;
   memory limits commented with 4 GB-VPS guidance values (DESIGN
   §18.1).
6. `scripts/prod-smoke.sh`: from a clean VM/host with only Docker:
   validates `.env`, `docker compose -f infra/compose.prod.yml config`
   lints, brings the stack up with a self-signed/internal CA Caddy
   variant (`FAMCHAT_DOMAIN=localhost` internal TLS), waits for
   `/readyz`, runs: operator invite → space create → login → send
   message → upload image → verify `/media` serves — via curl against
   the API (script is the deploy acceptance test and is reused by 53's
   E2E environment).
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

18 (worker image contents), 38 (web assets incl. SW), 35 (ops CLI used
in smoke). 51 publishes the real images.

## Non-goals

Backups (50), CI (51), Prometheus/alerting (v2), multi-host scaling
(v2), CDN.

## Design References

- DESIGN §18.1 (topology), §19.2 (exposure), §3.4 (env); ADR-005;
  ISSUE_PLAN U8
