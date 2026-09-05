# ADR-001: TypeScript full-stack monorepo and core stack

- Status: Accepted (owner-confirmed 2026-07-11)
- Deciders: Owner (Saber5656), Fable (design agent)

## Context

famchat spans a web app, a mobile app, an API server, and a background worker.
Implementation will be executed mostly by lower-capability coding agents from
granular issues, so the stack must minimize cross-language contract drift and
favor mainstream, heavily documented libraries. The owner confirmed
TypeScript full-stack as the preferred direction.

## Decision

One pnpm-workspaces monorepo, TypeScript strict everywhere:

| Concern | Choice |
|---|---|
| API server | Fastify 5 + tRPC 11 (zod inputs/outputs) |
| Realtime | socket.io 4 (+ redis adapter) |
| ORM / migrations | Prisma 6 / prisma-migrate |
| Web | Next.js 15 (App Router) + TanStack Query + Tailwind |
| Mobile | Expo (React Native) + expo-router |
| Queue | BullMQ on Redis 7 |
| DB / storage | PostgreSQL 16 / S3-compatible (MinIO) |
| i18n | i18next on web, mobile, and server |
| Build orchestration | plain pnpm scripts (no Turborepo in v1) |

Shared zod schemas and constants live in `packages/shared`; API types flow to
clients via tRPC's `AppRouter` type-only import — no codegen step.

## Alternatives considered

| Alternative | Why rejected |
|---|---|
| Go backend + TS front | Duplicate type definitions; OpenAPI codegen adds a drift-prone step; Go's perf advantage is irrelevant at family scale |
| NestJS | Heavier abstraction layers for agents to misuse; Fastify+tRPC is flatter |
| REST + OpenAPI instead of tRPC | Codegen pipeline is an extra failure point for weak agents; tRPC gives compile-time contract enforcement in one language |
| Drizzle ORM | Fine choice, but Prisma's generated client + migrate workflow is more mechanical for implementation agents |
| Turborepo | Caching benefits don't justify config surface at this repo size |

## Consequences

- Single `node_modules` universe: one lockfile to audit (supply-chain review is simpler), but Node is a required runtime for every service.
- tRPC couples clients to the server package's types; REST is still provided where third parties need it (`/healthz`, `/admin/v1`, `/media`).
- Prisma migrations become the only sanctioned schema-change path (raw SQL only inside reviewed migration files).
- Minimal-dependency policy applies: adding a runtime dependency requires stating why in the PR description.
