# Issue 40: i18n completeness gate + tooling

## Summary

Make bilingual completeness mechanically enforceable: the catalog parity/
usage checker in CI, extraction-based missing-key detection, locale
switcher polish, shared date/relative-time helpers, and a translator/
contributor guide — then fix every violation that exists at that point.

## Context

ADR-007: ja and en are both first-class; "translation is never a
follow-up task." DESIGN §15 defines catalogs, the child-register policy,
and this gate. By this point waves 0–4 have accumulated real strings —
this issue trues them up and locks the door.

## Scope

In scope: `scripts/check-i18n.mjs`, CI wiring, key-usage extraction,
furigana coverage lint for child namespaces, `Intl` helpers, locale
switcher polish, docs/i18n.md, violation fixes across apps/packages.
Out of scope: adding locales (v2), mobile catalog wiring (41 consumes the
same package), translation of docs/ (55/56 own their bilingual docs).

## Detailed Requirements

1. `scripts/check-i18n.mjs` (Node, no heavy deps) with checks:
   a. **Parity**: every key present in ja exists in en and vice versa,
      per namespace; values non-empty; placeholder tokens (`{{name}}`)
      identical across locales.
   b. **Usage**: extract referenced keys from `apps/web`, `apps/mobile`,
      `apps/api`, `apps/worker` sources via `i18next-parser` config
      (`t('ns:key')`, `<Trans>`, server `i18n.t`) → fail on referenced-
      but-missing; warn-list unused keys (fail only in `--strict`, used
      by CI on `main`).
   c. **Child furigana coverage**: for keys under any `*.child.*` path or
      namespaces `safety`, `chat.child`: ja values containing kanji above
      the grade-2 list (embed the jōyō grade-1/2 kanji allowlist in the
      script) must use `漢字|かんじ` ruby markup — fail listing
      offenders.
   d. Output machine-readable JSON + human table; exit codes distinct
      per failure class.
2. CI: add `pnpm i18n:check` to the 01 workflow (and keep in 51's
   pipeline); root script wiring.
3. Shared `Intl` helpers in `packages/i18n/src/format.ts`:
   `formatDate(d, locale, tz)`, `formatTime`, `relativeTime(d, locale,
   now)` ("さっき/3分前/昨日" vs "just now/3m ago/yesterday" — rule table
   documented + unit-tested per locale, DST-safe), `formatBytes(locale)`;
   refactor web call sites (21/22/24/31) onto them (mobile does so in
   41+).
4. Locale switcher polish (20's basic toggle → final): settings radio
   (25) + auth-screen footer switcher (pre-login, cookie-persisted);
   switching re-renders notification feed rows correctly (39's
   client-render decision makes this work — regression test).
5. `docs/i18n.md`: catalog structure, key naming, furigana markup spec,
   child-register style guide (both languages), how to run the checker,
   contribution workflow for translators.
6. Fix pass: run the checker across the repo at this point in history;
   fix every parity/missing/furigana violation in the same PR (expected
   drift from waves 0–4); record the before/after counts in the PR
   description.

## Acceptance Criteria

- [ ] Checker catches each failure class (self-test fixtures in
      `scripts/__tests__` with deliberately broken catalogs).
- [ ] Repo passes `pnpm i18n:check --strict` clean; CI gate active.
- [ ] Format helpers unit-tested (ja + en tables) and adopted by all
      existing web call sites (grep: no raw `toLocaleString` in
      apps/web).
- [ ] docs/i18n.md complete; furigana spec matches the 20 `<Furigana>`
      implementation.

## Validation

```bash
pnpm i18n:check --strict
pnpm --filter @famchat/i18n test
node scripts/check-i18n.mjs --self-test
```

## Dependencies

20 (i18n foundation + existing strings), 37/39 (notification catalogs to
check).

## Non-goals

New locales, machine translation, ICU migration, docs translation
management (55/56).

## Design References

- DESIGN §15 (i18n, furigana policy, completeness gate), §21 (CI);
  ADR-007
