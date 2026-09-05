# Research: Platform constraints affecting famchat v1 (push, PWA, distribution)

Status: verified 2026-07-11 via web sources below; re-verify at implementation time.
Design impact: DESIGN.md §14 (notifications), §16–17 (clients), ADR-004.

## 1. iOS Web Push (affects web PWA on iPhone/iPad)

Verified facts (2026-07):

- The Push API on iOS works **only for Home Screen web apps** (Safari → Share
  → Add to Home Screen). A normal Safari tab — or Chrome/Firefox on iOS,
  which are all WebKit — has no `PushManager`.
- Requires iOS 16.4+ (≈95%+ of devices in 2026).
- There is **no automatic install prompt** on iOS; users must manually use
  the share sheet.
- EU-region iOS (17.4+) has additional home-screen web app restrictions;
  Japan (primary market) is unaffected, but this matters if EU users adopt
  the OSS build.

Design consequences:

1. Web onboarding must include a per-platform "enable notifications" guide
   (iOS: install to Home Screen first → then grant permission). Issue 35
   includes this guide as an acceptance criterion.
2. Web push on iOS is treated as **best-effort**; families wanting reliable
   child-device notifications are steered to the Expo app (TestFlight/
   internal) in docs.
3. Notification design never assumes delivery (unread state is always
   DB-derived).

## 2. Expo / EAS distribution and push

Verified facts (2026-07):

- Current Expo SDK generation: **SDK 55** (React Native 0.79+). Floor pinned
  in DESIGN.md is SDK ≥ 53.
- `expo-notifications` **does not work in Expo Go on Android since SDK 53**;
  a development build (`expo-dev-client`) is required for push testing.
- Expo push service (`expo-server-sdk-node`) requires an Expo project ID;
  APNs/FCM credentials are managed via EAS.
- TestFlight (iOS) requires an Apple Developer Program membership ($99/yr);
  Play internal testing requires a Play Console account ($25 one-time).
  Owner registers both manually; agents never touch store credentials.

Design consequences:

1. Mobile issues assume **development builds**, not Expo Go, from the first
   push-related issue onward (Expo Go acceptable only up to issue 42).
2. Push credentials (APNs key, FCM service account) are owner-provisioned
   secrets, documented as manual steps in the EAS runbook (issue 45).
3. `EXPO_ACCESS_TOKEN` on the worker is optional: web-only self-hosters can
   run without Expo push.

## 3. Store publication requirements (v2 gate — recorded now to avoid design traps)

- Apple **Kids Category** and Google **Play Families** impose: parental
  gates, no third-party analytics/ads SDKs, published moderation policy, UGC
  apps must provide block/report/moderation mechanics, privacy labels/data
  safety forms.
- v1 already ships report + guardian moderation + zero third-party trackers,
  so nothing in the v1 design forecloses store submission; remaining v2 work
  is process/compliance, not architecture.

## Sources

- [Expo: Push notifications setup](https://docs.expo.dev/push-notifications/push-notifications-setup/)
- [Expo: Notifications SDK](https://docs.expo.dev/versions/latest/sdk/notifications/)
- [expo-server-sdk-node](https://github.com/expo/expo-server-sdk-node)
- [Expo push notifications guide 2026 (reactnativerelay)](https://reactnativerelay.com/article/react-native-push-notifications-expo-complete-guide-2026)
- [PWA push on iOS 2026 (webscraft)](https://webscraft.org/blog/pwa-pushspovischennya-na-ios-u-2026-scho-realno-pratsyuye?lang=en)
- [PWA iOS limitations (MagicBell)](https://www.magicbell.com/blog/pwa-ios-limitations-safari-support-complete-guide)
- [OneSignal: iOS web push setup](https://documentation.onesignal.com/docs/en/web-push-for-ios)
- [MobiLoud: PWAs on iOS 2026](https://www.mobiloud.com/blog/progressive-web-apps-ios)
