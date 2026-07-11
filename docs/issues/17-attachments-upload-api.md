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
   `presignPut(key, contentType, maxBytes)` with `content-length-range`
   condition (5-min expiry), `presignGet(key, 60s)`, `deleteObject`,
   `headObject`. Keys per DESIGN §7.4 layout.
3. `attachments.requestUpload({ spaceId, mime, sizeBytes })` — any member;
   mime ∈ {image/jpeg, image/png, image/webp, image/heic}; sizeBytes ≤
   `UPLOAD_MAX_BYTES`; **quota check**: sum of space attachments in status
   pending/processing/ready + request ≤ `SPACE_MEDIA_QUOTA` ⇒ else
   `UPLOAD_TOO_LARGE` with `details.quota`; insert row (status pending,
   original_key `q/<spaceId>/<id>`) → `{ attachmentId, putUrl,
   expiresAt }`. Rate limit `uploadRequest` (20/h/user) registered in the
   12 map. Pending rows expire: cleanup job clears `pending` older than
   24 h (job registered in 18's worker; the query helper ships here).
4. `attachments.finalize({ spaceId, attachmentId })` — uploader only;
   `headObject` confirms existence + size ≤ declared; status pending →
   processing; enqueue BullMQ `image-process` job (producer here, consumer
   in 18); idempotent on repeat calls.
5. `attachments.get({ spaceId, attachmentId })` — member; returns status +
   dimensions + reject_reason (for uploader UI).
6. REST `GET /media/:attachmentId/:variant(full|thumb)` — session-authed;
   attachment must be `ready`; requester must be a member of the
   attachment's space (room-level checks intentionally relaxed to space
   membership in v1 — documented decision, DESIGN §12.1 note); 302 to
   presigned GET; `Cache-Control: private, max-age=55`;
   `X-Content-Type-Options: nosniff`.
7. Enable `messages.sendImage` (14): flip capability, implement atomic
   claim (`UPDATE … SET message_id = $mid WHERE id = $aid AND message_id
   IS NULL AND status='ready' AND space_id=$sid AND uploader_id=$uid` —
   0 rows ⇒ `ATTACHMENT_NOT_READY`/`VALIDATION_FAILED` per cause);
   unskip its tests.
8. Tests: presign constraints (oversize PUT rejected by MinIO — assert via
   real upload attempt); mime rejection; quota arithmetic incl. pending
   reservations; finalize before upload fails; double-claim race yields
   one winner; `/media` denies non-members + not-ready + bad variant
   (404); GET URL expires (skew-tolerant assertion); cross-tenant sweep.

## Acceptance Criteria

- [ ] End-to-end (with 18 running locally): request → PUT → finalize →
      ready → sendImage → `/media` 302 serves bytes. Without 18: flow
      parks at `processing` cleanly.
- [ ] Quarantine keys never appear in any served URL (test greps DTOs).
- [ ] Quota + rate limits enforced; claim is race-safe.

## Validation

```bash
pnpm --filter @famchat/api test -- --grep attachments
```

## Dependencies

12, 14. (18 consumes the queue; 23/44 build UIs.)

## Non-goals

Processing (18), video/gif (v1 non-goal), avatar uploads (v2), CDN.

## Design References

- DESIGN §7.4 (schema/layout), §12.1 (flow), §19.2/§19.5 (upload boundary,
  quotas)
