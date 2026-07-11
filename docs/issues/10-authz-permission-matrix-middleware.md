# Issue 10: Permission matrix + tRPC authz middleware

## Summary

Encode the DESIGN §13.1 permission matrix as data in `@famchat/shared`,
implement `requirePermission` tRPC middleware and membership resolution, and
add the test harness that guarantees every space-scoped procedure is
annotated — replacing the ad-hoc role checks left by issues 07–09.

## Context

DESIGN §13.1: "Matrix is exhaustive in code… no unguarded procedure can
ship." This issue is the single authorization backbone; every later feature
router declares its permission and inherits enforcement plus cross-tenant
denial semantics.

## Scope

In scope: `packages/shared/src/authz.ts`, middleware + context extension,
procedure-annotation registry + meta test, refactor of 07–09 local checks,
cross-tenant test helpers.
Out of scope: room-level access (13 implements `canAccessRoom` using this),
quiet-hours gate (29).

## Detailed Requirements

1. `packages/shared/src/authz.ts`:
   - `export const PERMISSIONS = { 'space.updateSettings': { guardian: true, adult: false, child: false }, … } as const satisfies Record<string, Record<Role, boolean>>`
     — one entry per capability row of DESIGN §13.1, using those exact
     constant names, including owner-only capabilities expressed as
     `{ …, ownerOnly: true }`.
   - `can(role, isOwner, permission): boolean`.
   - Type `Permission = keyof typeof PERMISSIONS`.
2. API middleware (`apps/api/src/trpc.ts`):
   - `spaceProcedure` builder: requires authenticated session; expects
     `input.spaceId` (zod-merged); loads active membership (cached per
     request); throws `NOT_A_MEMBER` when absent, `SPACE_SUSPENDED` when
     space inactive; attaches `ctx.space`, `ctx.membership`.
   - `.permission(p: Permission)` chain: evaluates `can()`; throws
     `PERMISSION_DENIED`. Owner-only honored via `membership.isOwner`.
   - `guardianProcedure`, `ownerProcedure` conveniences derived from it.
3. Annotation registry: each procedure created via these builders registers
   `{ path, permission | 'member' | 'public' | 'authed' }` into an exported
   `PROCEDURE_REGISTRY`. A vitest meta-test walks `appRouter._def` and fails
   if any procedure path is missing from the registry — mechanically
   preventing unguarded endpoints (DESIGN §13.1).
4. Refactor issues 07–09 routers onto the builders; delete every
   `TODO(issue-10)` marker; behavior unchanged (tests still green).
5. Cross-tenant helper in the test harness: `expectCrossTenantDenied(caller,
   procedure, input)` creating a second space + user and asserting
   `NOT_A_MEMBER`/`PERMISSION_DENIED`; applied to all existing space-scoped
   procedures and mandated (by convention documented in the harness) for
   every future router's tests (ISSUE_PLAN §6.8).
6. Matrix-driven unit tests: table test over every (permission × role ×
   isOwner) combination asserting `can()` equals the DESIGN §13.1 table —
   hardcode expectations; a mismatch means code or design drifted.

## Acceptance Criteria

- [ ] `PERMISSIONS` covers every DESIGN §13.1 row with identical semantics
      (unit-test enforced table).
- [ ] Meta-test fails when a new unannotated procedure is added (prove by
      temporarily adding one in a test fixture router).
- [ ] All 07–09 procedures migrated; their suites still pass; cross-tenant
      denial tests green.
- [ ] `PERMISSION_DENIED` vs `NOT_A_MEMBER` vs `SPACE_SUSPENDED` returned
      in the correct precedence (membership first, then suspension, then
      permission).

## Validation

```bash
pnpm --filter @famchat/shared test -- --grep authz
pnpm --filter @famchat/api test
```

## Dependencies

07 (uses existing routers to refactor); 08, 09 recommended merged first.

## Non-goals

Room-level `canAccessRoom` (13), quiet-hours enforcement (29), per-object
ACLs (not in v1 design).

## Design References

- DESIGN §13.1 (matrix), §8.4 (mutation pipeline order), §5.3 (roles);
  ADR-002
