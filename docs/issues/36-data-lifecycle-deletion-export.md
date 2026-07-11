# Issue 36: Data lifecycle — deletion, purge, export

## Summary

Implement space deletion with a 7-day grace period, terminal purge jobs
(space data, soft-deleted content, expired rows), owner-requested space
export (JSON + media archive), and the small settings-page UI for
request/cancel/download.

## Context

DESIGN §22 commitments: families can take their data out and delete it for
real. §7.5 `space_exports`; §13.1 owner-only `export.request` and
`space.delete`. Purge jobs run in the worker (18's scaffold).

## Scope

In scope: deletion request/cancel/execute, purge jobs (space, soft-deleted
content 30d, expired sessions/codes/invites/dedupe/exports), export job +
download, danger-zone UI completion (32), audit, tests.
Out of scope: user self-serve deletion (guardian/operator-mediated in v1),
GDPR-grade tooling (v2), backups (50).

## Detailed Requirements

1. Deletion request: `spaces.requestDeletion({ spaceId })` — owner;
   status → `pending_deletion`, `delete_after = now + 7 days`; space
   becomes read-only (writes rejected `VALIDATION_FAILED
   details.pending_deletion`; reads allowed so the family can say
   goodbye/export); WS `space.updated`; notification type
   `space.deletion_requested` to all members (this type extends the
   DESIGN §14.2 catalog — register it in 37's type table; coordination
   note in both issues); audit `space.delete_request`.
   `spaces.cancelDeletion` — owner, while pending; restores active;
   audit `space.delete_cancel`.
2. Space purge job (worker, repeatable hourly `space-purge`): for spaces
   past `delete_after`: delete S3 prefixes (`m/<spaceId>/`, `q/…`,
   `exports/<spaceId>/`), then DB rows via cascading deletes in FK order
   (explicit transaction: content tables → memberships → space; users are
   NOT deleted — child users whose only membership was here are set
   status `deleted` + sessions revoked per 09/30 semantics); final status
   `deleted` tombstone row retained (id, name-hash, deletedAt) for audit
   integrity; instance-level audit `space.purged` (add to catalog);
   idempotent + resumable (S3-first then DB, re-entrant on crash).
3. Soft-deleted content purge (`content-purge`, daily): messages/posts/
   comments with `deleted_at < now - 30 days` → hard-delete rows + any
   attachment objects (and attachment rows) they claimed; dedupe-id
   uniqueness rows older than 24 h cleared implicitly by message purge
   (dedupe window documented in 14).
4. Expiry sweeper (`expiry-sweep`, hourly): hard-delete expired+used
   password_resets, child_link_codes, space_invites (expired > 30 d),
   sessions (expired/revoked > 30 d), stale `space_exports` past
   `expires_at` (+ S3 object).
5. Export: `exports.request({ spaceId })` — owner; one active export at a
   time per space; creates `space_exports` row (pending) + worker job
   `space-export`: builds a zip streamed to
   `exports/<spaceId>/<exportId>.zip` containing `space.json` (space,
   members, rooms, board, reports/moderation summaries — no other
   spaces' data), `rooms/<id>.json` (messages with ISO timestamps,
   sender ids + display names, deleted markers), `board.json`, and
   `media/<attachmentId>-full.webp` files; row → ready with
   `expires_at = now + 7 days`; notify `export.ready` (owner); audit
   `export.request` (+ `export.download` on URL issuance).
   `exports.status`, `exports.downloadUrl` — owner; 302-style presigned
   GET (5-min) via the API (never a public URL); download audited.
6. UI completion in 32's settings danger zone: request/cancel deletion
   states with countdown (unskip 32's Playwright cases); export card:
   request button, progress state, download button with expiry note,
   "one at a time" messaging.
7. Tests: grace-period state machine (request → read-only → cancel →
   restored; request → time-travel → purge job → gone incl. S3 prefixes
   emptied against MinIO); purge idempotency (crash-rerun via fault
   injection between S3 and DB phases); content-purge 30-day boundary;
   expiry sweeper matrix; export content correctness on a seeded family
   (zip parsed in test: counts match DB, no foreign-space leakage, media
   present); export access owner-only + expiry; audit rows throughout.

## Acceptance Criteria

- [ ] Deletion lifecycle exactly: request → 7-day read-only grace →
      purge; cancel restores fully.
- [ ] Purge leaves zero S3 objects and zero content rows for the space
      (tombstone + audit excepted) — machine-verified.
- [ ] Export zip is complete, correct, owner-only, expiring.
- [ ] All jobs idempotent and crash-resumable (fault-injection tests).

## Validation

```bash
pnpm --filter @famchat/worker test -- --grep "purge|export"
pnpm --filter @famchat/api test -- --grep "deletion|exports"
pnpm --filter @famchat/web exec playwright test --grep @dangerzone
```

## Dependencies

18 (worker), 30 (settings surface), 35 (operator context), 32 (UI slots).
Notification types register with 37 (coordination note).

## Non-goals

User self-serve deletion, selective message retention policies, legal-hold,
backup integration (50), COPPA/GDPR programs (v2).

## Design References

- DESIGN §7.5 (space_exports), §13.1 (owner), §22 (commitments), §12
  (media keys), §13.8 (audit)
