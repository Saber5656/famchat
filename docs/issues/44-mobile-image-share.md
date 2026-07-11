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

1. Uploader port `src/lib/uploads.ts`: same state machine + polling
   contract as 23 (`picking → requesting → uploading(pct) → processing →
   ready | failed(reason)`); upload via `expo-file-system`
   `uploadAsync` (or fetch with FormData PUT — must send raw bytes, not
   multipart: use `uploadAsync` with `httpMethod: 'PUT'`, binary body)
   honoring the presigned URL + content-type; progress events wired to
   UI; shared constants from `@famchat/shared/limits`.
2. Composer attach: image button → action sheet (library / camera);
   `expo-image-picker` with permissions flows (denied → settings-link
   guidance, localized); preview thumb with remove + caption (500 cap);
   send gated on `ready`; optimistic bubble with local URI.
3. HEIC handling: request `expo-image-picker` with
   `mediaTypes: images`; if the instance rejects HEIC (18 capability →
   specific error code), retry path: re-pick with picker-side conversion
   (`allowsEditing:false, quality:1` exports JPEG on iOS when
   `exif:false` — if conversion unavailable, show the localized guidance
   from 23's HEIC copy). Never silently fail.
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
7. Tests: unit — state machine port parity (shared fixture table with
   23), mediaSource header logic; manual device checklist: camera shot
   (iPhone HEIC path) → appears on web client sanitized; library pick on
   Android; lightbox gestures; permission-denied flows both platforms.

## Acceptance Criteria

- [ ] Photo round-trip device→web and web→device with progress +
      processing states.
- [ ] iPhone camera (HEIC) path works under whichever 18 capability
      outcome, with the specified UX.
- [ ] Children cannot save images; adults can (checklist).
- [ ] All error states localized ja/en.

## Validation

```bash
pnpm --filter @famchat/mobile test -- --grep upload
# manual: device checklist (HEIC camera, Android library, lightbox, permissions)
```

## Dependencies

43 (chat), 18/17 (pipeline). Reference: 23.

## Non-goals

Board image UI (45), multi-image, video, client-side editing, avatar
uploads (v2).

## Design References

- DESIGN §12 (media), §17 (mobile), §19.3 (media mitigations);
  research/platform-constraints.md (HEIC unknown U4)
