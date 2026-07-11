# Issue 54: Security hardening gate (DESIGN §19 checklist)

## Summary

Convert DESIGN §19 into an evidence-backed pass/fail checklist, execute
it against the production-shaped stack, fix everything fixable, and
obtain owner sign-off — the release-blocking security gate.

## Context

DESIGN §19 defines trust boundaries, abuse cases, headers, limits, and
secret rules; ISSUE_PLAN §6.5 makes this a release gate. Prior issues
implemented controls piecemeal with tests; this issue verifies the
assembled system and hunts the gaps between issues.

## Scope

In scope: checklist doc, automated verification scripts, manual
verification passes, fixes for findings (in-scope severity: all high/
critical, medium by judgment), sign-off record.
Out of scope: external pentest (v2), bug bounty, COPPA/GDPR assessment
(v2 gate), SECURITY.md text (57 — verified here as present-by-then or
stubbed).

## Detailed Requirements

1. `docs/security/checklist.md`: one row per control from DESIGN
   §19.2 (boundary controls), §19.3 (abuse cases), §19.4 (headers/
   cookies/CSP), §19.5 (limits/quotas), §19.6 (secrets) — columns:
   control, verification method (test id / script / manual step),
   evidence link, status, owner-visible risk note for accepted
   residuals. Every row must land on pass / fixed / accepted-with-
   rationale.
2. Automated verifiers (added to repo, runnable anytime):
   - `scripts/check-headers.sh`: curl assertions against the e2e stack
     — CSP exact-match on web routes, HSTS, nosniff, Referrer-Policy,
     Cache-Control: no-store on API, cookie attributes on login
     response, absence of `Server`/`X-Powered-By`.
   - Cross-tenant sweep: tag `@tenant` integration tests (10's helper
     applied to **every** space-scoped procedure — this issue audits
     coverage: script walks `PROCEDURE_REGISTRY` and fails if any
     space-scoped procedure lacks a registered tenant-denial test;
     write the missing ones).
   - Upload abuse pack: re-run 18's hostile fixtures through the full
     HTTP path (not worker-direct): fake-mime, polyglot
     (GIFAR-style), oversize, dimension bomb, GPS fixture → served
     variant sanitized.
   - Auth pack: session fixation (pre-login cookie ≠ post-login),
     rotation, logout-everywhere, reset-token single-use, brute-force
     lockouts (login, invite preview, child link, PIN) with timing
     sanity (constant-time claims spot-checked statistically, 1000
     samples, documented as smoke-level only).
   - CSRF/CORS pack: cookie-bearing mutation without header → 403;
     disallowed origin preflight → blocked; admin endpoints reject
     browser-origin requests.
   - `gitleaks` full-history + dependency audit re-run (51 jobs) with
     triage notes.
3. Manual passes (documented with evidence in the checklist):
   - Port exposure scan of the beta VPS (only 22/80/443; SSH
     hardening per selfhost doc).
   - Guardian-oversight boundary walk: adult-only room invisible to
     non-member guardian in UI + API + WS (existence-oracle check).
   - Operator content-free walk: every admin endpoint response
     reviewed against the 35 deny-list test + manual spot check.
   - Log review on the e2e stack after the full 53 suite: grep for
     tokens/passwords/message bodies in api+worker logs (must be
     absent; add redaction fixes if found).
   - Push payload review: locked-child suppression, flagged-content
     notifications term-free.
4. Findings workflow: each finding → fix in this issue if ≤ 1 day,
   else a new numbered issue (ISSUE_PLAN §8 known-unknown process) —
   zero open high/critical to pass the gate; the checklist records
   the decision trail.
5. Sign-off: PR description includes the completed checklist summary
   table; owner review/approval of the PR constitutes the gate record
   (referenced by 57's release checklist).

## Acceptance Criteria

- [ ] Checklist covers 100% of DESIGN §19 rows (script cross-counts
      section items vs checklist rows).
- [ ] All automated verifiers pass on the e2e stack; committed and
      re-runnable.
- [ ] Tenant-denial coverage: every space-scoped procedure has a
      registered denial test (meta-script green).
- [ ] Zero open high/critical findings; residual risks documented
      with owner sign-off.

## Validation

```bash
bash scripts/check-headers.sh https://localhost
pnpm --filter @famchat/api test -- --grep @tenant
node scripts/check-security-coverage.mjs
```

## Dependencies

49 (stack), 51 (scan jobs), 53 (suite exercising the system), 35, 38.

## Non-goals

External pentest, formal threat-model re-write (design §19 is it),
compliance audits, SECURITY.md authoring (57).

## Design References

- DESIGN §19 (entire section), §13.7 (operator boundary), §21
  (gates); ISSUE_PLAN §6.5
