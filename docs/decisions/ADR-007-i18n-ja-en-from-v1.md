# ADR-007: Japanese + English i18n from v1, i18next everywhere

- Status: Accepted (owner-confirmed 2026-07-11)
- Deciders: Owner (Saber5656), Fable (design agent)

## Context

The owner explicitly chose shipping both Japanese and English UI in v1
(rejecting the ja-only recommendation) to position famchat as an
international OSS project from the start. Children aged 6–12 are the primary
users, which imposes a reading-level constraint on Japanese strings.

## Decision

- Locales `ja` and `en` are both first-class in v1; CI fails on catalog
  divergence (missing/extra keys) via `scripts/check-i18n.mjs`.
- One library — i18next — on web (react-i18next), mobile (react-i18next),
  and server (notification/email/system-message composition), with shared
  JSON catalogs in `packages/i18n`.
- Child-register policy: child-facing ja strings use kana-friendly wording
  with ruby (`漢字|かんじ` markup rendered by a shared `<Furigana>`
  component); child-facing en strings use simple vocabulary.
- Locale resolution: user profile setting first, then platform signals;
  notifications always use the recipient's stored locale.

## Alternatives considered

| Alternative | Why rejected |
|---|---|
| ja-only v1 (original recommendation) | Owner decision: international OSS posture is worth the cost |
| next-intl (web) + i18next (mobile) | Two libraries, two syntaxes, double the ways for agents to err |
| Message extraction via ICU/FormatJS | i18next's JSON catalogs are simpler for bilingual maintenance |

## Consequences

- Every UI issue carries "strings exist in ja and en, checker passes" in its
  acceptance criteria — translation is never a follow-up task.
- String freeze discipline: PRs adding keys must add both locales; the CI
  gate makes this mechanical.
- Furigana markup adds a mini-DSL that the moderation pipeline must ignore
  (it applies to user content, not catalog strings — no interaction).
