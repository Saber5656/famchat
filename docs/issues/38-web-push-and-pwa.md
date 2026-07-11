# Issue 38: Web Push (VAPID) + PWA installability

## Summary

Make the web app an installable PWA with Web Push: manifest + icons +
service worker, the VAPID subscribe/unsubscribe flow with per-platform
onboarding (iOS home-screen guide), worker-side webpush transport, and
endpoint lifecycle handling.

## Context

DESIGN §14 (transports), §16 (PWA), research/platform-constraints.md §1:
iOS requires home-screen installation for push, and famchat treats web
push as best-effort (unread state is DB-derived). VAPID keys are
instance-level env (§3.4).

## Scope

In scope: manifest/icons/SW, push subscribe UX + settings toggle, webpush
transport registration in the worker, click-through deep links, iOS
onboarding guide, degradation without VAPID env, manual checklist.
Out of scope: notification center UI (39), Expo push (46), offline message
queue (v1 non-goal — offline shell only).

## Detailed Requirements

1. PWA: `manifest.webmanifest` (name famchat, short_name, ja/en localized
   via route, `display: standalone`, theme/background colors from tokens,
   icons 192/512 + maskable — generate simple lettermark placeholders as
   SVG-derived PNGs committed to repo); iOS meta tags
   (`apple-mobile-web-app-capable`, touch icons); install prompt handling
   (`beforeinstallprompt` deferral + settings "install app" entry on
   supporting browsers).
2. Service worker `apps/web/public/sw.js` (plain JS, no bundler magic):
   `push` event → parse `{ title, body, link, tag }` → `showNotification`
   (tag-based collapse per room); `notificationclick` → focus existing
   client + navigate to `link` or `openWindow`; minimal offline shell:
   cache the app shell route + an `/offline` page (network-first,
   fallback), **no API caching** (`no-store` posture from 12); SW
   versioning + `skipWaiting`/`clientsClaim` update flow with an in-app
   "update available" toast.
3. Subscribe flow: `lib/push.ts` — feature-detect (`PushManager` in SW
   registration); iOS-not-installed detection (`'standalone' in
   navigator && !navigator.standalone` heuristic + UA) → show the
   localized install guide sheet (step-by-step share-sheet instructions
   per research doc) instead of a dead permission prompt; permission
   request only from explicit user action (settings toggle or the
   engagement prompt after first sent message — never on load);
   `pushManager.subscribe({ userVisibleOnly: true, applicationServerKey:
   NEXT_PUBLIC_VAPID_PUBLIC_KEY })` → `notifications.registerPush({ kind:
   'webpush', … })`; unsubscribe symmetric.
4. Settings integration (25): push card — status (on/off/unsupported/
   needs-install), toggle, per-device note; child sessions: push allowed
   (device is family-managed) but copy mentions guardian visibility of
   devices.
5. Worker transport: register `webpush` in 37's dispatch registry using
   the `web-push` lib (VAPID from env; `VAPID_SUBJECT`); payload =
   rendered `{ title, body, link, tag }` JSON ≤ 3 kB; failures: 404/410 ⇒
   `disabled_at` set; ≥ 5 consecutive `failure_count` ⇒ disable; others
   retry via BullMQ backoff. If VAPID env unset: transport not
   registered, API `registerPush(webpush)` returns `VALIDATION_FAILED
   details.push_disabled`, settings UI shows "not configured on this
   server" (self-host graceful degradation).
6. Manual checklist `docs/ops/webpush-checklist.md`: desktop Chrome/
   Firefox, Android Chrome, iOS Safari installed-PWA — subscribe, receive
   (app closed), click-through lands in room, unsubscribe; unsupported
   matrix documented.
7. Tests: unit — subscribe flow state machine incl. iOS-needs-install
   branch and disabled-env branch (mocked APIs); worker — 410 disables,
   payload snapshot, no-VAPID no-register; Playwright (chromium supports
   CDP push): grant permission, subscribe row created, trigger
   notification via API → SW `push` handled and notification shown
   (Playwright's context.grantPermissions + service worker assertion),
   click routes to the room.

## Acceptance Criteria

- [ ] Lighthouse PWA installability passes (manifest + SW + icons).
- [ ] Push round-trip proven in Playwright (chromium) and on the manual
      checklist for Android + iOS-installed.
- [ ] iOS uninstalled state shows the guide, never a broken prompt.
- [ ] Instance without VAPID keys degrades exactly as specified.

## Validation

```bash
pnpm --filter @famchat/web test -- --grep push
pnpm --filter @famchat/web exec playwright test --grep @push
# manual: docs/ops/webpush-checklist.md
```

## Dependencies

37 (framework), 20 (app shell), 25 (settings card slot).

## Non-goals

Offline message queue, background sync, notification center UI (39), Expo
(46), push text minimization option (v2).

## Design References

- DESIGN §14 (notifications), §16 (PWA), §3.4 (VAPID env);
  research/platform-constraints.md §1
