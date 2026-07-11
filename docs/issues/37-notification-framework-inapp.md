# Issue 37: Notification framework + in-app feed

## Summary

Replace the notify stub with the real pipeline: BullMQ fanout in the
worker, per-recipient localized composition, the notifications/
push_subscriptions tables, the in-app feed APIs, unread aggregation, and
quiet-hours/preference suppression — push transports follow in 38/46.

## Context

DESIGN §14.1 (pipeline), §14.2 (type catalog + suppression). Every
feature issue since 05 has been calling `notify.enqueue(event)` against a
stub; this issue makes those calls real without touching the call sites.

## Scope

In scope: migration (notifications, push_subscriptions), enqueue → worker
fanout, composer with i18next, type catalog implementation (incl.
`space.deletion_requested` from 36), feed/markRead/unreadCounts/
registerPush APIs, WS `notification.created`, suppression rules, tests.
Out of scope: Web Push delivery (38), Expo delivery (46), notification UI
(39), email channel (v1 non-goal).

## Detailed Requirements

1. Migration per DESIGN §7.6: `notifications`, `push_subscriptions`
   (unique (kind, endpoint_or_token), failure_count).
2. Producer: `notify.enqueue(event)` now serializes the typed event to
   BullMQ queue `notifications` (jobId dedupe where natural, e.g.
   `message:<messageId>`); api-side stays fire-and-forget post-commit.
3. Worker consumer `notifications` job:
   1. Resolve recipients per DESIGN §14.2 exactly — implement the catalog
      as a table `NOTIFICATION_TYPES: Record<type, { recipients(event),
      suppressedBy(recipient) }>` in `apps/worker/src/notifications/
      catalog.ts` covering: `message.new`, `board.post.new`,
      `board.comment.new`, `moderation.flagged`, `report.new`,
      `child.device.linked`, `member.joined`, `export.ready`,
      `quiet_hours.updated`, `space.deletion_requested` (36).
   2. Per recipient: apply suppression — room `notify=none`
      (message.new), membership `board_notify=none` (board types), child
      quiet hours via 29's `isChildQuietNow` (suppresses **push only**;
      in-app row always written; guardian safety types never suppressed),
      suspended space/user ⇒ drop entirely (35).
   3. INSERT `notifications` row: type, `payload` (i18n params: sender
      name, room name, excerpt ≤ 30 chars — computed **at compose time**,
      accepting staleness; never message ids requiring later joins),
      `link` (app route e.g. `/s/<sid>/r/<rid>?m=<mid>`).
   4. Emit WS `notification.created` to `user:<id>`.
   5. Hand off to transport dispatch: `dispatchPush(recipient,
      rendered)` — this issue implements it as a registry with zero
      transports (38 registers webpush, 46 registers expo); rendering
      happens here: i18next server instance renders title/body in
      `users.locale` from `notifications.*` catalog keys (flagged-content
      types must not include matched terms — compose from category counts
      only, DESIGN §14.1).
4. API (`notifications` router): `feed({ cursor? })` (own, newest-first,
   50/page), `markRead({ ids[] })`, `markAllRead()`, `unreadCounts()` →
   `{ notifications: n, rooms: [{ roomId, unread }] }` (reuses 16's
   query; one aggregate for app badges), `registerPush({ kind, token/
   subscription })` upsert + `unregisterPush` — validation per kind
   (webpush: endpoint+keys shape; expo: `ExponentPushToken[…]` format).
5. Retention: feed rows > 90 days pruned by 36's expiry sweeper (add the
   clause there — cross-issue note; implement the query helper here).
6. Tests: catalog table-driven — for every type: correct recipients,
   correct suppressions (room pref, board pref, quiet-hours child
   push-only suppression with in-app row present, guardian never-suppress,
   suspended drop); locale rendering snapshot ja+en per type (params
   interpolated, no missing keys); excerpt truncation multibyte-safe;
   WS emit observed; feed pagination + markRead idempotency; registerPush
   validation + upsert dedupe; end-to-end: message send → recipient's
   feed row + WS within test harness.

## Acceptance Criteria

- [ ] Every DESIGN §14.2 row (+ `space.deletion_requested`) implemented
      and table-tested for recipients & suppression.
- [ ] All notification text server-rendered in recipient locale; ja/en
      catalogs complete for every type (parity-checked).
- [ ] In-app feed correct and live; unreadCounts aggregates notifications
      + rooms.
- [ ] Transport registry ready for 38/46 with zero behavior change needed
      at their call sites.

## Validation

```bash
pnpm --filter @famchat/worker test -- --grep notif
pnpm --filter @famchat/api test -- --grep notifications
```

## Dependencies

15 (WS), 18 (worker), 29 (suppression), 35 (suspension), 36 (type
coordination).

## Non-goals

Push transports (38/46), UI (39), digests/email, per-type user
preferences beyond the modeled ones (v2).

## Design References

- DESIGN §7.6 (schema), §14.1–14.2 (pipeline, catalog, suppression), §9.2
  (`notification.created`)
