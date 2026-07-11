# Issue 42: Mobile auth — login, QR device link, PIN lock

## Summary

Implement mobile authentication: guardian/adult email login, the child
device-link flow with camera QR scanning and manual code entry, secure
token storage, the PIN unlock screen, invite-accept deep links, and
revocation/logout handling.

## Context

DESIGN §17 (mobile auth surface), §6.2 (device link), ADR-008 (PIN is a
sibling lock, not a security boundary). The child link flow is the
make-or-break onboarding moment for the product's primary user.

## Scope

In scope: `(auth)` screens (welcome, login, link, PIN), SecureStore token
store, session bootstrap, invite deep link accept, revoked-session
handling.
Out of scope: push registration (46), feature screens (43+), EAS (48),
biometrics (v2).

## Detailed Requirements

1. Welcome screen: two large paths — 「おとなの ログイン」(login) and
   「こどもの きき を つなぐ」(link), ja/en with furigana on the child
   path; version footer.
2. Login screen (adults): email/password with zod validation,
   `auth.login({ client: 'mobile' })`, error mapping (invalid/suspended/
   rate-limited), forgot-password → opens web `/reset` in the browser
   (`expo-web-browser`) — no native reset flow in v1 (documented).
3. Token store (`src/lib/tokenStore.ts`, replaces 41's placeholder):
   expo-secure-store keychain entry `famchat_session`; stores `{ token,
   client: 'mobile'|'device_link', userId }`; exposed as the interface 41
   defined; cleared on logout/revocation.
4. Child link screen: camera QR scan (`expo-camera` barcode scanning,
   permission-gated with friendly fallback) parsing the canonical
   **fragment** URL format from 09/DESIGN §6.2
   (`…/link#c=<code>&s=<spaceId>` — parse the fragment, never emit the
   code into any query string or log; unit-test the parser with
   fragment, legacy-query rejection, and malformed inputs), plus manual
   6-digit entry (large numeric keypad UI); calls
   `auth.childLink({ code, client: 'mobile' })` (code alone suffices per
   09); success → celebratory screen (child register) → app; failures
   friendly per 20's copy (expired → ask guardian for a new code);
   camera permission denial → manual entry emphasized.
5. Session bootstrap: on app start, token present → `auth.me` →
   route to app (kid mode per role) or auth on 401; splash held until
   resolution (expo-splash-screen).
6. PIN lock: `auth.me` gains `hasPin: boolean` for child sessions — an
   API addition made in this issue: extend the me DTO schema in
   `packages/shared/src/api/auth.ts`, populate from
   `child_settings.pin_hash IS NOT NULL`, and add an api test for both
   states. Lock overlay on cold start and on returning from background
   after > 5 min (AppState listener); 4–6 digit pad; verify via
   `auth.verifyChildPin` (rate-limited per 09); offline/unreachable →
   allow if last successful verify < 24 h (timestamp in SecureStore —
   the **offline grace rule defined in ADR-008**, which this issue's
   sibling ADR update records: PIN is a sibling deterrent, not a
   security boundary, so availability wins offline; revocation remains
   the boundary); guardian-reset messaging on repeated failure
   ("おうちの人に きいて ね").
7. Invite deep link: `famchat://invite?code=…` and universal
   `${APP_BASE_URL}/invite/<code>` (route config now; OS association
   files in 48) → preview screen (08 API) → logged-in accept, or
   redirect to login-then-accept (state preserved).
8. Revocation/logout: 401 or WS `session.revoked` (listener registered
   once 43's socket lands — here handle the 401 path) → wipe token store
   → welcome screen with localized notice; logout button in Settings tab
   (works pre-47 via a minimal settings screen stub listing logout +
   locale + version).
9. Tests: unit — QR URL parser (valid/foreign/expired formats), PIN
   grace logic with fake timers, token store interface; jest-expo
   component tests for login validation + link manual entry; manual
   device checklist in PR: real QR from web guardian console (32) links
   a device end-to-end on iOS + Android.

## Acceptance Criteria

- [ ] Child device link works end-to-end on physical devices via QR and
      manual code (checklist evidence in PR).
- [ ] Tokens only in SecureStore (grep test: no AsyncStorage of tokens);
      wipe on logout/revocation proven.
- [ ] PIN lock behavior incl. background timeout + offline grace exactly
      per ADR-008.
- [ ] All auth strings ja/en, child paths furigana-complete.

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/shared test
pnpm --filter @famchat/api test -- -t "auth.me"
pnpm --filter @famchat/mobile test -- -t auth
# manual: device checklist (QR link, PIN, revocation from guardian console)
```

## Dependencies

41, 09 (link API), 06. Guardian console (32) recommended for manual
verification.

## Non-goals

Push (46), biometric unlock (v2), native password reset, account
switching on one device (v2).

## Design References

- DESIGN §17 (mobile), §6.2–6.3 (device link, sessions); ADR-008
