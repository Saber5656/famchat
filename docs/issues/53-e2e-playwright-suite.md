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
   proxies iOS-Safari risk); `SEED_IDS` imported from `@famchat/db`;
   auth helpers: session-injection via API login → storageState per
   persona (mom/dad/grandma/uncle/haruto/hinata) built once per run;
   two-context orchestration helper.
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
       (36).
4. Flake policy: `retries: 1` in CI; specs must pass 5× locally
   (`--repeat-each 5`) before merge (documented in CONTRIBUTING via
   57); a `@quarantine` tag excludes from required runs with an issue
   link required.
5. CI (`e2e.yml`): nightly on main + PRs labeled `e2e`; builds images,
   runs `e2e-up.sh`, executes suite (chromium job + webkit smoke job),
   uploads traces/videos on failure; ~25-min budget documented.
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

52 (seed/SEED_IDS), 49 (stack), 34/33/32/24/23 (features under test),
36, 38, 51 (CI base).

## Non-goals

Mobile E2E (v2), performance/load tests, visual regression, cross-
browser matrix beyond chromium+webkit-smoke.

## Design References

- DESIGN §21 (strategy), ISSUE_PLAN §6 (validation strategy, golden
  paths)
