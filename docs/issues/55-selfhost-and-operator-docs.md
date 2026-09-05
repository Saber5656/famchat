# Issue 55: Self-host + operator docs (ja/en)

## Summary

Write the operator-facing documentation set in English and Japanese: the
self-host installation guide verified against a fresh VM, the operator
quickstart (first family onboarding), and the consolidated ops runbook
index (deploy/upgrade/backup/restore/troubleshooting/FAQ).

## Context

ADR-005 (self-host = SaaS) makes these docs part of the product. DESIGN
§18.3 fixes the structure. Pieces were drafted by 35/49/50; this issue
unifies, verifies, and translates them.

## Scope

In scope: `docs/selfhost.md` + `docs/ja/selfhost.md`, operator
quickstart, `docs/ops/` index + gap-fill (upgrade.md, troubleshooting.md,
faq.md), doc lint, fresh-VM verification.
Out of scope: developer docs (CONTRIBUTING — 57), legal docs (56),
marketing/README rewrite (57).

## Detailed Requirements

1. `docs/selfhost.md` (English, canonical) sections: prerequisites
   (VPS ≥ 2 vCPU/4 GB, Docker + compose plugin, domain + DNS A record,
   open 80/443, SMTP relay account); install (clone/release-tag →
   `.env` from example → secret generation one-liners → compose up →
   first `/readyz` check); first operator steps (ops CLI setup, mint
   instance invite, hand to the first guardian, verify space creation);
   push setup (VAPID generation; Expo token optional with degradation
   note); upgrade (pointer to ops/deploy.md); backup enablement
   (pointer to ops/backup.md, cron lines); security baseline for the
   host (SSH keys-only, ufw allow 80/443, unattended-upgrades — the
   §19.2 "SSH hardening documented" row); **secret file hygiene** per
   §19.6 (`.env` chmod 600 in an owner-only directory; secrets excluded
   from backups by design and stored separately, e.g. a password
   manager); uninstall/data-removal.
2. `docs/ja/selfhost.md`: full Japanese translation (not a summary).
   Parity enforcement: do NOT reuse the 40 catalog checker (that is for
   UI strings); add `scripts/check-docs-parity.mjs` — a **generic**
   heading-structure comparator driven by a pair-registry config
   (`docs/.docs-pairs.json`); this issue registers the selfhost pair
   only, and issue 56 registers its three legal pairs in the same
   config (heading-count/anchor parity only; content drift is caught
   by human review).
3. Operator quickstart `docs/ops/quickstart.md`: the closed-beta
   operator's day-1: mint invite → onboard guardian → guardian
   assembles family → where to watch (stats, audit tail, backup logs).
   The family-facing quick guide is a section inside quickstart.md
   with screenshots committed to `docs/assets/quickstart/*.png`,
   captured per 52's screenshot recipe (paths referenced relatively so
   they render on GitHub).
4. Consolidate `docs/ops/`: index README; ensure `deploy.md` (49),
   `backup.md`/`restore.md` (50), `admin.md` (35),
   `webpush-checklist.md` (38), `heic.md` (18) exist and cross-link; write the missing
   `upgrade.md` (tag bump walkthrough + migration caveat) and
   `troubleshooting.md` (cert issuance, 502s, WS drops behind
   proxies, disk full, queue stuck — each: symptom/diagnosis/fix) and
   `faq.md` (can I use managed Postgres/S3? — yes, env URLs; multiple
   families? — yes, invites; E2EE? — no, link ADR-003 explanation;
   resource sizing; iOS push question from research).
5. Fresh-VM verification: execute `docs/selfhost.md` verbatim on a
   clean VM (or local multipass/UTM) recording a timed transcript;
   every deviation found = doc fix in this PR; the transcript is the
   acceptance evidence (same procedure as 49's smoke but following
   only the doc, no scripts knowledge).
6. Doc lint: markdown link checker script over `docs/` (internal links
   resolve; external links HEAD-checked warn-only) wired as
   `pnpm docs:check` + CI job (51's workflow gains the step).

## Acceptance Criteria

- [ ] A technical stranger can self-host following selfhost.md alone —
      evidenced by the verbatim fresh-VM transcript (≤ 60 min).
- [ ] ja/en pairs structurally in parity (checker green) and fully
      translated.
- [ ] ops/ index complete with all runbooks cross-linked; no dead
      internal links.
- [ ] FAQ answers the E2EE and managed-services questions with ADR
      links.

## Validation

```bash
pnpm docs:check
node scripts/check-docs-parity.mjs
# evidence: fresh-VM transcript attached to PR
```

## Dependencies

35, 49, 50 (content sources), 51 (CI wiring for docs:check), 52
(screenshots).

## Non-goals

README/marketing (57), CONTRIBUTING (57), legal docs (56), versioned
docs site (v2).

## Design References

- DESIGN §18.3 (selfhost), §18.1–18.2 (ops), §19.2 (host hardening
  row); ADR-005
