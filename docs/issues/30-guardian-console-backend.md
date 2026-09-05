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
   one composed DTO (pinned in `packages/shared/src/api/guardian.ts`):
   `{ child: { userId, displayName, avatarPreset, birthYear, locale },
   devices: DeviceDTO[] (exactly 09's listDevices shape), rooms: [{ id,
   name, type, memberCount }] (all rooms the child is in), quietHours:
   { config, activeNow, until? }, recentHits: ModerationHitDTO[≤10]
   (27's DTO) authored by the child — derivation rule: for content
   types message/board_post/board_comment join the content table on
   content_id and filter author = child; display_name hits carry
   `metadata.userId` and match on that, recentReports:
   ReportQueueDTO[≤10] (28's DTO) where the child is reporter or
   target }`.
1b. `guardian.dashboard({ spaceId })` — guardian; the 31 dashboard's
   single data source: `{ openHits, openReports, children: [{
   childUserId, displayName, avatarPreset, quietNow, deviceCount,
   openHitCount, openReportCount }] }` — one procedure, parallel
   queries.
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
8. Message removal: per DESIGN §8.2 there is deliberately NO
   `guardian.removeMessage` alias — the console (31) calls
   `messages.delete` (guardian `message.deleteAny` path). This issue
   only verifies that trail renders in the guardian audit view.
9. Tests: childOverview + dashboard composition against a seeded family
   (shape snapshots + counts); every members rule above incl.
   LAST_GUARDIAN, owner-protection, owner-transfer atomicity under
   parallel calls; settings validation (bad TZ rejected) +
   timezone-change quiet-state push; role-branched roster DTO;
   cross-tenant sweep; audit rows for each mutation; WS
   `member.updated` emissions observed.

## Acceptance Criteria

- [ ] `childOverview` and `dashboard` return the exact pinned DTO shapes
      in one call each (snapshot tests).
- [ ] `members.list/remove/leave/changeRole` and
      `spaces.transferOwnership` each honor their §13.1 rule with a
      dedicated test (incl. LAST_GUARDIAN, owner-protection).
- [ ] Ownership transfer is atomic and index-safe under race.
- [ ] Settings changes propagate to moderation + quiet-hours behavior.
- [ ] Every mutation writes its audit row and emits `member.updated`
      where applicable.

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/api test -- -t guardian
pnpm --filter @famchat/api test -- -t members
```

## Dependencies

09, 11 (audit rows), 13, 15 (WS events), 27, 28, 29 (aggregated data).

## Non-goals

Console UIs (31/32), space deletion/export (36), child multi-space (v2).

## Design References

- DESIGN §13.2 (surfaces), §13.1 (matrix rows), §5.1 (owner rules), §7.1
  (schema)
