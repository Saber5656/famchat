# Issue 20: `apps/web` skeleton — i18n foundation, auth pages, child link page

## Summary

Create the Next.js web app with the i18next foundation (shared catalogs,
`<Furigana>` component, kid-mode theme tokens), the auth surface (login,
password reset, invite accept, child device link), same-origin API
plumbing, and the protected app shell scaffold.

## Context

DESIGN §16 (routes), §15 (i18n), §6 (auth UX). Web and API are served from
one hostname via path routing (prod: Caddy; dev: Next.js rewrites to the
local API) so cookies are first-party and `SameSite=Lax` works — this issue
establishes that plumbing.

## Scope

In scope: app scaffold, `packages/i18n` package creation (catalog structure
+ initial namespaces), theming, tRPC/TanStack Query client with CSRF
header, `(auth)` routes, `/link`, protected layout + `me` bootstrap,
role-aware nav skeleton, Dockerfile.
Out of scope: rooms/chat/board UIs (21–24), guardian console (31/32), push/
PWA (38), settings page content (25).

## Detailed Requirements

1. Scaffold `apps/web` (`@famchat/web`): Next.js 15 App Router, TS strict,
   Tailwind; port 3000; `next.config` rewrites in dev:
   `/trpc/:path* | /ws | /media/:path* → http://localhost:8080/...`
   (prod uses Caddy for the same mapping). **Canonical URL-resolution
   rule** (applies to web everywhere): `NEXT_PUBLIC_API_URL` and
   `NEXT_PUBLIC_WS_URL` default to the empty string, which the client
   helper `apiOrigin()` resolves to `window.location.origin`
   (same-origin path routing); non-empty values override for split-host
   deployments. `API_BASE_URL` is the server-side (SSR/route-handler)
   counterpart, defaulting to `http://localhost:8080` in dev — used only
   where the Next server itself calls the API. These rules live in
   `apps/web/src/lib/env.ts` with the `webPublicEnvSchema` from 03.
2. Extend `packages/i18n` (`@famchat/i18n`) — the package skeleton and
   namespace layout were created in issue 03 and `auth.json` seeded by 06;
   this issue adds the keys these screens need (both locales always) and
   the web-side exports: typed helpers `resources`, `namespaces`, and the
   furigana markup convention (`漢字|かんじ`) parser `parseRuby(value)`.
3. Web i18n init (react-i18next): detector order user-profile → cookie →
   `Accept-Language`; `<Furigana>` React component rendering ruby markup
   (`<ruby>漢字<rt>かんじ</rt></ruby>`); ESLint rule or lint script
   forbidding hardcoded UI strings in `apps/web/src` (regex heuristic for
   JSX text nodes with non-ASCII — warn level; the hard gate is 40).
4. Theme: Tailwind config with famchat design tokens — kid-mode scale
   (base font 18 px in child context, tap targets ≥ 44 px, high-contrast
   palette, rounded-2xl), light default; tokens consumed via CSS variables
   so role-based theming is a class swap (`data-kid="true"`).
5. API client: tRPC + TanStack Query provider; `x-famchat-csrf: 1` header
   on every request; error → `FamchatErrorCode` extraction helper +
   `errors.json` i18n mapping; global 401 handler → redirect `/login`
   (or `/link` when previous session kind was device_link — stored in
   `localStorage.famchat_client_kind`).
6. Routes:
   - `/login`: email+password form; zod client validation; suspended/
     invalid states from error codes; link to `/reset`.
   - `/reset`: request form (always-success message) and token-confirm
     form (`?token=`), new-password with policy hints.
   - `/invite/[code]`: calls `invites.previewInvite`; shows space name +
     role; logged-in accept button OR signup form (email/password/display
     name/locale + ToS consent checkbox with links to `docs/legal`
     placeholders wired properly in 56); success → `/s/<spaceId>`.
   - `/link`: child device link — reads `#c=<code>&s=<spaceId>` from the
     URL fragment when present (QR path; fragment parsed client-side,
     never sent to the server) and prefills; otherwise big 6-digit code
     input (numeric keypad UX). Manual entry needs the code only —
     `auth.childLink({ code, client: 'web' })` per 09's contract (no
     spaceId required). Camera QR scan via `BarcodeDetector` when
     available (progressive enhancement, feature-detected, no external
     lib). **Session handling: the device session is established
     exclusively via the API's `Set-Cookie` (HttpOnly) response — the
     raw token must never be written to localStorage/sessionStorage or
     be readable by JS**; the only client-side artifact is the
     non-secret hint `localStorage.famchat_client_kind = 'device_link'`.
     Kid-toned ja/en strings with furigana; failure states friendly
     (expired → "おうちの人に 新しいコードを もらってね").
   - `(app)/layout`: `auth.me` bootstrap; unauth → redirect; space-scoped
     children get `data-kid` theme + simplified nav (rooms/board only)
     **and the permanent oversight notice element in the shell**
     (`safety.oversight_notice`, furigana — DESIGN §13.3/§16 makes this
     a shell-level requirement, not something deferred to the chat
     screen; 22 additionally shows it in room headers); adult nav:
     spaces, notifications placeholder, settings placeholder. Space
     switcher stub (list from `me`, links to `/s/[spaceId]` — full shell
     in 21).
7. Dockerfile (standalone output) mirroring api patterns; `HEALTHCHECK`
   on `/`.
8. Tests: vitest + @testing-library for `<Furigana>` (parses markup,
   renders rt), login form validation, CSRF header presence (mock fetch
   assertion), 401 redirect logic; Playwright smoke (dev stack): login
   page renders in ja and en via `?lng=` override.

## Acceptance Criteria

- [ ] Full auth surface works against the dev API: login, reset via
      Mailpit link, invite accept (new + existing user), child link with a
      code from `children.createLinkCode`.
- [ ] All strings exist in ja **and** en; child-facing ja strings carry
      furigana markup; no hardcoded JSX strings (lint clean).
- [ ] Cookies flow same-origin through the dev rewrite (manual + Playwright
      check); CSRF header present on all mutations; no session token ever
      readable from JS storage (grep + runtime assertion in the link
      Playwright spec).
- [ ] Kid-mode theme + shell oversight notice apply for child sessions.
- [ ] Camera access is requested only on `/link` (noted for 38/54's
      `Permissions-Policy: camera=(self)`-scoped header verification).

## Validation

```bash
pnpm --filter @famchat/web typecheck && pnpm --filter @famchat/web test
pnpm --filter @famchat/web dev  # manual: /login /reset /invite/x /link
pnpm --filter @famchat/web exec playwright test --grep @smoke
```

## Dependencies

06, 08, 09 (APIs consumed). Creates `packages/i18n` used by 40/41.

## Non-goals

Chat/board/guardian UIs, PWA/push (38), settings content (25), locale
switcher polish (40), production Caddy wiring (49).

## Design References

- DESIGN §16 (web app), §15 (i18n + furigana policy), §6.2–6.3 (auth UX,
  CSRF), §19.4; ADR-007
