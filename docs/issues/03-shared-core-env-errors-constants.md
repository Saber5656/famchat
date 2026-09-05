# Issue 03: `@famchat/shared` — env validation, error codes, constants, limits

## Summary

Create the `packages/shared` workspace package holding zod-validated env
schemas, the canonical `FamchatErrorCode` union, domain constants/enums, the
rate-limit/quota constants, and the audit action catalog — the single source
of truth every other package imports.

## Context

DESIGN §4 mandates that all cross-boundary schemas/constants live in
`packages/shared`. Later issues extend this package (authz in 10, ws in 15,
quiet hours in 29); this issue creates its foundation.

## Scope

In scope: package scaffold, `env.ts`, `errors.ts`, `constants.ts`,
`limits.ts`, `audit.ts`, `ids.ts`, unit tests, build config.
Out of scope: authz matrix (10), API DTO schemas (per-feature issues), ws
schemas (15), quiet-hours schema (29).

## Detailed Requirements

1. Scaffold `packages/shared` as `@famchat/shared`: `tsconfig.json` extends
   base; vitest configured; `exports` map with subpath exports
   (`./env`, `./errors`, `./constants`, `./limits`, `./audit`, `./ids`);
   `typecheck`/`test`/`build` scripts (tsc build to `dist`, or tsx-based
   consumption — choose tsc emit for portability).
2. `src/env.ts`: zod schemas + loader covering the DESIGN §3.4 matrix
   completely across four schemas: `apiEnvSchema`, `workerEnvSchema`
   (server processes; required/optional/defaults exactly as the matrix),
   `webPublicEnvSchema` (`NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_WS_URL`,
   `NEXT_PUBLIC_VAPID_PUBLIC_KEY`), and `mobilePublicEnvSchema`
   (`EXPO_PUBLIC_API_URL`, `EXPO_PUBLIC_WS_URL`) — the public schemas are
   consumed by the client apps at build/boot (20/41). A unit test
   cross-checks that the union of all four schemas' keys equals the §3.4
   variable list verbatim (hardcoded in the test), so matrix drift fails
   CI. `loadEnv(schema)` parses `process.env` and on failure prints every
   missing/invalid key then exits(1). No secret defaults. `NODE_ENV` enum
   `development|production|test`.
3. `src/errors.ts`: `export const FAMCHAT_ERROR_CODES = [...] as const` with
   at minimum: `AUTH_INVALID_CREDENTIALS, AUTH_SESSION_EXPIRED,
   INVITE_INVALID_OR_EXPIRED, PERMISSION_DENIED, NOT_A_MEMBER,
   ROOM_ARCHIVED, QUIET_HOURS_ACTIVE, CONTENT_BLOCKED_NG_WORD,
   UPLOAD_TOO_LARGE, UPLOAD_TYPE_UNSUPPORTED, ATTACHMENT_NOT_READY,
   RATE_LIMITED, LAST_GUARDIAN, SPACE_SUSPENDED, VALIDATION_FAILED,
   NOT_FOUND` (DESIGN §8.3); type `FamchatErrorCode`; class
   `FamchatError extends Error { code; details?: Record<string, unknown> }`.
4. `src/constants.ts`: `ROLES = ['guardian','adult','child']`,
   `ROOM_TYPES = ['family','group','direct']`, `USER_KINDS`,
   `LOCALES = ['ja','en']`, `MODERATION_MODES = ['flag','block']`,
   `MODERATION_STATUSES`, `REPORT_REASONS = ['unkind','scary',
   'inappropriate','other']`, `AVATAR_PRESETS` (≥ 12 slugs, e.g. `bear-01`…),
   text length caps (`MESSAGE_MAX_CHARS = 4000`, `CAPTION_MAX = 500`,
   `POST_TITLE_MAX = 100`, `POST_BODY_MAX = 8000`, `COMMENT_MAX = 2000`,
   `DISPLAY_NAME_MAX = 30`) — all as `const` objects with derived types.
5. `src/limits.ts`: the DESIGN §19.5 table as named constants, e.g.
   `RATE_LIMITS.authLogin = { max: 5, windowSec: 900, scope: 'ip' }` etc.,
   plus `UPLOAD_MAX_BYTES_DEFAULT = 10 * 1024 * 1024`,
   `SPACE_MEDIA_QUOTA_BYTES_DEFAULT = 2 * 1024 ** 3`,
   `INVITE_TTL_HOURS = 72`, `CHILD_LINK_CODE_TTL_MIN = 10`,
   session TTLs (web 30d / mobile 90d / device_link 180d sliding).
6. `src/audit.ts`: `AUDIT_ACTIONS` string-literal union exactly per DESIGN
   §13.8, plus `GUARDIAN_VISIBLE_ACTIONS` computed by an explicit rule —
   all actions EXCEPT those prefixed `auth.`, `admin.`, or `session.`, and
   except instance-level actions (`space.purged` when added) — exported as
   a concrete readonly array with a unit test asserting the rule and the
   array agree (so adding an action forces a conscious visibility choice).
7. Also create the `packages/i18n` (`@famchat/i18n`) **skeleton** in this
   issue (it is a shared cross-boundary asset like this package):
   `locales/{ja,en}/{common,auth,chat,board,guardian,safety,
   notifications,errors}.json` (all present, may be near-empty), a typed
   `resources` export, and a parity unit test (ja/en key sets equal).
   Issue 06 seeds `auth.json` email keys; 20 adds web tooling; 40 adds
   the full completeness gate.
8. `src/ids.ts`: `newId()` built on `monotonicFactory()` from `ulid` (module-
   level factory instance) so same-millisecond calls remain strictly
   lexicographically increasing, and `isId(s)` validator (26-char Crockford
   base32 regex).
9. Unit tests: env loader failure lists all bad keys; env-matrix
   completeness (req 2); error codes unique; limits table matches DESIGN
   §19.5 values (hardcoded expectations); `newId()` strictly increasing
   across 1000 tight-loop calls (valid because of the monotonic factory).

## Acceptance Criteria

- [ ] `pnpm --filter @famchat/shared test|typecheck|build` all pass.
- [ ] Every constant listed above exists with the exact names/values.
- [ ] Env loader exits non-zero listing all problems when given empty env.
- [ ] No runtime dependency besides `zod` and `ulid`.

## Validation

```bash
pnpm --filter @famchat/shared typecheck && pnpm --filter @famchat/shared build
pnpm --filter @famchat/shared test
```

## Dependencies

01.

## Non-goals

Authz matrix (10), ws payloads (15), quiet-hours schema (29), API schemas.

## Design References

- DESIGN §3.4 (env matrix), §4 (conventions), §8.3 (error model), §13.8
  (audit catalog), §19.5 (limits)
