# Issue 33: Web report flow UI (child-friendly)

## Summary

Add the report action to web chat and board surfaces with a child-friendly
reason dialog (icon-based), an adult variant with notes, and reassuring
confirmation states.

## Context

DESIGN §13.6. The reporter is often a 6–12-year-old who feels
uncomfortable: the UI must be low-literacy-friendly, one-handed, and end
with reassurance that a grown-up will see it — without notifying the
reported member (backend guarantees from 28).

## Scope

In scope: report entry points (message menu, board post/comment menus,
member profile sheet), reason dialog (child + adult variants), success/
error states, i18n.
Out of scope: guardian queue UI (31), mobile (47), backend (28).

## Detailed Requirements

1. Entry points: long-press / kebab menu on message bubbles (22), board
   posts and comments (24), and "report this member" on the member
   profile sheet (roster/room member list) — visible to all roles; never
   shown on own content (matches 28 self-report rule).
2. Child dialog (device-link/child sessions): four large icon buttons for
   `REPORT_REASONS` — unkind 😢「いじわる を いわれた」, scary 😨「こわい」,
   inappropriate 🙅「よくない ことば や しゃしん」, other ❓「そのほか」 —
   with furigana; single tap selects + confirm button 「おとなの ひとに
   しらせる」; no free-text for children (privacy + literacy); success
   screen: reassuring copy 「つたえたよ。おうちの おとなが みてくれるよ」
   with a calm illustration placeholder and a close button.
3. Adult dialog: same reasons as labeled radio list + optional note
   (500 cap); success toast.
4. Duplicate handling: idempotent success (28) renders as normal success
   — never "already reported" to a child (documented decision:
   reassurance over precision); adults see a subtle "already sent" toast.
5. Error handling: rate-limit → gentle wait message; offline → fail with
   retry affordance (no offline queue in v1); `QUIET_HOURS_ACTIVE` is
   handled defensively like any mapped error until 29 lands, after which
   the 29 exemption guarantees it never surfaces (29's integration test
   proves it); `details.noReviewer: true` (28's sole-guardian case)
   appends the "talk to another trusted adult" guidance line to the
   success screen. Component boundary (contract for 47's native
   re-implementation): `apps/web/src/components/report/ReportDialog.tsx`
   exporting `ReportDialog({ target: { type: 'message'|'board_post'|
   'board_comment'|'user', id: string }, spaceId, variant:
   'child'|'adult', onDone }: Props)` — no room-context coupling;
   lock-screen contextless reporting is explicitly out of scope (a
   report always targets specific content or a member).
6. i18n keys (namespace `safety.report.*`, ja+en both authored here):
   `reason_unkind` (ja 「いじわる を いわれた」/ en "Someone was
   unkind"), `reason_scary` (「こわい」/ "It's scary"),
   `reason_inappropriate` (「よくない ことば や しゃしん」/ "Bad words
   or pictures"), `reason_other` (「そのほか」/ "Something else"),
   `confirm_cta` (「おとなの ひとに しらせる」/ "Tell a grown-up"),
   `success_child` (「つたえたよ。おうちの おとなが みてくれるよ」/
   "Done! A grown-up in your family will take a look"),
   `success_no_reviewer_hint` (「ほかの しんらいできる おとなにも
   はなしてみてね」/ "Please also talk to another adult you trust").
6. Analytics/telemetry: none (no trackers — DESIGN §22).
7. Tests: component — child vs adult variant switching by session kind,
   reason payload correctness, self-content menus hide report; Playwright:
   child context reports a message → guardian queue (31) shows it with
   correct reason; duplicate tap → single row (idempotency, asserted via
   queue count).

## Acceptance Criteria

- [ ] Report reachable from every content surface in ≤ 2 taps; hidden on
      own content.
- [ ] Child dialog is icon-first, furigana-complete, free-text-free, and
      ends in reassurance.
- [ ] Round-trip to guardian queue proven in E2E; reported member's UI
      shows nothing.
- [ ] ja/en complete for both variants.

## Validation

```bash
pnpm --filter @famchat/web test -- --grep report
pnpm --filter @famchat/web exec playwright test --grep @report
```

## Dependencies

22, 24 (surfaces), 28 (API), 31 (guardian queue UI for the E2E
round-trip assertion). 29 hardens the quiet-hours guarantee later.
Component contract consumed by 47.

## Non-goals

Lock-screen contextless reporting, free text for children, guardian queue
(31), block/mute mechanics (v2 store-compliance item).

## Design References

- DESIGN §13.6 (flow + privacy), §15 (child register), §16 (web)
