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

1. `infra/compose.dev.yml` services:
   - `postgres`: `postgres:16-alpine`, port `5432:5432`, env
     `POSTGRES_USER/PASSWORD/DB = famchat/famchat/famchat`, volume
     `famchat-dev-pg`, healthcheck `pg_isready -U famchat`.
   - `redis`: `redis:7-alpine`, port `6379:6379`, `--appendonly yes`,
     volume `famchat-dev-redis`, healthcheck `redis-cli ping`.
   - `minio`: `minio/minio`, `server /data --console-address :9001`, ports
     `9000:9000`, `9001:9001`, env `MINIO_ROOT_USER=famchat`,
     `MINIO_ROOT_PASSWORD=famchat-dev-secret`, volume `famchat-dev-minio`,
     healthcheck via `mc ready local` or curl `/minio/health/live`.
   - `minio-init`: one-shot `minio/mc` container that waits for minio and
     runs `mc mb --ignore-existing local/famchat` (bucket stays private; no
     anonymous policy).
   - `mailpit`: `axllent/mailpit`, ports `1025:1025` (SMTP), `8025:8025`
     (UI).
2. All service versions pinned by major tag as above (no `latest`).
3. `.env.example` at repo root reproducing the DESIGN §3.4 matrix exactly,
   with dev-working defaults pointing at the compose services
   (`localhost:5432` etc.), placeholder-only secrets (`change-me-32-bytes`),
   and a comment header explaining generation one-liners
   (`openssl rand -base64 48`).
4. README "Development" section gains:
   `docker compose -f infra/compose.dev.yml up -d` and
   `cp .env.example .env` steps, plus the Mailpit (`:8025`) and MinIO
   console (`:9001`) URLs.

## Acceptance Criteria

- [ ] `docker compose -f infra/compose.dev.yml up -d --wait` reaches healthy
      for postgres, redis, minio; minio-init exits 0; mailpit serves `:8025`.
- [ ] Bucket `famchat` exists and is not anonymously readable.
- [ ] `docker compose … down && up` preserves data (volumes named).
- [ ] `.env.example` contains every variable from DESIGN §3.4 that applies
      to api/worker/web dev, and no real secret values.

## Validation

```bash
docker compose -f infra/compose.dev.yml up -d --wait
docker compose -f infra/compose.dev.yml ps   # all healthy
curl -sf localhost:9000/minio/health/live
curl -sf localhost:8025 >/dev/null
docker run --rm --network host minio/mc sh -c \
  "mc alias set l http://localhost:9000 famchat famchat-dev-secret && mc ls l/famchat"
```

## Dependencies

01.

## Non-goals

Production topology, TLS, resource limits, backup volumes (issues 49–50).

## Design References

- DESIGN §3.1 (components), §3.4 (env matrix), §18 (ops), §21 (integration
  tests use these services)
- ADR-005 (compose-first deployment)
