# Issue 32: Web space admin UI (members, invites, children, settings)

## Summary

Build the guardian console's management side on web: member roster with
role/ownership actions, invite lifecycle with QR/link display, child
account management including the device-link code screen and PIN controls,
and the space settings form (safety fields, custom NG words). The
deletion/export danger zone is issue 36's.

## Context

DESIGN §13.2, §5.2 (flows), §16 (routes). This screen is where the family
is actually assembled: it must make invite/link codes effortless to hand
to a grandparent or child while keeping their secrecy semantics visible.

## Scope

In scope: `/s/[spaceId]/guardian/members`, `…/guardian/invites`,
`…/guardian/children` (+ child create/edit/link modals),
`…/guardian/settings` (safety settings form + custom NG words).
Out of scope: monitoring surfaces (31), quiet-hours editor (34), the
settings danger zone — space deletion request/cancel and export UI ship
entirely with issue 36 (this page leaves a marked slot), operator
tooling (35).

## Detailed Requirements

1. Members (`/s/[spaceId]/guardian/members`): roster from `members.list` (role badges,
   owner crown, joined date); actions per matrix rendered only when
   permitted: remove (confirm, guardian-removal rules from 30 surfaced as
   disabled-with-tooltip states), change role (owner only), transfer
   ownership (owner; two-step confirm typing the space name); leave-space
   entry point for self with LAST_GUARDIAN/owner guidance.
2. Invites (`/s/[spaceId]/guardian/invites`): active/expired/used/revoked list from
   08; create dialog (role selector adult/guardian with plain-language
   explanation of guardian powers); creation success screen shows —
   exactly once — the invite URL, a rendered QR (client-side canvas from
   the URL string; add tiny dependency `qrcode` to web only), copy
   button, expiry countdown, and a "this code appears only once" notice;
   revoke with confirm.
3. Children (`/s/[spaceId]/guardian/children`): list with avatar/name/devices count/
   quiet-now badge; create modal (displayName, birthYear picker within
   the 09 range, avatar preset grid, locale); edit modal; remove (typed
   confirm, explains what happens to content per 09/36 semantics); **link
   device flow**: modal showing the 6-digit code huge + a QR that renders
   exactly the backend `qrPayload` string from 09 (the
   `${APP_BASE_URL}/link#…` fragment URL — never a client-reconstructed
   variant) + 10-minute countdown + regenerate button (invalidates
   prior, per 09) + step-by-step instructions for the child device
   (ja/en);
   device list per child with revoke (mirrors 31's child overview,
   shared component); PIN set/reset/clear dialog (explains PIN is a
   sibling lock, not security — copy from ADR-008 wording).
4. Settings (`/s/[spaceId]/guardian/settings`): form for 30's full set —
   name, timezone (searchable IANA select), default locale, moderation
   mode (flag/block with consequence copy), builtin list toggles ja/en;
   custom NG words manager (add input with normalization preview using
   `normalizeText` imported from `@famchat/moderation` — the same
   function, not a copy; list with remove, 500 cap indicator). The page
   ends with a clearly marked extension slot (`<DangerZoneSlot />`
   placeholder component rendering nothing) that issue 36 replaces with
   the deletion/export danger zone — no dead or skipped UI ships in
   this issue.
5. All mutations optimistic-with-rollback; all strings ja/en.
6. Tests: matrix-driven control visibility (owner vs guardian vs
   adult-viewing-roster); invite create → QR renders + code shown once
   (navigating back shows no code); link-code countdown + regenerate;
   custom-word normalization preview equals package output (same import
   — property test over fixtures); Playwright: invite full loop (create
   → second context accepts → roster updates live via `member.updated`),
   child create + link code displayed, settings save persists +
   moderation mode flip changes 27 behavior (send NG term before/after).

## Acceptance Criteria

- [ ] Every management surface of DESIGN §13.2 present and permission-
      faithful.
- [ ] Codes (invite/link) shown exactly once, with QR, countdown, and
      secrecy notices.
- [ ] Custom words UX round-trips through real normalization.
- [ ] Live roster updates on membership changes.

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/web test -- -t admin
pnpm --filter @famchat/web exec playwright test --grep @spaceadmin
```

## Dependencies

08, 09, 21 (shell), 30. (Danger zone/export UI: issue 36.)

## Non-goals

Monitoring queues (31), quiet-hours editor (34), space export
request/download UI (issue 36 adds it to this settings page as a small
follow-through), operator admin (35).

## Design References

- DESIGN §13.2 (console), §5.2 (flows), §6.2 (device link), §13.4 (custom
  words), §16 (routes); ADR-008
