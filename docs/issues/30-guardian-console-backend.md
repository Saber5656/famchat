# Issue 30: Guardian console backend

## Summary

Implement the guardian console aggregate APIs: per-child overview, member
role management and ownership transfer, full space settings (safety fields
included), and guardian message-removal glue — completing the backend
surface that 31/32 render.

## Context

DESIGN §13.2 lists the console surfaces. Individual capabilities exist
across 09/11/13/14/27/28/29; this issue adds the aggregates and the
remaining member/space management procedures per the §13.1 matrix.

## Scope

In scope: `guardian.childOverview`, `members` router completion (remove/
changeRole/leave/transferOwnership), full `spaces.updateSettings`,
`guardian.removeMessage` alias audit, tests.
Out of scope: UIs (31/32), deletion lifecycle (36), quiet-hours setter
(29 shipped it).

## Detailed Requirements

1. `guardian.childOverview({ spaceId, childUserId })` — guardian; returns
   one composed DTO:
   `{ child: { profile, birthYear, avatarPreset }, devices: [session
   summaries from 09], rooms: [{ id, name, type, memberCount }] (all rooms
   the child is in), quietHours: { config, activeNow, until? },
   recentHits: last 10 moderation_hits involving the child as author,
   recentReports: last 10 reports where the child is reporter or target
   (target identity per 28 privacy rules), counts: { openHits,
   openReports } }` — single procedure, parallel queries, one round trip
   for the 31 dashboard.
2. `members.list({ spaceId })` — any member; roster with roles, isOwner,
   joinedAt, child birthYear visible to guardians only (DTO branches by
   requester role — snapshot-tested).
3. `members.remove({ spaceId, userId })` — guardian; cannot remove: self
   (use leave), any guardian unless requester is owner, the owner ever;
   removing a child delegates to 09's child removal semantics; removes
   room memberships via 13's maintenance; audit `member.remove`; WS
   `member.updated`.
4. `members.leave({ spaceId })` — guardian/adult; last-guardian guard
   (`LAST_GUARDIAN`); owner must transfer first (`VALIDATION_FAILED
   details.owner_must_transfer`).
5. `members.changeRole({ spaceId, userId, role: 'guardian'|'adult' })` —
   **owner only** (matrix); child roles immutable; demoting the last other
   guardian allowed (owner remains guardian); audit `member.role_change`.
6. `spaces.transferOwnership({ spaceId, toUserId })` — owner → active
   guardian membership only; atomic swap of `isOwner` flags (single
   transaction, respects the partial unique index from 04); audit
   `space.ownership_transfer`.
7. `spaces.updateSettings` — extend (from 07's name-only) to the full
   guardian-editable set: `name, timezone (IANA-validated), defaultLocale,
   moderationMode, ngBuiltinJa, ngBuiltinEn`; timezone changes affect
   quiet-hours evaluation immediately (recompute + push states via 29's
   machinery); moderation fields take effect next write (27 reads the
   row); audit `space.settings_update` with a changed-fields diff in
   metadata (values for non-sensitive fields only).
8. `guardian.removeMessage` — thin alias over 14's guardian delete kept
   for console ergonomics? **No**: do not add a duplicate procedure;
   instead document in the router that 31 calls `messages.delete` — this
   issue only verifies the audit trail (`message.delete_any`) renders in
   the guardian audit view (11). (Explicit non-duplication decision.)
9. Tests: childOverview composition against a seeded family (shape
   snapshot + counts); every members rule above incl. LAST_GUARDIAN,
   owner-protection, owner-transfer atomicity under parallel calls;
   settings validation (bad TZ rejected) + timezone-change quiet-state
   push; role-branched roster DTO; cross-tenant sweep; audit rows for
   each mutation.

## Acceptance Criteria

- [ ] childOverview returns the exact composed shape in one call.
- [ ] Member management honors every §13.1 owner/guardian rule with tests
      per rule.
- [ ] Ownership transfer is atomic and index-safe under race.
- [ ] Settings changes propagate to moderation + quiet-hours behavior.

## Validation

```bash
pnpm --filter @famchat/api test -- --grep "guardian|members|updateSettings"
```

## Dependencies

27, 28, 29 (aggregated data), 09, 13.

## Non-goals

Console UIs (31/32), space deletion/export (36), child multi-space (v2).

## Design References

- DESIGN §13.2 (surfaces), §13.1 (matrix rows), §5.1 (owner rules), §7.1
  (schema)
