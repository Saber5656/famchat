# ADR-005: Deployment — Docker Compose on a single VPS; images via GHCR

- Status: Accepted (owner-confirmed 2026-07-11)
- Deciders: Owner (Saber5656), Fable (design agent)

## Context

The closed beta is operated by the owner on commodity infrastructure, and the
OSS promise is "self-host = SaaS": the artifact OSS users run must be the
artifact the beta runs. Cloud-specific managed services (Workers/D1, Lambda,
managed queues) would fork the architecture into hosted vs self-host variants.

## Decision

- One `infra/compose.prod.yml` runs the entire stack: Caddy (auto-TLS),
  web, api, worker, PostgreSQL, Redis, MinIO, plus a one-shot `migrate`
  service gating app startup.
- CI publishes versioned images to GHCR; deploy/upgrade = bump tag, `docker
  compose pull && up -d`.
- Target: 1 VPS (≥ 2 vCPU / 4 GB) for the whole beta. Horizontal scale-out is
  out of v1 scope, but nothing blocks it later (socket.io redis adapter,
  stateless api, S3 media).
- Backups are host-cron scripts (`pg_dump` + object mirror) with a tested
  restore runbook.

## Alternatives considered

| Alternative | Why rejected |
|---|---|
| Fly.io / Render PaaS | Config diverges from self-host path; per-service pricing; less instructive for OSS operators |
| Cloudflare Workers stack | Incompatible with Node/Postgres/socket.io design; kills self-hosting |
| Kubernetes | Absurd ops burden at this scale |
| External managed Postgres/S3 | Allowed as a documented variant (env URLs point anywhere), but the default path stays fully self-contained |

## Consequences

- The VPS is a single point of failure; accepted for beta with RPO 24h/RTO 4h.
- MinIO/Postgres/Redis run as containers with volumes — disk sizing and
  monitoring guidance belong in the selfhost guide.
- Everything in the stack must be runnable offline from public clouds except
  push egress (browser push endpoints, Expo API) and SMTP relay.
