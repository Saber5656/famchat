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
(v2 gate), SECURITY.md (entirely issue 57's — this gate neither
creates nor verifies it).

## Detailed Requirements

1. Control inventory + checklist: first author
   `docs/security/controls.json` — a manually curated array
   `[{ id: "S19.2-01", section: "19.2", control: "...", … }]`
   transcribing every control row of DESIGN §19.2 (boundary controls),
   §19.3 (abuse cases), §19.4 (headers/cookies/CSP), §19.5
   (limits/quotas), §19.6 (secrets) with stable ids (the human
   transcription IS the counting rule — no prose parsing); then
   `docs/security/checklist.md` has exactly one row per inventory id —
   columns: control, verification method (test id / script / manual
   step), evidence link, status, owner-visible risk note for accepted
   residuals. `scripts/check-security-coverage.mjs` cross-counts
   inventory ids vs checklist rows (and vs the tenant-coverage walker
   below). Every row must land on pass / fixed /
   accepted-with-rationale.
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
4. Findings workflow — severity rubric (mechanical): **High** =
   authz/tenant-isolation bypass, content or secret exposure, auth
   weakness (fix in this issue, gate-blocking); **Medium** = hardening
   gap with compensating controls (fix here if the fix touches ≤ 2
   files, else file a follow-up issue); **Low** = defense-in-depth nit
   (file or accept with rationale). Every finding is recorded in the
   checklist row with fields `{ id, control, finding, severity,
   evidence, resolution: fixed@<commit> | issue docs/issues/NN-*.md |
   accepted:<rationale> }`; follow-up issues use the next free NN and
   are added to ISSUE_PLAN §2 + §8. Zero open High to pass the gate.
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
bash scripts/check-headers.sh --base https://localhost --insecure  # internal-TLS stack (script flag maps to curl -k)
pnpm --filter @famchat/api test -- -t @tenant
pnpm --filter @famchat/api test -- -t @upload-abuse
pnpm --filter @famchat/api test -- -t @auth-pack
pnpm --filter @famchat/api test -- -t @csrf-cors
node scripts/check-security-coverage.mjs
gitleaks detect --source . && pnpm audit --prod --audit-level high
# manual evidence (logs review, push payloads, VPS port scan) recorded per checklist row
```

## Dependencies

49 (stack), 51 (scan jobs), 53 (suite exercising the system), 35, 38.

## Non-goals

External pentest, formal threat-model re-write (design §19 is it),
compliance audits, SECURITY.md authoring (57).

## Design References

- DESIGN §19 (entire section), §13.7 (operator boundary), §21
  (gates); ISSUE_PLAN §6.5
