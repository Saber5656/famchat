# Issue 02: Dev dependencies via Docker Compose

## Summary

Provide `infra/compose.dev.yml` running PostgreSQL 16, Redis 7, MinIO, and
Mailpit with healthchecks and an auto-created bucket, plus `.env.example`
matching the canonical environment matrix.

## Context

All local development and API integration tests run against real backing
services (DESIGN §3.1, §21). Dev compose contains **dependencies only** —
apps run via `pnpm dev` on the host. Production compose is issue 49.

## Scope

In scope: dev compose file, named volumes, healthchecks, MinIO bucket init,
Mailpit, `.env.example`, README dev-setup update.
Out of scope: app containers, Caddy/TLS, backups (49/50).

## Detailed Requirements

1. `infra/compose.dev.yml` services (all ports bound to loopback only,
   `127.0.0.1:<port>:<port>`, so dev dependencies are never LAN-exposed):
   - `postgres`: `postgres:16-alpine`, port `127.0.0.1:5432:5432`, env
     `POSTGRES_USER/PASSWORD/DB = famchat/famchat/famchat` (dev-only
     credentials, see req 3), volume `famchat-dev-pg`, healthcheck
     `pg_isready -U famchat`.
   - `redis`: `redis:7-alpine`, port `127.0.0.1:6379:6379`,
     `--appendonly yes`, volume `famchat-dev-redis`, healthcheck
     `redis-cli ping`.
   - `minio`: `server /data --console-address :9001`, ports
     `127.0.0.1:9000:9000`, `127.0.0.1:9001:9001`, env
     `MINIO_ROOT_USER=${DEV_MINIO_ROOT_USER}` /
     `MINIO_ROOT_PASSWORD=${DEV_MINIO_ROOT_PASSWORD}` interpolated from
     the root `.env` (compose `env_file`), volume `famchat-dev-minio`,
     healthcheck `CMD curl -f http://localhost:9000/minio/health/live`
     (the upstream MinIO image ships curl; if the pinned tag does not,
     use `CMD mc ready local` with the alias configured in the
     entrypoint — pick one and commit it working).
   - `minio-init`: one-shot `mc` container that waits for minio healthy
     and runs `mc mb --ignore-existing local/famchat` (bucket stays
     private; no anonymous policy).
   - `mailpit`: ports `127.0.0.1:1025:1025` (SMTP), `127.0.0.1:8025:8025`
     (UI).
2. Image tags: pin every image to the exact upstream tag current at
   implementation time (e.g. `postgres:16-alpine`, `redis:7-alpine`,
   `minio/minio:RELEASE.YYYY-MM-DD…`, `minio/mc:RELEASE.…`,
   `axllent/mailpit:v1.x`). Never an implicit or explicit `latest`; the
   chosen tags are recorded in the compose file itself.
3. `.env.example` at repo root reproducing the DESIGN §3.4 matrix exactly,
   in two clearly separated blocks:
   - **Dev backing-service credentials** (`DEV_MINIO_ROOT_USER=famchat`,
     `DEV_MINIO_ROOT_PASSWORD=famchat-dev-only`, matching `S3_*` values,
     `DATABASE_URL` for the local postgres, etc.): real working values,
     explicitly commented `# dev-only — never reuse in production`.
   - **Application secrets** (`SESSION_SECRET`, `OPERATOR_TOKEN`,
     `VAPID_*`, production `SMTP_URL`, …): placeholder-only
     (`change-me-…`) with generation one-liners
     (`openssl rand -base64 48`) in the comment header. No application
     secret ships a working default (DESIGN §19.6).
4. README "Development" section gains:
   `docker compose -f infra/compose.dev.yml up -d` and
   `cp .env.example .env` steps, plus the Mailpit (`:8025`) and MinIO
   console (`:9001`) URLs.

## Acceptance Criteria

- [ ] `docker compose -f infra/compose.dev.yml up -d --wait` reaches healthy
      for postgres, redis, minio; minio-init exits 0; mailpit serves `:8025`.
- [ ] Bucket `famchat` exists; unauthenticated object listing/read is
      denied (explicit check below).
- [ ] All published ports are loopback-bound; all image tags exact-pinned.
- [ ] `docker compose … down && up` preserves data (volumes named).
- [ ] `.env.example` contains every variable from DESIGN §3.4 that applies
      to api/worker/web dev; application secrets are placeholders only.

## Validation

```bash
docker compose -f infra/compose.dev.yml up -d --wait
docker compose -f infra/compose.dev.yml ps   # all healthy
curl -sf http://127.0.0.1:9000/minio/health/live
curl -sf http://127.0.0.1:8025 >/dev/null
# authenticated access works (run inside the compose network for portability):
docker compose -f infra/compose.dev.yml run --rm minio-init \
  sh -c 'mc alias set l http://minio:9000 "$DEV_MINIO_ROOT_USER" "$DEV_MINIO_ROOT_PASSWORD" && mc ls l/famchat'
# anonymous access is denied (expect HTTP 403):
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9000/famchat/ | grep -x 403
# loopback binding check:
docker compose -f infra/compose.dev.yml config | grep -E 'published' | grep -v 127.0.0.1 && echo "FAIL: non-loopback port" || echo OK
```

## Dependencies

01.

## Non-goals

Production topology, TLS, resource limits, backup volumes (issues 49–50).

## Design References

- DESIGN §3.1 (components), §3.4 (env matrix), §18 (ops), §21 (integration
  tests use these services)
- ADR-005 (compose-first deployment)
