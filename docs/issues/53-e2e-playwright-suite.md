# Issue 53: Web E2E suite (golden paths)

## Summary

Build the Playwright suite covering the product's golden paths against a
production-shaped compose stack seeded with the 52 fixtures, wired into
CI as a nightly + label-triggered job.

## Context

DESIGN §21 and ISSUE_PLAN §6.4 list the golden paths. Individual web
issues shipped scoped Playwright specs (@chat, @guardian, …); this issue
consolidates them into one suite with a real-stack environment,
cross-user orchestration, and CI wiring.

## Scope

In scope: e2e environment (compose overlay), fixtures/auth helpers,
golden-path specs, flake policy, CI job, artifacts.
Out of scope: mobile E2E (v2 Maestro), load testing, visual regression
(v2).

## Detailed Requirements

1. Environment: `infra/compose.e2e.yml` overlay on compose.prod.yml —
   locally-built images (`build:`), `FAMCHAT_DOMAIN=localhost` internal
   TLS (49's variant), mailpit added, seeded via
   `pnpm db:seed --e2e --force-reset` after migrate; helper
   `scripts/e2e-up.sh` (up → wait readyz → seed) / `e2e-down.sh`.
2. Playwright config (`apps/web/playwright.config.ts` extended or root
   `e2e/` package — choose root `e2e/` package `@famchat/e2e` so specs
   can exercise multi-app flows without web-package coupling): chromium
   primary + webkit smoke project (chat + link paths only — WebKit
   proxies iOS-Safari risk);
   auth helpers split by kind: adults (mom/dad/grandma/uncle) log in
   via API with the 52 credentials → storageState built once per run;
   **children (haruto/hinata) have no credentials** — the child helper
   drives the real flow: guardian-context API call
   `children.createLinkCode` → `auth.childLink` (or the `/link` UI in
   the specs that test it) → capture the device-session storageState.
   Two-context orchestration helper. Seed ids come from
   `loadSeedOutput()` (52), never hardcoded.
3. Golden-path specs (each independent, seeded state + API arrangement,
   asserting UI **and** relevant API/DB effects):
   1. `onboarding`: operator invite (via admin API) → new guardian
      signup + space create → invite grandma → she joins.
   2. `child-link`: guardian generates code (32 UI) → child context
      links via `/link` → lands in family room with oversight banner.
   3. `chat`: two-context text loop, receipts, delete tombstone,
      reconnect recovery (CDP-forced socket drop).
   4. `image`: upload → processing → both contexts render; GPS-EXIF
      fixture verified sanitized (S3 read in test).
   5. `moderation`: child sends NG term → flag badge (guardian ctx) +
      dashboard queue → remove → tombstone for all.
   6. `report`: child reports → guardian queue shows reason → resolve;
      reported member context sees nothing.
   7. `quiet-hours`: guardian sets window covering now → child locks
      live → report exemption NOT offered on lock (34) → window ends
      (short window) → unlock.
   8. `board`: post with images → pin (guardian) → comment (child) →
      order + live updates in second context.
   9. `notifications`: message → bell increments (recipient ctx) → feed
      → deep link lands in room; web push subscribe + receive
      (chromium CDP push, from 38).
   10. `lifecycle`: owner requests deletion → read-only + countdown →
       cancel; export request → download zip → parse + assert counts
       (36). (Added beyond the DESIGN §21 base list as release
       coverage; ISSUE_PLAN §6.4 includes it.)
4. Flake policy: `retries: 1` in CI; specs must pass 5× locally
   (`--repeat-each 5`) before merge (documented in CONTRIBUTING via
   57); a `@quarantine` tag excludes from required runs with an issue
   link required.
5. CI (`e2e.yml`): nightly on main + PRs labeled `e2e`; builds images,
   runs `e2e-up.sh`, executes suite (chromium job + webkit smoke job),
   uploads traces/videos on failure with **artifact hygiene (DESIGN
   §19.6)**: storageState/auth files are written outside the artifact
   dirs and never uploaded; Playwright configured with
   `recordHar: false` and trace network capture limited to
   status/timing (no bodies/headers); CI env masks all secrets;
   artifact retention 7 days. The e2e stack uses seed-only dummy data,
   which keeps residual exposure low — but session material must still
   never land in artifacts (a CI grep step asserts no
   `famchat_session` value appears in uploaded files). ~25-min budget
   documented.
6. Consolidation: migrate the per-issue scoped specs (@chat @guardian
   @quiet …) into `e2e/` where duplicated, keeping fast component-level
   Playwright in apps/web only where it tests web-only concerns
   (documented split rule: cross-user/full-stack ⇒ e2e package).

## Acceptance Criteria

- [ ] All 10 golden paths green against the prod-shaped stack locally
      and in the nightly job (link first green run).
- [ ] Reconnect, sanitization, and export-content assertions are real
      (not UI-only) — reviewed against the spec list above.
- [ ] Suite is deterministic: 5× repeat locally clean (transcript).
- [ ] Failure artifacts (trace/video) upload verified on a forced
      failure.

## Validation

```bash
bash scripts/e2e-up.sh && pnpm --filter @famchat/e2e test
pnpm --filter @famchat/e2e test -- --repeat-each 5   # determinism proof
```

## Dependencies

23, 24, 32, 33, 34, 36, 38, 39 (features under test), 49 (stack), 51
(CI base), 52 (seed + loadSeedOutput).

## Non-goals

Mobile E2E (v2), performance/load tests, visual regression, cross-
browser matrix beyond chromium+webkit-smoke.

## Design References

- DESIGN §21 (strategy), ISSUE_PLAN §6 (validation strategy, golden
  paths)
