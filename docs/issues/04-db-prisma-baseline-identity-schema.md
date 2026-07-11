# Issue 04: `@famchat/db` — Prisma baseline with identity/tenancy schema

## Summary

Create the `packages/db` package with Prisma configured against PostgreSQL,
the initial migration implementing the DESIGN §7.1 identity/tenancy tables,
a shared client singleton, and a seed skeleton.

## Context

DESIGN §7 defines the full data model; this issue lands only §7.1 (users,
sessions, password_resets, instance_invites, spaces, memberships,
space_invites, child_link_codes, child_settings). Feature tables ship with
their feature issues (13, 14, 17, 19, 26–28, 36, 37) as new migrations.

## Scope

In scope: Prisma setup, §7.1 schema + enums, citext, partial unique indexes,
migration, client export, seed skeleton, npm scripts, integration test.
Out of scope: every table from §7.2–§7.6; demo seed content (52).

## Detailed Requirements

1. Scaffold `packages/db` as `@famchat/db` with deps `@prisma/client`,
   `prisma` (dev), importing `@famchat/shared` for env. Scripts:
   `db:generate`, `db:migrate` (`prisma migrate dev`), `db:deploy`
   (`prisma migrate deploy`), `db:seed`, `db:studio`.
2. `prisma/schema.prisma`: datasource postgres from `DATABASE_URL`;
   generator client. **Physical naming rule (applies to this and every
   later migration): DESIGN §7 names are canonical — snake_case tables
   and columns.** Every model uses `@@map("snake_case_table")` and every
   multi-word field `@map("snake_case_column")` (e.g. model `Membership`
   → table `memberships`, field `spaceId` → column `space_id`); all raw
   SQL in migration files uses the snake_case physical names. Models
   exactly per DESIGN §7.1:
   - Enums: `UserKind(adult,child)`, `UserStatus(active,suspended,deleted)`,
     `Locale(ja,en)`, `SessionKind(web,mobile,device_link)`,
     `SpaceStatus(active,suspended,pending_deletion,deleted)`,
     `ModerationMode(flag,block)`, `Role(guardian,adult,child)`,
     `MembershipStatus(active,removed)`, `InviteRole(guardian,adult)`.
   - `User`: id String @id; kind; email String? @unique @db.Citext;
     passwordHash String?; displayName; avatarPreset (default "bear-01");
     locale (default ja); birthYear Int?; status (default active);
     createdAt/updatedAt.
   - `Session`: id, userId FK onDelete Cascade, kind, tokenHash @unique,
     createdAt, expiresAt, lastUsedAt, ip `String? @db.Inet`, userAgent
     String?, revokedAt DateTime?; @@index([userId]),
     @@index([expiresAt]).
   - `PasswordReset`, `InstanceInvite`, `Space`, `Membership`,
     `SpaceInvite`, `ChildLinkCode`, `ChildSettings` — columns/types/
     defaults/uniques exactly as the DESIGN §7.1 tables (memberships
     `@@unique([spaceId, userId])`; spaces include `wordlistRevision Int
     @default(0)` reserved for issue 27; childSettings `quietHours Json?`).
3. The initial migration must additionally contain hand-added SQL (Prisma
   `migration.sql` edits are allowed and reviewed):
   - `CREATE EXTENSION IF NOT EXISTS citext;`
   - Partial unique index: one active owner per space —
     `CREATE UNIQUE INDEX one_owner_per_space ON memberships (space_id)
     WHERE is_owner = true AND status = 'active';`
4. `src/client.ts`: the standard Prisma dev-safe singleton — cache the
   client on `globalThis` (`const db = globalThis.__famchatDb ??= new
   PrismaClient(...)`) so hot-reload/test re-imports never create extra
   connection pools; query-event logging enabled only when
   `NODE_ENV=development`.
5. `src/seed.ts` skeleton: when `SEED_MODE=dev`, creates one instance
   invite whose code follows the issue-07 format (`fi_` + 16 random bytes
   base64url = 128-bit entropy, printed to stdout exactly once, stored as
   sha256). Full demo data is issue 52 — leave a marked extension point.
6. Integration test (vitest, requires issue 02 services): migration deploys
   on a clean DB; creating two active owner memberships in one space
   violates the partial index; email uniqueness is case-insensitive
   (citext).
7. Document in package README: "schema changes only via
   `prisma migrate dev`; raw SQL only inside migration files" (DESIGN §4,
   ADR-001).

## Acceptance Criteria

- [ ] `pnpm --filter @famchat/db db:deploy` succeeds on a fresh postgres.
- [ ] All §7.1 tables/enums/indexes exist exactly as specified (verified by
      the integration test using `information_schema` checks for the partial
      index and citext).
- [ ] Owner-uniqueness and citext tests pass.
- [ ] Physical schema is fully snake_case (integration test lists
      `information_schema.tables/columns` and rejects any camelCase name).
- [ ] Package export shape verified: a fixture consumer tsconfig inside
      the package's tests type-checks `import { db } from '@famchat/db'`
      (real consumers arrive in issue 05).

## Validation

```bash
docker compose -f infra/compose.dev.yml up -d --wait
pnpm --filter @famchat/db db:deploy && pnpm --filter @famchat/db test
```

## Dependencies

02, 03.

## Non-goals

Feature tables (§7.2–7.6), demo seed (52), backups (50).

## Design References

- DESIGN §7.1 (schema), §4 (conventions), §5 (tenancy semantics)
- ADR-002 (tenancy), ADR-008 (sessions)
