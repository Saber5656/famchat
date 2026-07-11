# Issue 23: Web image share UI

## Summary

Add image sending to the web chat composer and image rendering to the
message list: picker/drag-drop/paste, upload progress against the
presigned flow, processing states, thumbnails, and a lightbox. (No camera
capture on web — DESIGN §19.4 scopes camera permission to the link page
only; photos are taken with the device camera app or the mobile app.)

## Context

DESIGN §12 (media pipeline) and §16. The client uploads directly to S3 via
presigned PUT (17), waits for worker processing (18), then sends the
message referencing the ready attachment (14/17).

## Scope

In scope: composer attach button + drag-drop + paste, upload state machine
UI, image message bubbles, lightbox, caption, error states.
Out of scope: board post images (24 reuses the uploader component built
here), mobile (44), multi-image messages (one per message by design).

## Detailed Requirements

1. Shared uploader component `useAttachmentUpload()` (placed in
   `apps/web/src/lib/uploads.ts`, reused by 24):
   states `picking → requesting → uploading(pct) → processing → ready |
   failed(reason)`; flow: client-side pre-checks (mime allowlist, size ≤
   `UPLOAD_MAX_BYTES` with friendly error before any network) →
   `attachments.requestUpload` → presigned PUT executed via
   **XMLHttpRequest** (fetch has no upload-progress events; XHR
   `upload.onprogress` drives the percentage) → `attachments.finalize` →
   poll `attachments.get` every 1.5 s (cap 60 s → treat as failed
   `processing_timeout`, keep polling in background and recover if it
   turns ready).
2. Composer integration: attach button (image icon, kid-size; opens the
   OS file picker — on phones the OS sheet itself offers the camera, no
   in-app camera code), drag-drop onto the room pane,
   paste-from-clipboard; preview thumbnail with remove; optional caption
   (500 cap, counter); send disabled until `ready`; sending calls
   `messages.sendImage`; optimistic bubble uses the local object URL
   until server message arrives.
3. Message rendering: image bubbles use `/media/<id>/thumb` (aspect-
   preserving, max 320 px, blur-up placeholder from width/height), click →
   lightbox (`/media/<id>/full`, zoom, download button — adults only for
   download per child-safety conservatism, documented UI decision),
   caption below.
4. Error states (all localized ja/en, kid-friendly): `UPLOAD_TOO_LARGE`
   (with size), `UPLOAD_TYPE_UNSUPPORTED`, quota exceeded (guardian hint),
   rejected reasons from 18 mapped via `errors.json`
   (`type_mismatch`, `too_large_dimensions`, `animated_unsupported`,
   `processing_failed`), offline/retry.
5. HEIC: if the instance rejects HEIC (18's capability), the picker error
   explains iPhone-photo conversion (localized) — detect from the specific
   error code detail.
6. Tests: component tests for the state machine (mock tRPC + XHR: happy,
   too-large pre-check, reject-after-processing, timeout-recover);
   Playwright: real upload of a fixture jpeg through dev api+worker →
   thumb renders for the second browser; GPS-fixture sanitization
   assertion — the Playwright spec downloads the served
   `/media/<id>/full` bytes with the test session's request context and
   asserts metadata absence **in the Node test process** via
   `sharp(buffer).metadata()` (no new test API routes, no direct S3
   credentials in the browser).

## Acceptance Criteria

- [ ] End-to-end photo share works between two browsers on the dev stack,
      including progress and processing states.
- [ ] Every failure mode above has a distinct localized message.
- [ ] Render-path invariant: the Playwright spec intercepts network
      requests during chat rendering and asserts every image the app
      loads is a `/media/...` URL (the presigned `q/` PUT URL is used
      exactly once, for upload, and never fetched/rendered).
- [ ] Uploader component API documented for 24's reuse.

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/web test -- -t upload
pnpm --filter @famchat/web exec playwright test --grep @image
```

## Dependencies

22, 18 (worker running), 17.

## Non-goals

Multi-image per message, client-side resize, board UI (24), avatar upload
(v2), video.

## Design References

- DESIGN §12 (media), §16 (web), §8.2 (`attachments`), §19.3 (media
  mitigations)
