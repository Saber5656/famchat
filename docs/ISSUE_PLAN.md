# famchat — v1 Issue Plan

Status: Draft for review (canonical roadmap; GitHub Issues are derived from
this file and `docs/issues/*.md`)
Last updated: 2026-07-11
Design source: `docs/DESIGN.md` (referenced below as DESIGN §N)

## 1. v1 completion statement

**v1 is complete when all 57 issues below are implemented and validated.**
At that point the following is true, with no additional undocumented work
except newly discovered implementation unknowns (§8):

> A technical operator can deploy famchat to a single VPS with Docker Compose
> and TLS; issue an instance invite; a guardian can create a family space,
> invite adults/guardians, create child accounts, and link child devices by
> QR; the family can chat (text + images with EXIF/GPS stripped) in family/
> group/direct rooms with read receipts and realtime delivery, and use a
> pinned bulletin board — from the Japanese/English web PWA and the Expo
> mobile app distributed via TestFlight/Play internal testing, with web and
> mobile push notifications; NG-word flagging, reporting, guardian console,
> quiet hours, and audit logging protect children per DESIGN §13; the
> operator can administer the instance via CLI, take tested backups, restore
> them, export/delete spaces; CI enforces lint/type/tests/i18n parity; the
> security hardening checklist (DESIGN §19) passes; self-host and closed-beta
> legal docs exist in ja+en; and the repository is ready to be made public
> once the owner decides the license (ADR-009).

## 2. Issue list in recommended execution order

Effort: S ≈ half day, M ≈ 1 day, L ≈ 2 days for one focused implementation
agent. `Deps` = issue numbers that must be completed first.

| # | File | Title | Effort | Deps |
|---|---|---|---|---|
| 01 | `01-monorepo-scaffold-ci-baseline.md` | Monorepo scaffold + CI baseline | M | — |
| 02 | `02-dev-environment-docker-compose.md` | Dev dependencies via Docker Compose | S | 01 |
| 03 | `03-shared-core-env-errors-constants.md` | `@famchat/shared`: env, errors, constants, limits | M | 01 |
| 04 | `04-db-prisma-baseline-identity-schema.md` | `@famchat/db`: Prisma baseline (identity/tenancy schema) | M | 02,03 |
| 05 | `05-api-server-skeleton.md` | `apps/api` Fastify + tRPC skeleton | M | 03,04 |
| 06 | `06-guardian-auth-sessions.md` | Guardian/adult auth + sessions + password reset | L | 05 |
| 07 | `07-instance-gate-and-space-creation.md` | Instance invite gate + space creation | M | 06 |
| 08 | `08-space-invites-adults.md` | Space invites (adult/guardian) | M | 07 |
| 09 | `09-child-accounts-and-device-link.md` | Child accounts + QR device link | L | 07 |
| 10 | `10-authz-permission-matrix-middleware.md` | Permission matrix + tRPC authz middleware | M | 07,08,09 |
| 11 | `11-audit-log-foundation.md` | Audit log foundation | M | 06,07,08,09,10 |
| 12 | `12-rate-limits-and-http-hardening-baseline.md` | Rate limits + HTTP hardening baseline | M | 06,08,09 |
| 13 | `13-rooms-model-and-api.md` | Rooms model + API (incl. guardian oversight rule) | L | 09,10,11 |
| 14 | `14-messages-model-and-api.md` | Messages model + API | L | 13 |
| 15 | `15-realtime-socketio.md` | Realtime socket.io server + event fanout | L | 14 |
| 16 | `16-read-receipts.md` | Read receipts + unread counts | M | 15 |
| 17 | `17-attachments-upload-api.md` | Attachment upload API (presigned, quarantine) | M | 12,14 |
| 18 | `18-image-worker-pipeline.md` | `apps/worker` + image processing pipeline | L | 17 |
| 19 | `19-board-backend.md` | Bulletin board backend | M | 13,15,18 |
| 20 | `20-web-skeleton-i18n-auth.md` | `apps/web` skeleton: i18n, auth pages, child link page | L | 06,08,09 |
| 21 | `21-web-spaces-rooms-shell.md` | Web shell: space switcher, room list, unread | M | 16,20 |
| 22 | `22-web-chat-room-ui.md` | Web chat room UI (list, composer, WS, receipts) | L | 21 |
| 23 | `23-web-image-share-ui.md` | Web image share UI | M | 18,22 |
| 24 | `24-web-board-ui.md` | Web bulletin board UI | M | 19,21,23 |
| 25 | `25-web-user-settings-ui.md` | Web user settings (profile, locale, sessions) | M | 20 |
| 26 | `26-moderation-package.md` | `@famchat/moderation`: normalize + matcher + base lists | L | 03 |
| 27 | `27-moderation-pipeline-integration.md` | Moderation pipeline on all content writes | L | 14,19,26 |
| 28 | `28-reports-backend.md` | Reports backend | M | 11,13,14,19 |
| 29 | `29-quiet-hours-backend.md` | Quiet hours backend + enforcement | L | 09,14,16,17,19,25 |
| 30 | `30-guardian-console-backend.md` | Guardian console backend (overview, removals, settings) | L | 27,28,29 |
| 31 | `31-web-guardian-dashboard-ui.md` | Web guardian dashboard (flags, reports, children, audit) | L | 22,24,30 |
| 32 | `32-web-space-admin-ui.md` | Web space admin UI (members, invites, children, settings) | L | 30,21 |
| 33 | `33-web-report-flow-ui.md` | Web report flow UI (child-friendly) | S | 22,24,28,31 |
| 34 | `34-web-quiet-hours-ux.md` | Web quiet hours UX (lock screen + editor) | M | 25,29,31 |
| 35 | `35-operator-admin-api-and-cli.md` | Operator admin REST API + `pnpm ops` CLI | M | 07,11,12 |
| 36 | `36-data-lifecycle-deletion-export.md` | Data lifecycle: deletion, purge, export | L | 18,30,35 |
| 37 | `37-notification-framework-inapp.md` | Notification framework + in-app feed (worker fanout) | L | 15,18,29,35,36 |
| 38 | `38-web-push-and-pwa.md` | Web Push (VAPID) + PWA installability | L | 20,25,37 |
| 39 | `39-web-notifications-ui.md` | Web notifications UI (feed, badges, prefs) | M | 21,22,24,31,37 |
| 40 | `40-i18n-completeness-gate.md` | i18n completeness gate + tooling | M | 20,24,25,31,32,33,34,38,39 |
| 41 | `41-mobile-scaffold-i18n.md` | `apps/mobile` Expo scaffold + i18n | M | 03,20 |
| 42 | `42-mobile-auth-and-device-link.md` | Mobile auth: login, QR link, PIN lock | L | 41,09 |
| 43 | `43-mobile-rooms-and-chat.md` | Mobile rooms + chat (WS, receipts) | L | 16,22,42 |
| 44 | `44-mobile-image-share.md` | Mobile image share | M | 43,18 |
| 45 | `45-mobile-board.md` | Mobile bulletin board | M | 19,43,44 |
| 46 | `46-mobile-push-and-deeplinks.md` | Mobile push (Expo) + deep links | L | 37,39,43 |
| 47 | `47-mobile-safety-surfaces.md` | Mobile safety surfaces (lock, report, guardian tab) | L | 28,29,30,33,34,43,45,46 |
| 48 | `48-eas-internal-distribution.md` | EAS internal distribution (TestFlight/Play internal) | M | 46,47 |
| 49 | `49-production-compose-and-caddy.md` | Production compose + Caddy TLS + migrate gate | L | 18,35,38 |
| 50 | `50-backup-and-restore.md` | Backup + tested restore | M | 49 |
| 51 | `51-ci-cd-full-pipeline.md` | Full CI/CD (integration tests, GHCR publish) | L | 49 |
| 52 | `52-seed-and-demo-data.md` | Seed + demo data | M | 19,27,28,29 |
| 53 | `53-e2e-playwright-suite.md` | Web E2E suite (golden paths) | L | 23,24,32,33,34,36,38,39,49,51,52 |
| 54 | `54-security-hardening-gate.md` | Security hardening gate (DESIGN §19 checklist) | L | 49,51,53 |
| 55 | `55-selfhost-and-operator-docs.md` | Self-host + operator docs (ja/en) | M | 35,50,51,52 |
| 56 | `56-beta-legal-docs.md` | Closed-beta ToS/Privacy templates + consent flow | M | 07,20,36,53,55 |
| 57 | `57-oss-release-prep.md` | OSS release prep (license, SECURITY.md, scans) | M | 51,54,55,56, ADR-009 decision |

## 3. Implementation waves

| Wave | Issues | Theme | Exit criterion |
|---|---|---|---|
| 0 Foundation | 01–12 | Repo, dev env, identity, auth, tenancy, authz, audit, hardening baseline | API integration tests: signup→space→invite→child link all green |
| 1 Messaging backend | 13–19 | Rooms, messages, realtime, receipts, media, board | Two test users chat via API+WS with images; board CRUD |
| 2 Web core | 20–25 | Web PWA shell, chat, images, board, settings | Family can fully use famchat from browsers (dev env) |
| 3 Safety | 26–36 | Moderation, reports, quiet hours, guardian console, operator admin, lifecycle | DESIGN §13 demo-able end-to-end on web |
| 4 Notifications & i18n | 37–40 | In-app + web push, notification UI, i18n gate | Push received on desktop+Android web; CI i18n gate on |
| 5 Mobile | 41–48 | Expo app feature parity + internal distribution | Beta family installs via TestFlight/Play internal and uses all core flows |
| 6 Ops & release | 49–57 | Production deploy, backup, CI/CD, E2E, security gate, docs, legal, OSS prep | Closed beta live on VPS; security checklist passed; repo publishable |

Waves are sequential; issues inside a wave may run in parallel when their
`Deps` allow. The dependency table (§2) is authoritative over wave grouping.

## 4. Dependency notes (cross-wave edges worth calling out)

- 20 (web skeleton) needs only wave-0 auth issues — web work can start while
  wave-1 backend continues in parallel.
- 26 (moderation package) depends only on 03 — can be built any time early.
- 37 (notifications) deliberately follows 29 (quiet hours) because child
  suppression is part of the fanout contract (DESIGN §14.2).
- 49 (production compose) precedes 51 (CI/CD) because the pipeline publishes
  the images the compose file pins.
- 57 (OSS release prep) is blocked on the **owner's license decision**
  (ADR-009) — flagged as a human gate, not an agent task.

## 5. Coverage table: DESIGN.md sections → issues

| DESIGN § | Topic | Covered by issues |
|---|---|---|
| §1–2 | Product scope | all (scope guard: ISSUE_PLAN §1 statement) |
| §3 | Architecture, env matrix | 01,02,03,05,49 |
| §4 | Repo layout, conventions | 01,03 |
| §5 | Tenancy, accounts, roles | 04,07,08,09 |
| §6 | Auth & sessions | 06,09,12,20,42 |
| §7.1 | Identity schema | 04 |
| §7.2 | Messaging schema | 13,14,16 |
| §7.3 | Board schema | 19 |
| §7.4 | Media schema | 17,18 |
| §7.5 | Safety schema | 11,26,27,28,36 |
| §7.6 | Notification schema | 37,38,46 |
| §8 | API design, error model | 05,06–19 (per-feature), 10 |
| §9 | Realtime | 15,16,22,43 |
| §10 | Messaging rules | 13,14,22,43 |
| §11 | Board | 19,24,45 |
| §12 | Media pipeline | 17,18,23,44 |
| §13.1 | Permission matrix | 10 (+ every feature issue enforces) |
| §13.2–13.3 | Oversight + transparency | 13,30,31,32,22,47 |
| §13.4 | NG filter | 26,27 |
| §13.5 | Quiet hours | 29,34,47 |
| §13.6 | Reports | 28,33,47 |
| §13.7 | Operator admin | 35 |
| §13.8 | Audit catalog | 11 (+ producers in feature issues) |
| §14 | Notifications | 37,38,39,46 |
| §15 | i18n | 20,40,41 (+ every UI issue ships ja+en) |
| §16 | Web app | 20–25,31–34,38,39 |
| §17 | Mobile app | 41–48 |
| §18 | Deployment & ops | 02,49,50,51,55 |
| §19 | Security model | 12,17,18,35,51,54,57 (+ security AC in every issue) |
| §20 | Observability | 05,35,49 |
| §21 | Testing strategy | 01,51,52,53 (+ Validation in every issue) |
| §22 | Compliance posture | 56,36 |
| §23 | Glossary/conventions | 01,03 |

## 6. Validation strategy across the product

1. **Per-issue gates**: every issue's Validation section lists exact commands
   (`pnpm -w typecheck && pnpm -w lint && pnpm -w test`, plus issue-specific
   integration/E2E steps). An issue is not done until they pass.
2. **Wave exit criteria** (§3) are demonstrable behaviors, not code review
   opinions.
3. **Continuous CI** (01 baseline → 51 full): lint, typecheck, unit,
   API integration against real Postgres/Redis/MinIO, i18n parity (40),
   gitleaks + dependency audit (51,54).
4. **E2E golden paths** (53) executed against the production-shaped compose
   stack: onboard → invite → child link → chat → image → NG flag → report →
   quiet hours → board → notifications → lifecycle (deletion/export).
5. **Security gate** (54): DESIGN §19 tables converted into a pass/fail
   checklist with evidence links; release-blocking.
6. **Restore drill** (50): backup restored onto scratch stack; data verified.
7. **Mobile smoke checklist** (48): manual runbook per internal build.
8. **Cross-tenant denial tests** (10, 13, 51): every space-scoped router has
   at least one "member of another space gets PERMISSION_DENIED/NOT_A_MEMBER"
   integration test — the multi-tenant safety invariant.

## 7. Deferred to v2 (explicitly out of all v1 issues)

Public signup + abuse program; billing; app-store publication (Kids Category
/ Families compliance); message editing; reactions/stickers; search; typing/
presence; voice messages; link previews; GIF/animated media; avatar uploads;
family calendar; AI-assisted moderation; email notification channel; TOTP
2FA; passkeys (guardians); admin web console; Prometheus/Grafana; Maestro
mobile E2E; multi-instance horizontal scaling; COPPA/GDPR programs;
additional locales.

## 8. Known unknowns (may create additional issues during implementation)

| # | Unknown | Trigger to resolve | Likely artifact |
|---|---|---|---|
| U1 | License choice (ADR-009) | Owner decision before 57 | LICENSE + README section |
| U2 | Expo push behavior in TestFlight/internal builds (credentials, quirks) | During 46/48 | Possible extra issue on push credential runbook |
| U3 | iOS Web Push reliability in practice | During 38 | Doc caveats; possibly steer-to-app banner issue |
| U4 | sharp HEIC decode support in chosen base image | During 18 | Either libvips build tweak or HEIC→client-side conversion issue |
| U5 | Japanese NG base list quality / false-positive rate | Beta feedback after 27 | List curation issues (data-only) |
| U6 | VPS sizing under real family media load | After 49 | compose resource tuning; maybe image CDN-ish cache issue |
| U7 | Prisma + ULID ordering edge cases (clock skew on same-ms inserts) | During 14 | Possibly per-room seq column issue |
| U8 | socket.io behind Caddy websocket timeouts | During 49 | Caddyfile keepalive tuning |
| U9 | Guardian UX for multi-space children (v2 shared custody) | v2 | ADR + design update |
