# Issue 18: `apps/worker` + image processing pipeline

## Summary

Create the worker process (BullMQ consumers, env, logging, Dockerfile) and
implement the image job: sniff, decode, strip all metadata, re-encode to
webp full+thumb variants, publish to the serve prefix, delete the original,
and manage attachment state transitions.

## Context

DESIGN §12.1: re-encoding is unconditional — it is the security control
that defeats polyglots and guarantees EXIF/GPS removal (§19.3). The worker
is also the future home of notification fanout (37) and purge/export jobs
(36), so its scaffold must be job-generic.

## Scope

In scope: worker scaffold, image-process consumer, state machine, fixture-
based tests, pending-cleanup job, Dockerfile, HEIC decision (U4).
Out of scope: notification fanout (37), purge/export jobs (36), client UI.

## Detailed Requirements

1. Scaffold `apps/worker` as `@famchat/worker`: `loadEnv(workerEnvSchema)`;
   pino logging (same redaction as api); BullMQ `Worker` registry pattern
   (`src/jobs/<name>.ts` exporting `{ name, processor, opts }`, index
   registers all); graceful shutdown (close workers, drain in-flight ≤ 30
   s); liveness: tiny HTTP server on `:8081/healthz` returning queue
   connectivity; Dockerfile mirroring the api one.
2. `image-process` job (queue name `image-process`, jobId = attachmentId
   for dedupe; attempts 3, exponential backoff 5 s base):
   1. Load attachment; skip unless status `processing` (idempotency).
   2. GET original from `q/`; verify actual size ≤ declared size exactly
      (no slack — finalize already enforced this; a mismatch here means
      the object changed and is rejected `size_mismatch`).
   3. Magic-byte sniff via `file-type`; must match a permitted image type
      AND the declared mime family; mismatch ⇒ reject
      (`reject_reason='type_mismatch'`).
   4. `sharp(input, { limitInputPixels: 80_000_000 })`; reject if either
      side > 12_000 px (`too_large_dimensions`) or animated
      (`animated_unsupported`); `.rotate()` (EXIF orientation applied,
      then discarded); output **without** `withMetadata()` so all
      EXIF/GPS/XMP/ICC is dropped; encode `full.webp` (fit inside
      2048×2048, quality 82) and `thumb.webp` (inside 512×512, quality
      78); record output dimensions.
   5. PUT both to `m/<spaceId>/<attachmentId>/…` with
      `ContentType: image/webp` set on the objects (serving-side
      Content-Disposition is applied by 17's presigned-GET overrides);
      update row (serve_key, thumb_key, width, height, status `ready`);
      DELETE `q/` original.
   6. Any decode/processing error after attempts ⇒ status `rejected` +
      reason `processing_failed`; original deleted on rejection too (no
      quarantined hostile bytes retained).
3. HEIC (known unknown U4) — concrete capability contract: on every
   worker boot, decode a bundled sample HEIC fixture; write Redis key
   `caps:heic` = `"1"` (decodable) or `"0"` (not), no TTL, overwritten
   each boot. The api's `attachments.requestUpload` (17) reads the key
   per request (cached 60 s in-process); missing key or `"0"` ⇒ reject
   `image/heic` with `UPLOAD_TYPE_UNSUPPORTED` +
   `details.reason='heic_unavailable'` (conservative default when the
   worker has never booted). Tests cover both key states via injected
   Redis. Record the runtime outcome and the chosen base-image decision
   in `docs/ops/heic.md` (created by this issue).
4. Pending-cleanup job (`attachments-cleanup`, repeatable hourly): delete
   `pending` rows older than 24 h + their `q/` objects (query helper from
   17).
5. Failure alerting: final-failure log at `error` with attachmentId (never
   content); BullMQ metrics counts exposed on the worker healthz payload.
6. Tests (vitest + fixtures in `apps/worker/test/fixtures/`): jpeg with
   GPS EXIF ⇒ output webp has **no** metadata (assert
   `await sharp(output).metadata()` yields `exif`, `icc`, `xmp` all
   undefined) and correct max dimensions; png transparent ok; fake-mime
   (renamed text file) rejected `type_mismatch`; dimension-bomb fixture
   (small-byte, > 80 MP or side > 12k px) rejected
   `too_large_dimensions`; animated GIF/WebP fixture rejected
   `animated_unsupported`; oversized-object rejected `size_mismatch`;
   corrupt jpeg rejected `processing_failed` after retries; idempotent
   re-run on `ready` is a no-op; original deleted in both success and
   reject paths; cleanup job removes stale pendings. **Cross-issue
   integration test (api+worker, closes 17's deferred AC): request →
   PUT → finalize → worker processes → ready → sendImage → `/media` 302
   serves webp bytes.**

## Acceptance Criteria

- [ ] GPS-tagged fixture provably sanitized (metadata absent in output).
- [ ] All reject reasons exercised; originals never survive terminal
      states.
- [ ] Full end-to-end with 17: request→PUT→finalize→ready→`/media` serves
      the webp (integration test spanning api+worker).
- [ ] HEIC capability decision recorded and enforced consistently.

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/worker test
pnpm --filter @famchat/api test -- -t "attachments e2e"
```

## Dependencies

17 (queue producer + schema), 02.

## Non-goals

Notification fanout (37), purge/export (36), video, AI image moderation
(v2), CDN caching.

## Design References

- DESIGN §3.2 (worker), §12.1 (pipeline), §19.2/§19.3 (upload boundary,
  malicious-image mitigations); research/platform-constraints.md (U4)
