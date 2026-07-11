# Issue 11: Audit log foundation

## Summary

Add the append-only `audit_logs` table, an `auditService` implementing the
interface stubbed in issue 06, the guardian-facing space-scoped audit query,
and wiring for all events emitted by issues 06–09.

## Context

DESIGN §7.5 (schema), §13.8 (action catalog + visibility rules). Audit is a
core safety/trust feature (ADR-003 compensating control): guardians see what
happened in their space; operators see instance-level events (35).

## Scope

In scope: migration, write service, guardian query, retro-wiring 06–09
events, tests.
Out of scope: operator audit endpoint (35), UI (31), retention/purge (36).

## Detailed Requirements

1. Migration `audit_logs` exactly per DESIGN §7.5: id, spaceId nullable FK
   (SET NULL on space delete is **wrong** for audit — use no FK constraint,
   plain text column, so audit survives space deletion; document this
   deviation in the migration comment and in DESIGN§7.5 terms it is
   "append-only"), actorKind enum(user,operator,system), actorId text?,
   action text, targetType text?, targetId text?, metadata jsonb, ip inet?,
   createdAt. Indexes: (spaceId, createdAt DESC), (action).
2. `apps/api/src/services/audit.ts`: `audit.log({ spaceId?, actorKind,
   actorId?, action: AuditAction, targetType?, targetId?, metadata?, ip? })`
   — validates `action` against `AUDIT_ACTIONS` (03); inserts within the
   caller's transaction when one is active (accept optional `tx`);
   **never throws into the business flow** on failure (log error + continue)
   except in tests where strict mode asserts.
3. No update/delete code paths exist for this table anywhere (append-only
   enforced by convention + a test asserting the Prisma model is not
   referenced with `.update`/`.delete` via a grep-style meta check in CI
   script `scripts/check-audit-append-only.mjs`).
4. `guardian.auditLog({ spaceId, cursor?, action? })` — guardian-gated
   (issue-10 builder), returns only `GUARDIAN_VISIBLE_ACTIONS ∩ space
   events`; never exposes `auth.*` events of other users nor instance-level
   rows; paginated 50/page.
5. Wire real persistence into the emit sites from 06 (auth events — stored
   with `spaceId = null`, visible only to the acting user via `auth`-scope
   rule), 07 (space.create, invite.accept), 08 (invite.*, member.joined via
   `invite.accept`), 09 (child.*). Remove the temporary stub.
6. Tests: every wired action produces exactly one row with correct actor/
   target/metadata (table-driven across flows); guardian query hides
   instance-level and non-visible actions; cursor pagination stable;
   append-only check script passes.

## Acceptance Criteria

- [ ] All 06–09 flows write audit rows (test-proven, table-driven).
- [ ] Guardian query respects the visibility subset exactly.
- [ ] Append-only meta-check wired into `pnpm test` at root.
- [ ] Audit failures never break the parent mutation (fault-injection test).

## Validation

```bash
pnpm --filter @famchat/api test -- --grep audit
node scripts/check-audit-append-only.mjs
```

## Dependencies

10 (guardian gating), 06–09 (emit sites).

## Non-goals

Operator queries (35), UI (31), export inclusion (36), SIEM shipping.

## Design References

- DESIGN §7.5 (audit_logs), §13.8 (catalog + visibility), §19.2 (controls)
