# Issue 51: Full CI/CD pipeline

## Summary

Grow the 01 baseline into the complete pipeline: integration tests
against real services, web/mobile typecheck+unit, i18n gate, secret and
dependency scanning, image build+publish to GHCR on tags, Renovate, and
the branch-protection documentation.

## Context

DESIGN §21 (gates table) and §19.3 (supply-chain row). Publishing images
is what makes 49's compose consumable; scanning gates are prerequisites
for the 54 security gate and 57 release prep.

## Scope

In scope: `.github/workflows/ci.yml` expansion, `release.yml` (tag →
GHCR), gitleaks + osv/pnpm-audit jobs, Renovate config, caching,
required-checks docs, README badge.
Out of scope: E2E job wiring (53 adds its job to this pipeline), EAS
builds (48's manual-dispatch workflow), deploy automation (manual per
49 runbook — v2).

## Detailed Requirements

1. `ci.yml` (PR + push main) — jobs with pnpm store caching:
   - `lint-type`: eslint, prettier check, `pnpm -r typecheck`.
   - `unit`: `pnpm -r --if-present test:unit` — convention set by this
     issue: every workspace defines `test:unit` (vitest project
     excluding `test/integration/**`) and `test:integration`
     (only that folder); root scripts call these, so no runner-flag
     assumptions leak across workspaces.
   - `integration`: services via GitHub Actions service containers
     postgres:16 (`POSTGRES_USER/PASSWORD/DB=famchat`, health cmd
     `pg_isready`) and redis:7 (health `redis-cli ping`); MinIO via a
     `docker run -d` step (pinned tag, `MINIO_ROOT_USER/PASSWORD` from
     job env) + `mc mb` bucket-creation step + curl health wait; job
     env block sets exactly `DATABASE_URL`, `REDIS_URL`, `S3_ENDPOINT`,
     `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`,
     `S3_SECRET_ACCESS_KEY`, `S3_FORCE_PATH_STYLE=true`,
     `SESSION_SECRET`, `SMTP_URL` (mailpit container), `APP_BASE_URL`,
     `WEB_ORIGINS` (CI-only dummy secrets); runs `pnpm -r --if-present
     test:integration` — which **includes the `@tenant` cross-tenant
     denial suites (ISSUE_PLAN §6.8 evidence lives in this job)**;
     artifacts on failure (logs).
   - `i18n`: `pnpm i18n:check --strict` (40).
   - `gitleaks`: full-repo scan (action, pinned by SHA), config
     `.gitleaks.toml` with documented allowlist (test fixtures).
   - `deps-audit`: `pnpm audit --prod --audit-level high` +
     osv-scanner (pinned action) — **fail on high/critical in prod
     deps; report-only for dev deps** (triage policy documented in the
     workflow comments; overrides via `pnpm.overrides` documented).
   - `web-build`: `pnpm --filter @famchat/web build` (Next production
     build compiles).
   - All third-party actions pinned to commit SHAs (supply-chain rule).
2. `release.yml` (on tag `v*`): builds and pushes
   `ghcr.io/saber5656/famchat-{api,worker,web}` multi-stage images
   (linux/amd64; arm64 optional matrix documented but off) using
   `docker/metadata-action` semver tagging: every tag publishes its
   exact version; **`latest` moves only for stable releases**
   (prerelease tags like `v0.0.1-rc0` never touch `latest`). OCI
   labels (source, revision, licenses); job permissions `packages:
   write` **and `id-token: write`** (required by provenance
   attestation in `docker/build-push-action`) — images become the 49
   pins.
3. Concurrency: cancel-in-progress per ref for ci.yml; release never
   cancelled.
4. `renovate.json`: weekly grouped minor/patch, daily security updates,
   lockfile maintenance, pin GitHub Actions digests, labels; pnpm
   workspaces preset.
5. Branch protection: `docs/ops/repo-settings.md` documenting required
   checks (lint-type, unit, integration, i18n, gitleaks, deps-audit,
   web-build),
   linear history, no force push (matches the user's ruleset), and that
   the owner applies these in GitHub settings (agent documents, owner
   clicks).
6. README: CI badge + a "Releases" section explaining tag-driven
   images.
7. Validation of the pipeline itself: a scratch PR exercising a
   deliberate failure per gate (type error, missing i18n key, planted
   fake secret in a fixture) — evidence links in this issue's PR
   description; the release flow validated by pushing `v0.0.1-rc0` tag
   on the working branch (images appear in GHCR; delete after).

## Acceptance Criteria

- [ ] All jobs green on main; each gate demonstrably fails on its
      counterexample (evidence links).
- [ ] `v*` tag produces pullable GHCR images that boot in 49's compose
      (smoke re-run with published images).
- [ ] Third-party actions SHA-pinned; Renovate active with first PRs.
- [ ] Required-checks doc matches the actual job names.

## Validation

```bash
gh run list --workflow ci.yml -L 5
# per-gate counterexample evidence: one scratch PR per gate, then
gh pr checks <scratch-pr> --json name,bucket
# release flow: git tag v0.0.1-rc0 && git push origin v0.0.1-rc0
gh run watch --workflow release.yml
docker pull ghcr.io/saber5656/famchat-api:v0.0.1-rc0
docker manifest inspect ghcr.io/saber5656/famchat-api:latest || echo "latest untouched by rc (expected)"
bash scripts/test-backup-cycle.sh   # 50's cycle wired as the scheduled job added here
```

## Dependencies

49 (Dockerfiles/compose consumers), 40 (i18n gate), 01.

## Non-goals

E2E job (53), auto-deploy, EAS in CI (48 manual dispatch), SBOM/signing
beyond provenance (v2), coverage thresholds (v2).

## Design References

- DESIGN §21 (gates), §19.3 (supply chain), §18.1 (image pins);
  ADR-005
