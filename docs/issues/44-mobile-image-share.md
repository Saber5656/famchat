# Issue 44: Mobile image share

## Summary

Add photo sending and viewing to mobile chat: library picker and camera
capture, the presigned upload state machine with progress, processing
states, thumbnails via expo-image, and a lightbox — mirroring web (23).

## Context

DESIGN §12 (pipeline is client-agnostic), §17. HEIC matters most here
(iPhone camera default) — the 18 capability decision surfaces in this UI.

## Scope

In scope: attach flow in the chat composer, upload state machine port,
image bubbles + lightbox, caption, permission handling, HEIC UX.
Out of scope: board images on mobile (45 reuses this uploader), multiple
images per message (design: one), editing/filters (never).

## Detailed Requirements

1. Uploader port `apps/mobile/src/lib/uploads.ts`: same state machine +
   polling contract as 23 — the state-machine test fixture table is
   shared: 23 exports it from
   `packages/shared/src/testing/upload-fixtures.ts` and this issue's
   unit tests consume the same table (parity by construction). Upload
   transport — exactly one sanctioned path:
   `FileSystem.uploadAsync(putUrl, fileUri, { httpMethod: 'PUT',
   uploadType: FileSystemUploadType.BINARY_CONTENT, headers:
   { 'Content-Type': mime } })` (raw bytes; never fetch/FormData/
   multipart); progress via the task's progress callback; shared
   constants from `@famchat/shared/limits`.
2. Composer attach: image button → action sheet (library / camera);
   `expo-image-picker` with permissions flows (denied → settings-link
   guidance, localized); preview thumb with remove + caption (500 cap);
   send gated on `ready`; optimistic bubble with local URI.
3. HEIC handling — exact contract: when `attachments.requestUpload`
   rejects with `UPLOAD_TYPE_UNSUPPORTED` +
   `details.reason === 'heic_unavailable'` (17/18's capability
   contract), the client converts locally and retries once:
   `ImageManipulator.manipulateAsync(uri, [], { format:
   SaveFormat.JPEG, compress: 0.92 })` → re-run requestUpload with
   `image/jpeg` and the new size. Conversion failure → 23's localized
   HEIC guidance copy. Never silently fail; unit-test the
   detect→convert→retry state transitions with the error fixture.
4. Rendering: bubbles use `/media/<id>/thumb` via expo-image (blurhash-
   free; width/height-based placeholder), tap → lightbox modal
   (`/media/full`, pinch-zoom via `react-native-gesture-handler` +
   `react-native-reanimated` already in Expo template, double-tap zoom,
   swipe-down dismiss); adults: save-to-device button
   (`expo-media-library`, permissioned); children: no save button
   (mirrors 23's download rule).
5. Auth for media GETs: `/media` requires the bearer header — expo-image
   supports `source.headers`; centralize a `mediaSource(attachmentId,
   variant)` helper adding the token header (and note the 302→S3 hop:
   expo-image follows redirects; presigned URL needs no header —
   verified in test).
6. Error states: full 23 matrix localized (too large, type, quota,
   rejected reasons, timeout-recover, offline).
7. Tests: unit — state machine parity via the shared fixture table
   (req 1), HEIC detect→convert→retry, mediaSource header logic;
   component test — image bubble renders through a mocked `/media` 302
   (assert the auth header on the first hop and that the redirect is
   followed); manual device checklist: camera shot (iPhone HEIC path) →
   appears on web client sanitized; library pick on Android;
   authenticated `/media` render on a real device; lightbox gestures;
   permission-denied flows both platforms.

## Acceptance Criteria

- [ ] Photo round-trip device→web and web→device with progress +
      processing states.
- [ ] iPhone camera (HEIC) path works under whichever 18 capability
      outcome, with the specified UX.
- [ ] Children cannot save images; adults can (checklist).
- [ ] All error states localized ja/en.

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/mobile test -- -t upload
# manual: device checklist (HEIC camera, Android library, /media render, lightbox, permissions)
```

## Dependencies

43 (chat), 18/17 (pipeline). Reference: 23.

## Non-goals

Board image UI (45), multi-image, video, client-side editing, avatar
uploads (v2).

## Design References

- DESIGN §12 (media), §17 (mobile), §19.3 (media mitigations);
  research/platform-constraints.md (HEIC unknown U4)
