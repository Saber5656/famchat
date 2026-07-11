# Issue 57: OSS release prep (license, community files, scans, public-flip checklist)

## Summary

Prepare the repository for public release: LICENSE per the owner's
ADR-009 decision, the README rewrite, CONTRIBUTING / CODE_OF_CONDUCT /
SECURITY.md, issue/PR templates, license compliance checking, the
full-history secret scan, and the final public-flip checklist the owner
executes.

## Context

**The repository is already public** (visibility PUBLIC since creation),
so there is no private→public flip; what this issue gates is the
**v1.0 release announcement**: history scanned, licensing decided
(ADR-009 — **blocked on owner**; until then the public repo carries no
license, i.e. all-rights-reserved), and trust documents in place. The
"flip checklist" below is therefore a release checklist; the only
owner-visibility action left is confirming repo settings (issues/
actions/packages exposure) rather than changing visibility.

## Scope

In scope: all files above, README bilingual rewrite, license checker in
CI, gitleaks + trufflehog full-history scans with triage, repo metadata,
the flip checklist.
Out of scope: the visibility flip itself (owner), marketing site,
governance beyond a solo-maintainer note, store publication (v2).

## Detailed Requirements

1. **License (human gate)**: obtain the owner's ADR-009 decision —
   which covers BOTH the license choice AND the header policy (the
   proposed default, repo-level notice without per-file headers, is
   part of what the owner accepts or amends in ADR-009; this issue
   implements whatever the accepted ADR text says); update ADR-009
   status to Accepted; add `LICENSE` (exact SPDX text), `license`
   fields in every package.json, and a README licensing section. Add
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
8. Repo metadata prep — exact split: the agent runs
   `gh repo edit Saber5656/famchat --description "…" --add-topic
   family --add-topic chat --add-topic child-safety --add-topic
   self-hosted --add-topic typescript` (reversible metadata,
   agent-executable); everything else (social preview image, branch
   protection clicks per 51's doc, the visibility flip) is owner-only
   and listed in the flip checklist.
9. `docs/release/v1-release-checklist.md` — the owner-executed gate:
   ADR-009 accepted + LICENSE merged; history scan clean/triaged;
   54 security gate signed off; 56 legal templates owner-reviewed;
   55 selfhost verified; CI green on main; screenshots current;
   `git log` author-email check (no private emails unintended);
   repo settings confirmed (issues/actions/packages exposure) →
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
gitleaks detect --source . --log-opts="--all" && echo gitleaks-clean
docker run --rm -v "$PWD:/repo" ghcr.io/trufflesecurity/trufflehog:v3 \
  git file:///repo --only-verified && echo trufflehog-clean
# both results + triage recorded in docs/security/history-scan.md
pnpm docs:check
```

## Dependencies

51 (CI), 54 (security gate), 55 (docs), 56 (legal owner review — gates
the flip checklist), **ADR-009 owner decision (human gate)**.

## Non-goals

The visibility flip (owner), governance docs beyond solo-maintainer,
store publication, announcement/marketing content beyond the README.

## Design References

- DESIGN §19.6–19.7 (secrets, disclosure), §22; ADR-009; user's
  global repo-publication rule (history scan before public)
