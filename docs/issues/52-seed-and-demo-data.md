# Issue 52: Seed + demo data

## Summary

Build the deterministic demo-family seed used by local development,
Playwright E2E (53), and screenshots: a realistic bilingual family with
rooms, messages, images, board posts, safety states (flags, reports,
quiet hours), all with stable IDs.

## Context

DESIGN §21 (test strategy) — E2E golden paths need known state; 04
shipped only a skeleton. Deterministic ULIDs make Playwright selectors
and API assertions stable.

## Scope

In scope: `packages/db/src/seed.ts` full implementation, fixture images,
reset flow, `--demo` and `--e2e` modes, docs.
Out of scope: production seeding (never — guard against it), load/perf
data generation (v2).

## Detailed Requirements

1. Guard (exact rules): `NODE_ENV=production` ⇒ hard refusal, no
   override flag exists. Otherwise: if the DB contains ANY row in any
   app table, the seed aborts unless `--force-reset` is passed;
   `--force-reset` truncates all app tables first. There is no
   "detect real data" heuristic — non-empty means abort-or-reset,
   period.
2. Identity strategy: guardian/adult accounts and the space are
   created with **deterministic credentials** (table below) via the
   service layer; because services generate their own ULIDs, the seed
   writes every created id into `packages/db/seed-output.json`
   (`{ spaceId, users: {mom,dad,grandma,uncle,haruto,hinata}, rooms:
   {family,siblings,…}, flaggedMessageId, openReportId, inviteCode,
   instanceInviteCode, … }`) — exported helper `loadSeedOutput()` is
   what 53's E2E and dev tools consume (no compile-time id constants).
   Deterministic credentials: mom `mom@example.famchat` / dad
   `dad@example.famchat` / grandma `grandma@example.famchat` / uncle
   `uncle@example.famchat`, all password `famchat-demo-1234!`
   (dev-only, printed with a warning); ひなた's PIN `1234`. Child
   sessions are NOT pre-seeded — E2E creates a link code via the
   guardian API and redeems it (the real flow).
3. Demo family (mode `--demo`, ja-default with en variants where
   noted):
   - Space 佐藤ファミリー (Asia/Tokyo, mode flag, both builtin lists
     on).
   - Members: mom (owner guardian, ja), dad (guardian, ja), grandma
     (adult, ja), uncle (adult, **en** locale — bilingual demo), kids
     はると (child, 9) and ひなた (child, 7, PIN set, quiet hours
     21:00–07:00 weekdays).
   - Rooms: family (auto), group 「きょうだい」 (kids+mom), direct
     grandma↔はると, adult-only direct mom↔dad (oversight-exclusion
     demo).
   - ~120 messages across 10 days (deterministic timestamps): everyday
     ja family chat with en sprinkles from uncle; includes: 3 image
     messages (fixtures), 1 soft-deleted, 1 flagged (contains a
     builtin ja list term — pick a mild bullying-category term), system
     join messages, read states staggered per member.
   - Board: 4 posts (1 pinned 「うんどうかいの おしらせ」 with 2
     images, 1 en post from uncle) + comments.
   - Safety: the flagged message's moderation_hit (pending), 1 open
     report (ひなた reported a message, reason `unkind`), 1 resolved
     report; audit trail consistent with all the above. **Service-layer
     rule**: every safety-state row (flag, report, quiet hours,
     memberships, invites) is created through the real services so
     invariants/audit stay true; the ONLY sanctioned raw-insert path is
     the bulk chat-history messages (for timestamp control), documented
     at its single call site.
   - Notifications: derived naturally from the service-layer calls;
     Web push subscriptions: none (device-specific).
   - One instance invite (unused) + one space invite (active) for
     invite-flow E2E.
4. Fixture images `packages/db/fixtures/`: 4 small (< 200 KB)
   license-free photos (generate simple SVG-rendered scenes → PNG in
   repo, documented provenance — no external downloads) + 1 jpeg with
   injected GPS EXIF (created by a script using piexifjs in devDeps;
   the fixture and its generator both committed) reused by 18/23
   tests.
5. `--e2e` mode: demo minus bulky history (20 messages) for fast CI
   setup; same `seed-output.json` contract.
6. `pnpm db:seed --demo|--e2e [--force-reset]` root wiring; fixture
   images run through the real 17/18 pipeline (the seed asserts
   S3/Redis reachability first). If media services are unreachable the
   seed **skips image content entirely** (affected messages/posts
   become text-only, one warning per skip) — no fake-`ready` rows and
   no dangling media references, ever.
7. Docs: `docs/dev/seed.md` — personas + credentials table,
   `loadSeedOutput()` reference, modes, reset warnings, screenshot
   recipe (`pnpm db:seed --demo` + URLs to visit).
8. Tests: seed runs idempotently twice with `--force-reset`; guard
   blocks production and non-empty-without-flag runs;
   `seed-output.json` ids all resolve to live rows (via
   `loadSeedOutput()`); the flagged message actually trips 27's
   pipeline when seeded via services (assert hit row exists);
   media-services-down path seeds clean text-only data.

## Acceptance Criteria

- [ ] One command yields the full demo family with all safety states
      visible in web UI (manual walkthrough + screenshots in PR).
- [ ] `--e2e` mode + `loadSeedOutput()` consumed by a seed-level
      integration test in this PR (53 owns the E2E suite itself).
- [ ] Production guard proven by test.
- [ ] Fixture provenance documented; GPS fixture generated in-repo.

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm db:seed --demo --force-reset && pnpm db:seed --e2e --force-reset
pnpm --filter @famchat/db test -- -t seed
```

## Dependencies

04, 18 (image path), 19, 27, 28, 29 (services the seed drives).

## Non-goals

Load testing data, per-locale full demo duplication (single bilingual
family instead), production data migration tooling.

## Design References

- DESIGN §21 (testing), §15 (bilingual demo), §13 (safety states to
  exhibit)
