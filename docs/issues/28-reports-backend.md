# Issue 28: Reports backend

## Summary

Implement member-initiated reporting: the `reports` table, create/queue/
resolve/dismiss APIs, reporter-privacy rules, guardian notifications, and
content snapshots that survive deletion.

## Context

DESIGN §13.6: any member (children especially) can report content or a
member; guardians review; the reported member is never notified
(anti-retaliation default). Reports are the human complement to the
automated NG filter (27).

## Scope

In scope: migration, `reports` router, guardian queue with content
snapshot DTOs, WS + notify events, audit, rate limit, tests.
Out of scope: report UIs (33/47), moderation hits queue (27), operator
visibility (none in v1 — space-internal only).

## Detailed Requirements

1. Migration per DESIGN §7.5 `reports` (reason enum from shared
   `REPORT_REASONS = unkind|scary|inappropriate|other`).
2. `reports.create({ spaceId, targetType: 'message'|'board_post'|
   'board_comment'|'user', targetId, reason, note? ≤ 500 })` — permission
   `report.create` (all roles); target must exist in the same space and be
   visible to the reporter (room member/observer for messages; any member
   for board/user); **exempt from quiet hours** (29 must whitelist this
   procedure — cross-referenced in both issues); self-reports rejected
   `VALIDATION_FAILED`; duplicate open report by same reporter on same
   target is idempotent (returns existing); rate limit `reportCreate`
   10/h/user (12 map). On create: WS `report.created` to
   `space:<id>:guardians`, notify event `report.new` to guardians, audit
   none (reports are not audit events; resolution is).
3. `reports.queue({ spaceId, status?, cursor? })` — guardian-only; DTO:
   reason, note, reporter (id+displayName — guardians may see reporters,
   DESIGN §13.6), createdAt, status, and a **content snapshot**: live
   content DTO when it still exists, else tombstone `{ deleted: true,
   authorId }`; for `user` targets: member summary.
4. `reports.resolve({ spaceId, reportId, note? })` /
   `reports.dismiss({ … })` — guardian; sets status/resolver/timestamp;
   audit `report.resolve` / `report.dismiss`. Resolution does **not**
   auto-delete content — guardians act via existing delete/moderation
   tools (UI links in 31); keep concerns separate.
5. Privacy invariants (tested): non-guardians can never read any report
   (including their own list — v1 has no "my reports" surface; create is
   fire-and-forget with friendly ack); reported member receives no event,
   no notification, no audit visibility; reporter identity appears only in
   guardian queue DTO.
6. Tests: create for each target type incl. visibility guard (cannot
   report a message in a room the reporter can't access); child reporter
   path; idempotent duplicate; self-report rejection; queue filtering +
   snapshot-after-deletion; resolve/dismiss audits; guardian-only
   enforcement; cross-tenant sweep; rate limit trip; quiet-hours exemption
   placeholder test marked to activate with 29 (skip-with-reason).

## Acceptance Criteria

- [ ] All four target types reportable with correct visibility guards.
- [ ] Anti-retaliation invariants proven by tests (no leak paths).
- [ ] Snapshots survive content deletion.
- [ ] Guardian notifications + WS event fire on create.

## Validation

```bash
pnpm --filter @famchat/api test -- --grep reports
```

## Dependencies

13 (visibility), 11 (audit), 19 (board targets). 29 wires the quiet-hours
exemption; 37 delivers notifications.

## Non-goals

Report UIs (33/47), escalation to operator, auto-actions, category ML.

## Design References

- DESIGN §13.6 (flow + privacy), §7.5 (schema), §9.2 (`report.created`),
  §19.5 (rate limit)
