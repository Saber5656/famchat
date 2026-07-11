# Issue 46: Mobile push (Expo) + deep links

## Summary

Wire mobile push end-to-end: expo-notifications registration with an
engagement-gated permission prompt, the Expo transport in the worker
(chunked sends, receipt polling, dead-token cleanup), deep-link routing
from notification taps, foreground handling, and badge counts.

## Context

DESIGN §14 (transports), research/platform-constraints.md §2: Android
push requires a dev build (not Expo Go); Expo push service credentials
(APNs key / FCM service account) are owner-provisioned via EAS —
credential provisioning is documented here but executed by the owner in
48. The worker transport slot was prepared in 37.

## Scope

In scope: device registration flow, worker `expo` transport + receipts
job, deep-link map, foreground/badge behavior, self-host degradation
(no `EXPO_ACCESS_TOKEN`), device checklist.
Out of scope: EAS build profiles/credentials execution (48), web push
(38), notification center UI parity (feed screen ships here minimally —
list + read, reusing 39's renderer rules).

## Detailed Requirements

1. Registration (`src/lib/push.ts`): prompt strategy — never on first
   launch; after the user's first successful message send (engagement
   gate per DESIGN §17), show a friendly pre-prompt sheet (ja/en,
   child-register for child sessions) then the OS permission;
   `getExpoPushTokenAsync({ projectId })` → `notifications.registerPush
   ({ kind: 'expo', token })`; re-validate token on every cold start
   (tokens rotate); unregister on logout/revocation (42's wipe hook);
   Android notification channel setup (`default`, importance high) +
   iOS foreground presentation options.
2. Worker transport: register `expo` in 37's dispatch registry using
   `expo-server-sdk`: validate token format, chunk sends (SDK
   chunking), map rendered `{ title, body, link }` → message
   (`data: { link }`, `sound: 'default'`, `badge` from recipient's
   unread aggregate — computed via 37's unreadCounts helper);
   `DeviceNotRegistered` ⇒ disable subscription; store tickets and run
   a `expo-receipts` BullMQ job 15 min later resolving receipt errors
   (same disable rules); absent `EXPO_ACCESS_TOKEN` ⇒ transport not
   registered + `registerPush(expo)` returns `VALIDATION_FAILED
   details.push_disabled` (mirror of 38's degradation).
3. Deep links: notification tap (`expo-notifications` response
   listener) + cold-start last-response → route by `link` using the
   same app-route shapes as web (`/s/<sid>/r/<rid>?m=<mid>`,
   `/s/<sid>/board/<postId>`, `/notifications`) via an expo-router
   `routeFromLink(link)` mapper (unit-tested table incl. unknown-link
   fallback to Rooms); OS-level universal/app links config (associated
   domains / intent filters) declared in `app.config.ts` but activation
   documented as part of 48 (needs owned domain files).
4. Foreground behavior: notifications received while the relevant room
   is open and focused are suppressed (no banner — the message arrives
   via WS); other rooms → in-app banner (expo-notifications foreground
   handler consulting current route); badge count synced on feed
   read/markAll (39 semantics) via `setBadgeCountAsync`.
5. Notifications tab (minimal): feed list reusing 37's row shape +
   39's renderer rules (type icons, explicit-read, deep links) — full
   parity assertions live in 39; this screen must at least render all
   types safely (unknown-type fallback).
6. Tests: unit — routeFromLink table, prompt-gating state machine,
   badge sync logic; worker — transport chunking/receipt/disable paths
   with SDK mocked, no-token degradation; manual device checklist
   (dev build, both platforms): receive with app killed → tap →
   correct room; foreground suppression in-room; badge increments/
   clears; child device receives `message.new` but guardian safety
   types never appear (backend guarantee re-verified on device);
   quiet-hours child gets no push but sees feed row after unlock.

## Acceptance Criteria

- [ ] Push round-trip on physical iOS (TestFlight/dev build) and
      Android (dev build) devices with deep-link landing (checklist
      evidence in PR).
- [ ] Receipt polling + dead-token disabling proven in worker tests.
- [ ] Engagement-gated prompt: zero permission requests before first
      send (state-machine test + checklist).
- [ ] Self-host without Expo token degrades exactly as specified.

## Validation

```bash
pnpm --filter @famchat/mobile test -- --grep push
pnpm --filter @famchat/worker test -- --grep expo
# manual: two-platform device checklist
```

## Dependencies

43 (app flows), 37 (framework). Credentials/builds finalized in 48.

## Non-goals

EAS credential execution (48), web push (38), notification sounds
customization, per-type mobile settings (v2).

## Design References

- DESIGN §14 (catalog/transports), §17 (mobile), §9 (WS vs push
  interplay); research/platform-constraints.md §2
