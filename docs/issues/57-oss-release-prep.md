# Issue 57: OSS release prep (license, community files, scans, public-flip checklist)

## Summary

Prepare the repository for public release: LICENSE per the owner's
ADR-009 decision, the README rewrite, CONTRIBUTING / CODE_OF_CONDUCT /
SECURITY.md, issue/PR templates, license compliance checking, the
full-history secret scan, and the final public-flip checklist the owner
executes.

## Context

The repo was designed public-first (task premise), but publication is a
one-way door: history is scanned, licensing decided (ADR-009 — **blocked
on owner**), and trust documents in place before the flip. The owner
performs the actual visibility change.

## Scope

In scope: all files above, README bilingual rewrite, license checker in
CI, gitleaks + trufflehog full-history scans with triage, repo metadata,
the flip checklist.
Out of scope: the visibility flip itself (owner), marketing site,
governance beyond a solo-maintainer note, store publication (v2).

## Detailed Requirements

1. **License (human gate)**: obtain the owner's ADR-009 decision;
   update ADR-009 status to Accepted with the choice; add `LICENSE`
   (exact SPDX text), `license` fields in every package.json, and a
   README licensing section. If AGPL-3.0 (proposed): add the
   recommended per-file header policy decision (repo-level notice
   only, no per-file headers — simpler; document). Add
   `license-checker-rseidelsohn` (or `pnpm licenses list` script) CI
   step failing on copyleft-incompatible or unknown prod dependency
   licenses relative to the chosen license (policy table in the
   script).
2. README rewrite (bilingual: English primary, 日本語 section
   mirroring the essentials): what famchat is (child-safety-first
   family chat, one paragraph), feature list with screenshots (52's
   demo recipe), safety philosophy summary (closed network / oversight
   with transparency / no E2EE and why — link ADR-003 honestly),
   self-host quickstart (5 lines + link to 55), beta status +
   expectations, tech stack overview, contributing/security/license
   links, badges (CI, license).
3. `CONTRIBUTING.md`: dev setup (01/02 recap), monorepo map (DESIGN
   §4), quality gates (lint/type/test/i18n/e2e — incl. the 53
   repeat-each rule), how issues are structured (docs/ISSUE_PLAN.md is
   canonical — derived-artifact rule from this repo's process),
   Conventional Commits, PR expectations, "safety-affecting changes
   need a DESIGN.md update first" rule.
4. `CODE_OF_CONDUCT.md`: Contributor Covenant 2.1, owner contact.
5. `SECURITY.md` per DESIGN §19.7: private reporting via GitHub
   Security Advisories, 90-day coordinated disclosure, supported
   versions (latest minor), scope notes (self-hosted instances are
   operators' responsibility; the beta instance is in scope), no
   bounty.
6. `.github/ISSUE_TEMPLATE/` (bug, feature, security-redirect) +
   `PULL_REQUEST_TEMPLATE.md` (checklist: tests, i18n parity, docs,
   safety impact).
7. History scan: gitleaks full-history + trufflehog (pinned versions)
   → triage doc `docs/security/history-scan.md` (findings, false-
   positive rationale); **any real secret ever committed ⇒ rotate +
   consider fresh-repo migration per the user's global rule** —
   escalate to owner, do not flip.
8. Repo metadata prep (documented for owner or via gh CLI where
   authorized): description, topics (family, chat, child-safety,
   self-hosted, typescript), social preview note; ensure default
   branch protection doc (51) applied.
9. `docs/release/public-flip-checklist.md` — the owner-executed gate:
   ADR-009 accepted + LICENSE merged; history scan clean/triaged;
   54 security gate signed off; 56 legal templates owner-reviewed;
   55 selfhost verified; CI green on main; screenshots current;
   `git log` author-email check (no private emails unintended);
   flip visibility → verify issues/actions/packages visibility →
   tag v1.0.0 → announce note.

## Acceptance Criteria

- [ ] LICENSE + headers policy in place matching the accepted ADR-009;
      license CI check green.
- [ ] README/CONTRIBUTING/CoC/SECURITY/templates complete; README
      bilingual and honest about E2EE posture.
- [ ] History scan executed with written triage; zero unrotated real
      secrets (evidence linked).
- [ ] Flip checklist complete up to the owner-action line; everything
      before it green.

## Validation

```bash
node scripts/check-licenses.mjs
gitleaks detect --source . --log-opts="--all" && echo clean
pnpm docs:check
```

## Dependencies

54 (security gate), 55 (docs), 56 (legal), 51 (CI), **ADR-009 owner
decision (human gate)**.

## Non-goals

The visibility flip (owner), governance docs beyond solo-maintainer,
store publication, announcement/marketing content beyond the README.

## Design References

- DESIGN §19.6–19.7 (secrets, disclosure), §22; ADR-009; user's
  global repo-publication rule (history scan before public)
