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

1. Guard: seed refuses to run when `NODE_ENV=production` or the DB has
   any real space (row count > seed-known ids) unless `--force-reset`;
   `--force-reset` truncates all app tables first (dev convenience).
2. Deterministic IDs: a seeded ULID factory (fixed timestamp base +
   counter) exported as `SEED_IDS` (typed constants: `SEED_IDS.space`,
   `.guardianMom`, `.roomFamily`, `.flaggedMessage`, …) — importable by
   E2E specs (53) and dev tools.
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
     report; audit trail consistent with all the above (seed writes
     through services where practical — **requirement: seed calls the
     real service layer, not raw inserts, wherever a service exists**,
     so invariants/audit stay true; raw inserts only for timestamp
     control, documented per call site).
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
   setup; same `SEED_IDS`.
6. `pnpm db:seed --demo|--e2e [--force-reset]` root wiring; runs the
   17/18 pipeline for fixture images through MinIO when services are up
   (skip-with-warning otherwise: image rows seeded as `ready` with
   pre-processed variants uploaded directly — both paths implemented).
7. Docs: `docs/dev/seed.md` — personas table, SEED_IDS reference,
   modes, reset warnings, screenshot recipe (`pnpm db:seed --demo` +
   URLs to visit).
8. Tests: seed runs idempotently twice with `--force-reset`; guard
   blocks production/real-data runs; SEED_IDS resolve to rows; the
   flagged message actually trips 27's pipeline when seeded via
   services (assert hit row exists).

## Acceptance Criteria

- [ ] One command yields the full demo family with all safety states
      visible in web UI (manual walkthrough + screenshots in PR).
- [ ] E2E mode + SEED_IDS consumed by at least one converted test in
      this PR (proof of stability).
- [ ] Production guard proven by test.
- [ ] Fixture provenance documented; GPS fixture generated in-repo.

## Validation

```bash
pnpm db:seed --demo --force-reset && pnpm db:seed --e2e --force-reset
pnpm --filter @famchat/db test -- --grep seed
```

## Dependencies

27, 29, 19 (services the seed drives), 18 (image path), 04.

## Non-goals

Load testing data, per-locale full demo duplication (single bilingual
family instead), production data migration tooling.

## Design References

- DESIGN §21 (testing), §15 (bilingual demo), §13 (safety states to
  exhibit)
