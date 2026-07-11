# Issue 56: Closed-beta ToS/Privacy templates + consent flow

## Summary

Write the closed-beta legal document templates (Terms of Use, Privacy
Notice, Operator Data Access Policy) in plain-language Japanese and
English, and wire the in-product consent touchpoints (signup checkbox,
footer links, consent timestamp already captured).

## Context

DESIGN §22: templates live in-repo, disclose server-readability
(ADR-003) and guardian oversight honestly, and are **not legal advice —
the owner reviews before real families use the beta**. The consent
timestamp column and checkbox slots were prepared in 07/08/20.

## Scope

In scope: `docs/legal/` six documents, in-app links + checkbox copy,
`/legal` web routes rendering the docs, parity check inclusion, owner-
review gate marker.
Out of scope: public-launch legal program (COPPA/GDPR — v2 gate),
lawyer engagement (owner's call, flagged), cookie banners (no
third-party cookies/trackers exist — documented instead).

## Detailed Requirements

1. `docs/legal/terms.en.md` + `terms.ja.md` (closed-beta Terms of
   Use), plain language, covering: service description (family-only
   chat, closed beta, no fee); eligibility (guardians create the
   space; children use it under guardian management — guardian is
   responsible for child use, consistent with the product model);
   acceptable use (family use; no unlawful content; operator may
   suspend per policy); availability disclaimer (beta, no SLA, data
   loss best-effort against 50's backups); termination (owner may end
   beta with `[OWNER-CONFIRM: 30-day]` notice + export window);
   changes to terms (notice via the board/notification); governing law
   `[OWNER-CONFIRM: Japan]`. Operator-binding parameters — the
   termination notice period, governing law, and the operator-access
   disclosure window — are written as bracketed `[OWNER-CONFIRM: …]`
   placeholders exactly like this, and the legal README's owner-review
   checklist requires resolving every marker before beta invites go
   out.
2. `docs/legal/privacy.en.md` + `privacy.ja.md` (Privacy Notice),
   the honest core: what is stored (accounts, messages, images,
   logs — enumerated per DESIGN §7 in plain words); **server
   readability**: "the server can technically read message content;
   the operator's rules for that access are in the Operator Data
   Access Policy" (ADR-003 disclosure); guardian oversight: "family
   guardians can read children's rooms — the app tells children
   this"; what is NOT collected (no ads, no trackers, no analytics,
   no sale of data); push/email egress (browser push services, Expo,
   SMTP) with data minimization note; retention (deletion semantics +
   30-day purge from 36, backups 14-day cycle from 50); rights
   (export, deletion via owner/guardian per 36); operator identity +
   contact placeholder; legal basis framing per Japanese APPI
   vocabulary (利用目的の特定・安全管理措置) in the ja version.
3. `docs/legal/operator-access.en.md` + `operator-access.ja.md`
   (Operator Data Access Policy, DESIGN §13.7/§22): no content
   endpoints exist; emergency direct-DB access only for (enumerated:
   legal obligation, imminent-harm report, critical data-integrity
   repair), logged manually in the ops log and **disclosed to the
   affected space within `[OWNER-CONFIRM: 7 days]`**; admin actions are
   audited; verification pointer (OSS: the code is public).
4. Every document header: version, date, and a bold "TEMPLATE — not
   legal advice; the operator must review (and ideally consult a
   professional) before production use" banner; `docs/legal/README.md`
   explains the template status and the owner-review gate (a
   checklist the owner ticks before inviting non-family testers —
   referenced by 57's release checklist).
5. In-product wiring:
   - Web `/legal/terms` and `/legal/privacy` routes rendering the
     markdown with a **safe renderer configuration**: react-markdown
     (or equivalent) with raw HTML disabled (no rehype-raw), external
     links forced `rel="noopener noreferrer"` — a hostile-markdown
     fixture (script tag + raw-HTML img) must render inert (component
     test); locale-matched with fallback + language toggle;
     linked from: signup forms' consent checkbox labels (07/08 flows
     in 20's UI — checkbox is already required; this issue finalizes
     its copy: 「利用規約とプライバシーノートに同意します」/EN
     equivalent with links), auth footer, settings about section,
     mobile settings (47's screen links out to the web routes via
     browser — no native rendering in v1).
   - Consent versioning: `users.tosAcceptedAt` exists (07);
     re-consent on template change is v2 (documented in
     legal/README).
6. Parity: this issue registers its three en/ja pairs in 55's
   pair-registry config (`docs/.docs-pairs.json`) — the generic checker
   needs no code change.
7. Tests: web routes render both locales + the hostile-markdown fixture
   (component tests); signup E2E asserts checkbox required + timestamp
   stored (extends 53's onboarding spec); links resolve (docs:check).

## Acceptance Criteria

- [ ] Six documents complete, plain-language, honest per ADR-003 —
      readable by a non-lawyer guardian in ≤ 10 minutes each
      (word-count guidance: terms ≤ 1500 words, privacy ≤ 2000).
- [ ] Server-readability, guardian oversight, and operator-access
      rules disclosed exactly as designed (cross-check table in PR:
      claim → DESIGN section → doc paragraph).
- [ ] Consent flow wired and E2E-proven; timestamp verified.
- [ ] Owner-review gate documented; ja/en parity green.

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm docs:check
node scripts/check-docs-parity.mjs
pnpm --filter @famchat/web test -- -t legal
pnpm --filter @famchat/e2e test -- --grep onboarding
```

## Dependencies

07/20 (consent slots), 36 (retention/deletion facts), 53 (onboarding
E2E extended here), 55 (parity script + docs:check). Owner review is a
human gate before beta invites go out.

## Non-goals

Public-launch compliance (COPPA/GDPR/特商法 — v2), lawyer-reviewed
final text (owner's step), cookie consent UI (nothing to consent to),
DPA templates.

## Design References

- DESIGN §22 (compliance posture), §13.7 (operator policy); ADR-003,
  ADR-006
