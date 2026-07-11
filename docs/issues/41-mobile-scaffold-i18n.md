# Issue 41: `apps/mobile` — Expo scaffold + i18n

## Summary

Create the Expo (React Native) app workspace: expo-router navigation
shell, TypeScript strict, i18next with the shared catalogs and a native
`<Furigana>` renderer, tRPC/TanStack Query client, environment plumbing,
theme tokens, and the dev-build configuration.

## Context

ADR-004 (Expo, internal distribution) and DESIGN §17. Push requires dev
builds (not Expo Go on Android — research/platform-constraints.md §2), so
`expo-dev-client` is configured from the start even though 41–45 can run
in Expo Go.

## Scope

In scope: app scaffold, navigation skeleton, i18n, API client + error
mapping, theme, env config, jest-expo, monorepo integration, LAN dev
docs.
Out of scope: auth screens (42), any feature UI (43–47), push (46), EAS
profiles/builds (48).

## Detailed Requirements

1. Scaffold `apps/mobile` (`@famchat/mobile`) with `create-expo-app`
   (SDK ≥ 53, TS template) adapted to pnpm workspaces (metro config with
   workspace root watch + node_modules resolution per Expo monorepo
   guide); `expo-dev-client` installed; app name famchat, scheme
   `famchat` (deep links consumed in 46), placeholder bundle ids
   `com.example.famchat` (owner replaces in 48 — marked clearly in
   `app.config.ts` comments).
2. `app.config.ts`: reads `EXPO_PUBLIC_API_URL`, `EXPO_PUBLIC_WS_URL`;
   `extra` carries build metadata; no secrets ever (build-time public
   only, DESIGN §3.4).
3. Navigation (expo-router): auth stack group `(auth)` with placeholder
   screens (42 fills), app tab group `(app)`: Rooms / Board /
   Notifications / Settings (+ Guardian tab conditionally — placeholder
   gating on `me` role, 47 fills); typed routes enabled.
4. i18n: react-i18next with `@famchat/i18n` catalogs (metro must resolve
   the workspace package — verify); locale detection: user profile (once
   authed) → `expo-localization` device locale → ja fallback; native
   `<Furigana>` component rendering ruby markup as stacked
   `<Text>` (reading above base, styled) — same parse function as web
   (`parseRuby` from `@famchat/i18n`).
5. API client: tRPC + TanStack Query against `EXPO_PUBLIC_API_URL`;
   bearer-token auth header injection from a token store module (42 fills
   SecureStore; here an in-memory placeholder with the same interface);
   `x-famchat-csrf` header not required for bearer (05) but harmless —
   omit; error → `FamchatErrorCode` mapper reusing `errors.*` catalog;
   global 401 → navigate to `(auth)`.
6. Theme: tokens mirroring web kid-mode scale (shared values exported
   from `@famchat/shared/constants` where sensible — tap targets ≥ 44 pt,
   type scale, palette) via a `ThemeProvider` + `useTheme`; `data-kid`
   equivalent: `KidModeProvider` keyed on membership role.
7. Testing: jest-expo preset; unit tests for `<Furigana>`, error mapper,
   locale detection fallback; `pnpm --filter @famchat/mobile test|
   typecheck|lint` wired into root scripts (CI runs typecheck+unit; no
   native build in CI at this issue).
8. Dev docs `apps/mobile/README.md`: running against local API over LAN
   (find host IP, set `EXPO_PUBLIC_API_URL=http://<lan-ip>:8080`, CORS:
   dev api allows the Expo origin — document adding the LAN origin to
   `WEB_ORIGINS` in dev; note that bearer mode avoids cookie/CORS
   complexity), Expo Go vs dev-build guidance (push needs dev build —
   pointer to research doc).

## Acceptance Criteria

- [ ] App boots in Expo Go (iOS + Android) and in a local dev build,
      showing the tab shell with localized ja/en placeholder screens.
- [ ] `<Furigana>` renders ruby correctly on-device (screenshot in PR).
- [ ] tRPC call to dev API succeeds from a physical device on LAN
      (documented + manually verified; `auth.me` unauthenticated returns
      the expected error shape through the mapper).
- [ ] typecheck/lint/unit green in CI.

## Validation

```bash
pnpm --filter @famchat/mobile typecheck && pnpm --filter @famchat/mobile test
pnpm --filter @famchat/mobile exec expo start  # manual device smoke
```

## Dependencies

03, 20 (shares `@famchat/i18n` created there).

## Non-goals

Auth (42), features (43–45), push/deep links (46), EAS/bundle ids (48),
tablets-specific layouts (v2 polish).

## Design References

- DESIGN §17 (mobile), §15 (i18n), §3.4 (public env); ADR-004;
  research/platform-constraints.md §2
