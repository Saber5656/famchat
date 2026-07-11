# Issue 31: Web guardian dashboard UI

## Summary

Build the guardian console's monitoring side on web: dashboard with open
flag/report counts, the flag queue with approve/remove actions, the report
queue with resolve/dismiss, per-child overview pages, and the space audit
log view.

## Context

DESIGN §13.2 (console surfaces) and §16 (routes under
`/s/[spaceId]/guardian/**`). Backend complete after 30; realtime safety
events arrive on the guardian channel (15/27/28).

## Scope

In scope: guardian route group + role guard, dashboard, moderation queue,
report queue, child overview screen, audit view.
Out of scope: members/invites/children management + space settings forms
(32), quiet-hours editor (34), mobile guardian tab (47).

## Detailed Requirements

1. Route guard: `/s/[spaceId]/guardian` layout renders only for guardian
   memberships (client guard + server components rely on API which
   enforces anyway); non-guardians get a localized not-available page (no
   existence details).
2. Dashboard (`/s/[spaceId]/guardian`): single data source =
   `guardian.dashboard` (30's aggregate DTO) — stat cards (openHits,
   openReports) + per-child mini-status cards (quietNow, deviceCount,
   open counts); latest 5 flags + 5 reports preview lists (first page of
   the queue queries); live updates via `moderation.flagged` /
   `report.created` WS events (increment + prepend, then invalidate the
   dashboard query).
3. Flag queue (`/guardian/moderation`): filter tabs
   (pending/approved/removed); rows: content snapshot (text excerpt /
   image thumb / blocked marker), author chip, matched-category count
   (never the terms in list view; expandable detail shows matched terms
   to guardians only — API provides them per 27), source room/post link;
   actions: approve / remove (confirm) → optimistic status move; empty
   states.
4. Report queue (`/guardian/reports`): open/closed tabs; rows: reason icon
   + label (kid-reason iconography consistent with 33), reporter chip,
   target snapshot or tombstone, note; detail drawer: full context +
   "open in room/board" deep link + actions resolve/dismiss with optional
   note; anti-retaliation reminder copy (safety.\* namespace).
5. Child overview (`/s/[spaceId]/guardian/children/[childUserId]`):
   renders 30's composed DTO — profile header, devices list (revoke
   button with confirm; "linked at/last used"), rooms list (deep
   links), quiet-hours summary card **read-only in this issue** (the
   edit affordance and editor route are added by issue 34 — no dead
   link ships here), recent hits/reports lists; revoke actions wired to
   09.
6. Audit view (`/s/[spaceId]/guardian/audit`): paged table of
   `guardian.auditLog` (localized action labels from a new `audit.*`
   i18n namespace mapping every guardian-visible action constant;
   metadata rendered as key:value chips **only for keys present in the
   shared `AUDIT_METADATA_ALLOWLIST`** — unknown keys are never
   rendered, so future producers cannot leak values into the UI),
   action filter dropdown, relative + absolute timestamps in space
   timezone.
7. All strings ja/en; this is an adult surface — standard register, no
   furigana requirement.
8. Tests: role-guard redirect; queue action flows with optimistic
   rollback on API error; WS-driven live increments (mock socket);
   Playwright: seeded flagged message → appears on dashboard + queue →
   remove → message tombstones in chat (cross-surface assertion); report
   lifecycle resolve path; audit table renders latest actions after the
   above (integration chain).

## Acceptance Criteria

- [ ] All five routes functional with the named actions:
      `/s/[id]/guardian` (dashboard cards + previews),
      `…/guardian/moderation` (approve/remove), `…/guardian/reports`
      (resolve/dismiss), `…/guardian/children/[id]` (device revoke),
      `…/guardian/audit` (filter + paginate).
- [ ] Queue actions round-trip and reflect in chat/board content state.
- [ ] Live updates without reload via guardian channel events.
- [ ] Localized action labels exist for the entire guardian-visible audit
      catalog (i18n parity test counts keys vs constants); metadata chips
      allowlist-gated.

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/web test -- -t guardian
pnpm --filter @famchat/web exec playwright test --grep @guardian
```

## Dependencies

30 (APIs incl. dashboard), 22 (chat deep links), 24 (board deep links),
27, 28.

## Non-goals

Admin/settings forms (32), quiet-hours editing (34), operator views (35),
charts/analytics.

## Design References

- DESIGN §13.2 (console), §13.4/§13.6 (queues), §13.8 (audit visibility),
  §16 (routes)
