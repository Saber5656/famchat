# Issue 17: Attachment upload API (presigned, quarantine)

## Summary

Implement the attachments table, presigned direct-to-S3 uploads into the
quarantine prefix, the finalize/enqueue step, space media quotas, the
authorized `/media/:id/:variant` redirect endpoint, and flip on
`messages.sendImage`.

## Context

DESIGN §12.1 upload flow and §19.2 upload boundary: clients never stream
bytes through the API; originals live in `q/` and are never served; only
worker-produced `m/` variants (18) are served via short-lived presigned
GETs after membership checks.

## Scope

In scope: migration (attachments incl. `messageId` claim column), S3
service, `attachments` router (requestUpload/finalize/get), quota
accounting, `/media` REST route, sendImage enablement + claim logic (from
14), tests.
Out of scope: processing pipeline (18 — finalize enqueues; without 18 the
attachment stays `processing`), board attachment joins (19), client UI (23).

## Detailed Requirements

1. Migration per DESIGN §7.4 + `messageId text? @unique` (claim edge used
   by 14's sendImage; board uses its own join table in 19 — constraint:
   an attachment may have messageId XOR a board_post_attachments row,
   enforced at claim time in code + a test).
2. S3 service (`@aws-sdk/client-s3` + presigner, path-style from env):
   `presignPut(key, contentType)` (5-min expiry; Content-Type signed),
   `presignGet(key, 60s, { contentDisposition })`, `deleteObject`,
   `headObject`. Keys per DESIGN §7.4 layout. **Size-enforcement
   contract**: presigned PUT cannot reliably enforce a byte cap
   (`content-length-range` is a presigned-POST policy feature), so the
   authoritative size checks happen at finalize (req 4) and again in the
   worker (18); the quota system reserves the declared size at request
   time.
3. `attachments.requestUpload({ spaceId, mime, sizeBytes })` — any member;
   mime ∈ {image/jpeg, image/png, image/webp, image/heic (subject to the
   18 capability flag)}; sizeBytes ≤ `UPLOAD_MAX_BYTES` (env, default
   `UPLOAD_MAX_BYTES_DEFAULT` from `@famchat/shared/limits`); **quota
   check**: sum of space attachments in status pending/processing/ready +
   request ≤ `SPACE_MEDIA_QUOTA_BYTES_DEFAULT` (same limits module;
   operator-adjustable via env in v2 — constant in v1) ⇒ else
   `UPLOAD_TOO_LARGE` with `details.quota`; insert row (status pending,
   original_key `q/<spaceId>/<id>`) → `{ attachmentId, putUrl,
   expiresAt }`. Rate limit `uploadRequest` (20/h/user) registered in the
   12 map. Pending rows expire: cleanup job clears `pending` older than
   24 h (job registered in 18's worker; the query helper ships here).
4. `attachments.finalize({ spaceId, attachmentId })` — uploader only;
   `headObject` confirms existence AND `ContentLength ≤ declared
   sizeBytes` — oversize ⇒ status `rejected`
   (reason `size_exceeded`) + delete the `q/` object (this is the
   authoritative size gate, per the req-2 contract); otherwise status
   pending → processing; enqueue BullMQ `image-process` job (producer
   here, consumer in 18); idempotent on repeat calls.
5. `attachments.get({ spaceId, attachmentId })` — member; returns status +
   dimensions + reject_reason (for uploader UI).
6. REST `GET /media/:attachmentId/:variant(full|thumb)` — session-authed;
   attachment must be `ready`; requester must be a member of the
   attachment's space (room-level checks intentionally relaxed to space
   membership in v1 — documented decision, DESIGN §12.1 note); 302 to
   presigned GET with
   `response-content-disposition=inline; filename="famchat-<id>-<variant>.webp"`
   and `response-content-type=image/webp` query overrides (DESIGN §19.2
   serving controls); redirect response itself carries
   `Cache-Control: private, max-age=55` and
   `X-Content-Type-Options: nosniff`.
7. Enable `messages.sendImage` (14): flip capability, implement atomic
   claim (`UPDATE … SET message_id = $mid WHERE id = $aid AND message_id
   IS NULL AND status='ready' AND space_id=$sid AND uploader_id=$uid` —
   0 rows ⇒ `ATTACHMENT_NOT_READY`/`VALIDATION_FAILED` per cause);
   unskip its tests.
8. Tests: oversize upload rejected at finalize (upload more bytes than
   declared → `rejected` + q/ object deleted); mime rejection; quota
   arithmetic incl. pending reservations; finalize before upload fails;
   double-claim race yields one winner; `/media` denies non-members +
   not-ready + bad variant (404); redirect headers incl.
   content-disposition override present; GET URL expires (skew-tolerant
   assertion); cross-tenant sweep.

## Acceptance Criteria

- [ ] Upload flow parks cleanly at `processing` (this issue's boundary);
      the full request → PUT → finalize → ready → sendImage → `/media`
      loop is asserted by issue 18's cross-issue integration test once
      the worker exists (explicitly deferred there).
- [ ] Authoritative size gate at finalize proven by test (presigned PUT
      is not trusted for size).
- [ ] Quarantine keys never appear in any served URL (test greps DTOs).
- [ ] Quota + rate limits enforced; claim is race-safe.

## Validation

```bash
pnpm --filter @famchat/db db:migrate
pnpm -w typecheck
pnpm --filter @famchat/api test -- -t attachments
```

## Dependencies

12, 14. (18 consumes the queue; 23/44 build UIs.)

## Non-goals

Processing (18), video/gif (v1 non-goal), avatar uploads (v2), CDN.

## Design References

- DESIGN §7.4 (schema/layout), §12.1 (flow), §19.2/§19.5 (upload boundary,
  quotas)
