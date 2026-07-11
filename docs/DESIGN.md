# famchat — v1 Design Document

Status: Draft for review (canonical source of truth)
Last updated: 2026-07-11
Owner: Saber5656
Authored by: Fable (requirements/design agent), from owner-confirmed requirements

This document is the canonical design for famchat v1. Implementation issues in
`docs/issues/*.md` are derived from this document via `docs/ISSUE_PLAN.md`.
If this document and any GitHub Issue disagree, this document wins and the
issue is stale.

---

## 1. Product Overview

**famchat** is a child-safety-first, family-only chat and bulletin board.

One sentence: a private messenger that a guardian sets up for their own
family, where children aged 6–12 can chat safely because the network is
structurally closed, guardians have full oversight, and content protections
and time rules are built in — not bolted on.

### 1.1 Problems it solves

| Problem | How famchat addresses it |
|---|---|
| General messengers (LINE, WhatsApp) expose children to strangers, spam, and open contact graphs | Closed network: accounts exist only inside a family space; there is no way to contact or be contacted by anyone outside it |
| Guardians cannot see what children do in other apps | Guardian role has structural oversight of all rooms a child participates in |
| Kids' first phone/tablet needs usage boundaries | Quiet hours (time rules) enforced server-side per child |
| Trouble (bullying between siblings/cousins, inappropriate words) goes unnoticed | NG-word detection, reporting flow, and guardian review queue |
| Families distrust big-tech data handling | Open-source, self-hostable, single-purpose; operator access is minimized and audited |

### 1.2 Personas

| Persona | Description | Primary devices |
|---|---|---|
| **Guardian** (保護者) | Parent who creates and administers the family space. Tech level: ordinary smartphone user (beta operator is technical). | Smartphone (mobile app), occasionally PC web |
| **Child** (子ども, 6–12) | Elementary-school child. May not read all kanji; no email address; uses a hand-me-down phone or tablet. | Tablet / phone (mobile app or PWA) |
| **Adult member** (おとな) | Grandparent or other trusted relative invited by a guardian. May join multiple family spaces (both children's families). | Smartphone, PC web |
| **Operator** (運営者) | Person running the SaaS instance (in closed beta: the owner). Also the self-host administrator persona for OSS users. | CLI / terminal |

### 1.3 Core use cases (v1)

1. Guardian creates a family space with an instance invite code, invites the other guardian and grandparents, and creates child accounts.
2. Child links their device by scanning a QR code shown on the guardian's device; no email or password needed.
3. Family chats in the default "family" room; siblings have their own room; a grandparent has a 1:1 room with a grandchild — all child rooms visible to guardians.
4. Family shares photos; images are re-encoded and GPS metadata is stripped automatically.
5. Guardian posts "swimming lesson moved to Saturday" on the bulletin board and pins it; members comment.
6. A message containing an NG word is flagged; guardians get a notification and review it in the guardian console.
7. A child reports a message that made them uncomfortable; guardians see the report queue.
8. At 21:00 the child's quiet hours start: the app locks for the child until 07:00.
9. Members get push notifications (web push / Expo push) respecting per-room preferences.
10. Everything works in Japanese and English UI; child-facing Japanese uses furigana-friendly plain wording.

### 1.4 Product principles

1. **Safety is structural, not moderational.** The strongest protections are the ones with no toggle: closed network, EXIF stripping, guardian oversight of child rooms.
2. **Guardians are the administrators, not surveillance.** Oversight is transparent: children's UI states clearly that guardians can see everything ("おうちの人が見られます"). No covert monitoring features.
3. **Server-readable by design.** Guardian oversight, NG-word filtering, and search-ability require plaintext on the server. famchat deliberately does not use E2EE (ADR-003); compensating controls: TLS everywhere, encryption at rest, minimal audited operator access.
4. **Self-host = SaaS.** The Docker Compose stack that OSS users self-host is the same artifact set the closed beta runs on. No hidden operator-only components.
5. **Weak-agent implementable.** Every design choice favors explicit schemas, mainstream libraries, and mechanical implementation over cleverness.

---

## 2. Scope

### 2.1 v1 scope (must all ship)

- Multi-tenant data model (family spaces, multi-space membership), invite-code-gated closed beta.
- Roles: guardian / adult / child, with the permission matrix in §13.1.
- Auth: guardian email+password; child QR device-link; DB-backed sessions; password reset via email.
- Chat: family/group/direct rooms, text + single-image messages, soft delete, read receipts, pagination, realtime delivery (WebSocket) with reconnect backfill.
- Bulletin board: posts (multi-image, pinning) + comments.
- Media: presigned upload, worker pipeline (re-encode, EXIF/GPS strip, thumbnails), signed serving.
- Safety: NG-word filter (ja/en built-in lists + per-space custom), guardian flag queue, reporting flow, guardian console, quiet hours, audit log.
- Notifications: in-app feed, Web Push (VAPID), Expo push; per-room prefs.
- i18n: Japanese + English full UI, i18next everywhere, furigana policy for child-facing ja strings.
- Web app (Next.js, installable PWA) and mobile app (Expo RN; TestFlight / Play internal distribution).
- Operator admin: token-guarded admin REST API + CLI wrapper (instance invites, suspend, stats, audit).
- Data lifecycle: space export, space/user/child deletion with grace period, purge jobs.
- Ops: production Docker Compose (Caddy TLS), backups + tested restore, CI (lint/type/test/build/publish), E2E suite, security hardening gate, seed/demo data, self-host guide, closed-beta ToS/privacy templates, OSS release prep.

### 2.2 v1 non-goals (explicitly out)

- E2E encryption (ADR-003; no planned migration path).
- Public signup, billing, app-store publication (Kids Category review), federation.
- Message editing (delete + resend instead), reactions/stickers, typing indicators, presence, search, threads, voice/video, link previews/unfurling, GIF/animated media, custom avatar uploads (preset avatar set only — no child face photos as avatars by default), calendar, AI moderation.
- Email as a notification channel (email is used only for guardian password reset).
- Horizontal scaling beyond one VPS (design is Redis-adapter-ready but v1 targets a single host).
- Full COPPA/GDPR programs (closed beta under Japanese law baseline; see §22).

### 2.3 v2 deferred (documented ideas, not designed here)

Reactions/stickers, search, message editing, typing/presence, store publication (Apple Kids Category / Google Families), public signup + abuse program, billing, E-mail digests, AI image/text moderation, voice messages, family calendar/events, avatar uploads with face-blur guidance, GDPR/COPPA program, multi-language beyond ja/en, admin web console, Maestro mobile E2E.

### 2.4 Known unknowns (may spawn issues during implementation)

Tracked in `docs/ISSUE_PLAN.md` §8. Highlights: license final decision (ADR-009 pending owner), Expo push behavior in internal distribution builds, iOS Web Push reliability, HEIC decode support in sharp build, Japanese NG base list quality, VPS sizing.

---

## 3. System Architecture

### 3.1 Component diagram

```
                    ┌──────────────────────────── VPS (Docker Compose) ───────────────────────────┐
                    │                                                                              │
 [Web browser] ──┐  │  ┌───────┐    ┌─────────────┐     ┌────────────┐    ┌─────────────────┐     │
 (Next.js PWA)   ├──┼──► Caddy ├────► web (Next.js)│     │ api        │    │ worker          │     │
                 │  │  │ :443  │    └─────────────┘     │ Fastify    │    │ BullMQ consumers │     │
 [Mobile app] ───┤  │  │ TLS   ├────────────────────────► tRPC+REST  │    │ image pipeline   │     │
 (Expo RN)       │  │  │       ├────────────────────────► socket.io  │    │ push/notif fanout│     │
                 │  │  └───────┘   (websocket upgrade)  └─┬───┬───┬──┘    └──┬───┬───┬────────┘     │
                 │  │                                     │   │   │          │   │   │              │
                 │  │             ┌───────────────────────┘   │   └───┐      │   │   └────────┐     │
                 │  │             ▼                           ▼       ▼      ▼   ▼            ▼     │
                 │  │      ┌────────────┐            ┌───────┐  ┌─────────────┐  ┌──────────────┐   │
                 │  │      │ PostgreSQL │            │ Redis │  │ MinIO / S3  │  │ SMTP relay   │   │
                 │  │      │  16        │            │ 7     │  │ (objects)   │  │ (ext., reset │   │
                 │  │      └────────────┘            └───────┘  └─────────────┘  │  mail only)  │   │
                 │  │                                                            └──────────────┘   │
                    └──────────────────────────────────────────────────────────────────────────────┘
 Push delivery (egress only): api/worker ──► Web Push endpoints (browser vendors), Expo Push API
```

### 3.2 Processes

| Process | Package | Responsibilities |
|---|---|---|
| `api` | `apps/api` | Fastify HTTP server: tRPC router (user-session APIs), REST (`/healthz`, `/readyz`, `/admin/v1/*`, `/media/*` redirects), socket.io server, auth, rate limiting |
| `worker` | `apps/worker` | BullMQ consumers: image processing, notification fanout (in-app row + web push + expo push), purge jobs, export jobs, nightly maintenance |
| `web` | `apps/web` | Next.js server (App Router) serving the web client / PWA |
| `mobile` | `apps/mobile` | Expo app (not a server; built via EAS) |
| `postgres`, `redis`, `minio`, `caddy` | infra | Datastores, queue/cache/pubsub, object storage, TLS reverse proxy |

### 3.3 Technology stack (versions = latest stable at implementation time; majors below are the floor)

| Layer | Choice | Rationale (details in ADR-001) |
|---|---|---|
| Language | TypeScript 5, strict | One language across server/web/mobile; shared types |
| Runtime | Node.js 22 LTS | LTS; Fastify/Prisma support |
| Package manager | pnpm ≥ 9, workspaces | Monorepo; no Turborepo in v1 (plain scripts) |
| HTTP server | Fastify 5 | Plugins for rate-limit/helmet; performance; first-class TS |
| RPC | tRPC 11 (fastify adapter) + zod | End-to-end types web/mobile without codegen; weak-agent-proof contracts |
| ORM | Prisma 6 + prisma-migrate | Mechanical migrations, mainstream docs |
| Realtime | socket.io 4 + @socket.io/redis-adapter | Reconnection/rooms built in; works on RN and web |
| Queue | BullMQ (Redis) | Simple, mainstream |
| DB | PostgreSQL 16 | Boring, reliable |
| Cache/queue/pubsub | Redis 7 | socket.io adapter + BullMQ + rate-limit store |
| Object storage | S3 API (MinIO self-host / any S3-compatible) | Presigned upload/serve |
| Images | sharp | Re-encode, resize, EXIF strip |
| Web | Next.js 15 App Router, Tailwind CSS, TanStack Query + tRPC client | Mainstream; PWA-capable |
| Mobile | Expo SDK (≥ 53), expo-router, TanStack Query + tRPC client | 1 codebase iOS/Android; EAS internal distribution |
| i18n | i18next (+react-i18next) on web, mobile, and server | One library everywhere; shared JSON catalogs in `packages/i18n` |
| Auth | Custom session auth: @node-rs/argon2, DB sessions | No auth SaaS; children have no email (ADR-008) |
| Push | web-push (VAPID), expo-server-sdk | Standard |
| Email | nodemailer → SMTP relay (Mailpit in dev) | Password reset only |
| Logging | pino (redacted) | Structured |
| Tests | vitest, Playwright, testcontainers-style compose services | See §21 |
| Lint/format | ESLint (typescript-eslint) + Prettier | Mainstream defaults |

### 3.4 Environment variable matrix (canonical; `packages/shared/src/env.ts` validates with zod)

| Variable | Used by | Required | Example / notes |
|---|---|---|---|
| `NODE_ENV` | all | yes | `development` / `production` / `test` |
| `DATABASE_URL` | api, worker | yes | `postgresql://famchat:***@postgres:5432/famchat` |
| `REDIS_URL` | api, worker | yes | `redis://redis:6379` |
| `S3_ENDPOINT` | api, worker | yes | `http://minio:9000` (path-style) |
| `S3_REGION` | api, worker | yes | `us-east-1` for MinIO |
| `S3_BUCKET` | api, worker | yes | `famchat` |
| `S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` | api, worker | yes | never logged |
| `S3_FORCE_PATH_STYLE` | api, worker | no (default true) | `true` for MinIO |
| `APP_BASE_URL` | api, worker | yes | public https origin of web, e.g. `https://beta.famchat.example` |
| `API_BASE_URL` | web (server), mobile build | yes | public https origin of api |
| `WEB_ORIGINS` | api | yes | comma-separated CORS allowlist |
| `SESSION_SECRET` | api | yes | ≥ 32 random bytes; cookie signing |
| `OPERATOR_TOKEN` | api | no | ≥ 32 random bytes when set; **unset ⇒ the `/admin/v1` surface does not exist (404)** — self-hosts may run without operator tooling |
| `SMTP_URL` | api | yes | `smtp://mailpit:1025` in dev |
| `MAIL_FROM` | api | yes | `famchat <noreply@…>` |
| `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY` | api, worker, web | yes (push) | generated once per instance |
| `VAPID_SUBJECT` | worker | yes (push) | `mailto:…` |
| `EXPO_ACCESS_TOKEN` | worker | no | only if Expo push enabled |
| `UPLOAD_MAX_BYTES` | api, worker | no (default 10485760) | 10 MiB |
| `LOG_LEVEL` | all | no (default info) | pino level |
| `DEFAULT_TIMEZONE` | api | no (default `Asia/Tokyo`) | new-space default |
| `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_WS_URL`, `NEXT_PUBLIC_VAPID_PUBLIC_KEY` | web | yes | build-time public config |
| `EXPO_PUBLIC_API_URL`, `EXPO_PUBLIC_WS_URL` | mobile | yes | build-time public config |

Secrets rule: secrets exist only in `.env` files on the host (never committed; `.env.example` documents shape with placeholders) and are injected via compose `env_file`. No secret has a default. See §19.6.

---

## 4. Repository Layout

```
famchat/
├── apps/
│   ├── api/          # Fastify + tRPC + socket.io server  (@famchat/api)
│   │   └── src/{index,server}.ts, src/routers/*, src/rest/*, src/ws/*, src/services/*
│   ├── worker/       # BullMQ consumers                    (@famchat/worker)
│   │   └── src/{index}.ts, src/jobs/*
│   ├── web/          # Next.js App Router                  (@famchat/web)
│   │   └── src/app/*, src/components/*, src/lib/*
│   └── mobile/       # Expo RN app                         (@famchat/mobile)
│       └── app/* (expo-router), src/components/*, src/lib/*
├── packages/
│   ├── shared/       # zod schemas, constants, error codes, authz matrix, env validation (@famchat/shared)
│   ├── db/           # Prisma schema + client + seed       (@famchat/db)
│   ├── moderation/   # normalization + matcher + base lists (@famchat/moderation)
│   └── i18n/         # message catalogs ja/en + helpers    (@famchat/i18n)
├── infra/
│   ├── compose.dev.yml        # postgres, redis, minio, mailpit (dev deps only)
│   ├── compose.prod.yml       # full stack incl. caddy, web, api, worker
│   └── caddy/Caddyfile
├── scripts/          # ops CLI (ops.mjs), backup/restore, seed helpers
├── docs/             # this file, ISSUE_PLAN.md, issues/, decisions/, research/, selfhost.md, legal/
├── .github/workflows/
└── package.json, pnpm-workspace.yaml, tsconfig.base.json, .nvmrc, .env.example
```

Conventions:

- Workspace packages are `@famchat/<name>`; internal imports only via package names, never relative `../../..` across packages.
- `apps/api` exports `type AppRouter` from its package entry (`types` export); web/mobile import it **type-only**.
- IDs are ULIDs (26-char Crockford base32, sortable) generated server-side via `ulid`; all tables use `id TEXT PRIMARY KEY` ULID unless noted.
- Timestamps are `TIMESTAMPTZ`, UTC in DB; display in space timezone.
- All zod schemas that cross a boundary (API input/output, env, quiet-hours JSON, WS payloads) live in `packages/shared` and are the single source for both validation and TS types.

---

## 5. Tenancy, Accounts, Roles

### 5.1 Model

- A **space** is one family tenant. All content (rooms, messages, board, attachments, reports, audit) is space-scoped by `space_id`.
- A **user** is a person; `kind` = `adult` | `child`. Adults have email+password. Children have neither (device-link only).
- A **membership** links user↔space with a role: `guardian` | `adult` | `child`. A user may hold memberships in multiple spaces (Slack model, ADR-002). One membership per (space, user).
- Exactly one membership per space has `is_owner = true` (the founding guardian; transferable to another guardian in the same space). Owner-only powers: delete space, promote/demote guardians, transfer ownership, request export.
- A **child user belongs to exactly one space** in v1 (enforced at creation; the DB model allows more for v2 shared-custody scenarios). Adults may join many spaces.
- Every space always has ≥ 1 active guardian membership (enforced: last guardian cannot leave/demote; owner cannot be removed).

### 5.2 Space creation and joining flows

| Flow | Who | Mechanism |
|---|---|---|
| Create space | New or existing adult | Requires valid **instance invite code** (operator-issued, closed-beta gate). Creator signs up (email+password) if new, names the space, becomes owner guardian. Default "family" room auto-created. |
| Invite adult/guardian | Guardian | Creates a **space invite**: one-time code (also rendered as URL + QR), role pre-set to `adult` or `guardian`, expires in 72h. Recipient opens link → signs up or logs in → membership created. |
| Create child account | Guardian | Creates child user (display name, birth year, avatar preset) inside the space. No email. Then generates a **device link code** (6-digit + QR, 10-min expiry, one-time) which the child device consumes to obtain a long-lived device session. |
| Leave / remove | Guardian removes members; adults can leave; children are removed only by guardians | Membership status → `removed`; child users additionally deactivated if no memberships remain. |

### 5.3 Roles (summary; full matrix in §13.1)

| Role | Intent |
|---|---|
| `guardian` | Parent/administrator: manages members, child accounts, safety settings; sees all child-participating rooms; reviews flags/reports |
| `adult` | Trusted relative: chats, posts; no admin or oversight powers |
| `child` | Protected account: chats/posts within safety rules; every room they join includes guardian oversight; UI discloses this |

---

## 6. Authentication & Sessions

### 6.1 Guardian / adult auth

- Email + password. Password policy: 10+ chars, zxcvbn score ≥ 3 gate on registration; hashed with argon2id (`@node-rs/argon2`, defaults: m=19456 KiB, t=2, p=1).
- Registration is only reachable via invite flows (instance invite → space creation; space invite → join). No open signup endpoint exists.
- Password reset: email token (32 random bytes, sha256-stored, 30-min expiry, single-use). SMTP via nodemailer. Reset revokes all sessions of the user.
- Optional TOTP is v2.

### 6.2 Child auth (device link)

1. Guardian, in guardian console: "link device" for child → API creates `child_link_codes` row: 6-digit numeric code + QR payload `${APP_BASE_URL}/link#c=<code>&s=<spaceId>` (hash stored, 10-min expiry, one-time). The code rides in the URL **fragment**: any camera app can open the link, the fragment is never sent to servers (keeps codes out of web/proxy logs), the web `/link` page reads it client-side, and the universal link opens the mobile app when installed.
2. Child device (mobile app or web) enters code / scans QR → `auth.childLink` verifies → creates a **device session** (kind `device_link`, 180-day expiry, sliding) bound to the child user, and returns session token.
3. Guardian console lists child devices (session records) and can revoke any of them instantly.
4. Optional per-child **PIN** (4–6 digits): client-side app lock to deter siblings; verified locally against a server-fetched-at-unlock check; explicitly **not a security boundary** (revocation is). Guardians set/reset it.

### 6.3 Sessions

- DB table `sessions`; token = 32 random bytes, sent as `famchat_session` cookie (web) or `Authorization: Bearer` (mobile). Only `sha256(token)` stored.
- Cookie: `HttpOnly; Secure; SameSite=Lax; Path=/`, 30-day sliding expiry, rotated on login (fixation defense). Mobile bearer: 90-day sliding. Child device sessions: 180-day sliding.
- CSRF: any state-changing HTTP request that authenticates **via cookie** must carry header `x-famchat-csrf: 1` (custom header ⇒ CORS preflight ⇒ cross-origin blocked by strict `WEB_ORIGINS` allowlist); bearer-authenticated requests (mobile) are exempt — the bearer header itself cannot be attached cross-site. The web tRPC client sets the header globally. Documented in §19.4.
- Session context resolution: every authed request resolves `{ user, memberships }`; per-space procedures additionally require an active membership in the target space and attach `{ space, membership }`.
- Logout revokes the session row. "Log out everywhere" revokes all of a user's sessions.
- Auth-route rate limits: see §19.5 table.

---

## 7. Data Model (PostgreSQL, via Prisma)

Types: `id` = ULID text PK; `ts` = timestamptz; enums are Prisma enums (Postgres native). FKs `ON DELETE` noted where non-default. All space-scoped tables index `space_id`.

### 7.1 Identity & tenancy

**users**
| column | type | notes |
|---|---|---|
| id | id | |
| kind | enum `adult,child` | |
| email | citext null | unique when not null; adults only |
| password_hash | text null | argon2id; adults only |
| display_name | text | 1–30 chars; moderation-checked |
| avatar_preset | text | key into built-in avatar set, default `bear-01` |
| locale | enum `ja,en` | default `ja` |
| birth_year | int null | children only; for guardian reference and defaults |
| status | enum `active,suspended,deleted` | suspended = operator action |
| created_at / updated_at | ts | |

**sessions**: id, user_id FK cascade, kind enum `web,mobile,device_link`, token_hash text unique, created_at, expires_at, last_used_at, ip inet null, user_agent text null, revoked_at ts null. Index (user_id), (expires_at).

**password_resets**: id, user_id FK cascade, token_hash unique, expires_at, used_at null, created_at.

**instance_invites**: id, code_hash unique, note text, max_uses int default 1, used_count int default 0, expires_at null, revoked_at null, created_at. (Operator-created.)

**spaces**: id, name text 1–40, timezone text (IANA, default env `DEFAULT_TIMEZONE`), default_locale enum `ja,en`, moderation_mode enum `flag,block` default `flag`, ng_builtin_ja bool default true, ng_builtin_en bool default true, status enum `active,suspended,pending_deletion,deleted`, delete_after ts null, created_by FK users, created_at / updated_at.

**memberships**: id, space_id FK cascade, user_id FK, role enum `guardian,adult,child`, is_owner bool default false, status enum `active,removed`, board_notify enum `all,none` default `all` (column added by issue 19's migration), created_at / updated_at. Unique (space_id, user_id). Partial unique index: one `is_owner=true AND status='active'` per space.

**space_invites**: id, space_id FK cascade, code_hash unique, role enum `guardian,adult`, created_by FK users, expires_at (≤ 72h), max_uses int default 1, used_count int default 0, revoked_at null, created_at.

**child_link_codes**: id, space_id FK cascade, child_user_id FK users, code_hash unique, expires_at (10 min), used_at null, created_by FK users, created_at.

**child_settings**: membership_id PK/FK cascade, pin_hash text null, quiet_hours jsonb null (schema §13.5), updated_by FK users, updated_at.

### 7.2 Messaging

**rooms**: id, space_id FK cascade, type enum `family,group,direct`, name text null (null for family/direct; derived display), direct_key text null unique (sorted member-id pair `"<idA>:<idB>"`, set only for direct rooms — dedupe key), created_by, archived_at null, created_at. Exactly one `family` room per space (partial unique index).

**room_members**: room_id FK cascade + user_id FK (composite PK), added_by, notify enum `all,none` default `all`, last_read_message_id text null, joined_at. Index (user_id).

**messages**: id (ULID = ordering key), room_id FK cascade, sender_id FK users null (null for system), kind enum `text,image,system`, body text null (≤ 4000 chars), attachment_id FK null, moderation_status enum `clean,flagged,blocked` default `clean`, deleted_at null, deleted_by null, created_at. Index (room_id, id DESC).

Message invariants: `text` ⇒ body non-empty; `image` ⇒ attachment_id set, body optional caption ≤ 500; `system` ⇒ body is an i18n key + params JSON (rendered client-side); soft-deleted messages keep the row, body/attachment redacted at read time.

### 7.3 Bulletin board

**board_posts**: id, space_id FK cascade, author_id, title text 1–100, body text ≤ 8000, pinned_at ts null, moderation_status enum as above, deleted_at/by, created_at / updated_at.
**board_post_attachments**: post_id FK cascade + attachment_id FK (composite PK), position int. ≤ 4 images per post.
**board_comments**: id, post_id FK cascade, author_id, body text 1–2000, moderation_status, deleted_at/by, created_at.

### 7.4 Media

**attachments**: id, space_id FK cascade, uploader_id, kind enum `image`, original_key text (S3 key, quarantine prefix), serve_key null, thumb_key null, mime text, size_bytes int, width/height int null, status enum `pending,processing,ready,rejected,purged`, reject_reason text null, created_at / updated_at. S3 layout: `q/<spaceId>/<attachmentId>` (quarantine original), `m/<spaceId>/<attachmentId>/full.webp|thumb.webp` (servable). Originals deleted after successful processing.

### 7.5 Safety

**moderation_hits**: id, space_id FK cascade, content_type enum `message,board_post,board_comment,display_name`, content_id text null (**null when the content was blocked and never created**), matched_terms jsonb (array of {term, source: builtin_ja|builtin_en|custom}), action enum `flagged,blocked`, metadata jsonb null (e.g. sha256 of blocked attempted text — raw attempted text is never persisted), resolution enum `pending,approved,removed` default pending, reviewed_by null, reviewed_at null, created_at. Index (space_id, resolution).

**reports**: id, space_id FK cascade, reporter_id, target_type enum `message,board_post,board_comment,user`, target_id text, reason enum `unkind,scary,inappropriate,other`, note text ≤ 500 null, status enum `open,resolved,dismissed`, resolved_by/at null, resolution_note null, created_at. Index (space_id, status).

**custom_ng_words**: id, space_id FK cascade, term text 1–50, normalized_term text, added_by, created_at. Unique (space_id, normalized_term).

**audit_logs**: id, space_id text null (**no FK by design** — audit rows must survive space deletion until purge; null = instance-level), actor_kind enum `user,operator,system`, actor_id text null, action text (catalog constant, e.g. `member.remove`), target_type text null, target_id text null, metadata jsonb (allowlisted fields only — never raw codes/tokens/passwords), ip inet null, created_at. Append-only (no update/delete API). Space purge (§ issue 36) explicitly deletes the space's audit rows (family privacy wins) while writing an instance-level `space.purged` record. Index (space_id, created_at DESC).

**space_exports**: id, space_id FK cascade, requested_by, status enum `pending,running,ready,failed,expired`, file_key null, expires_at null, created_at / updated_at.

### 7.6 Notifications

**notifications**: id, user_id FK cascade, space_id null, type text (catalog §14.2), payload jsonb (i18n params), link text (app route), read_at null, created_at. Index (user_id, read_at, created_at DESC).

**push_subscriptions**: id, user_id FK cascade, kind enum `webpush,expo`, endpoint_or_token text, keys jsonb null (webpush p256dh/auth), created_at, last_success_at null, disabled_at null, failure_count int default 0. Unique (kind, endpoint_or_token).

### 7.7 Read model notes

- Ordering/pagination: cursor = message `id` (ULID); `list(roomId, before?: id, limit ≤ 50)` returns descending.
- Unread count per room = `COUNT(messages.id > room_members.last_read_message_id AND sender != me AND deleted_at IS NULL)`; computed per request with the (room_id, id) index; acceptable at family scale (≤ 30 members/space).
- Receipts display: for each message, `read_by` derived from room_members' last_read pointers (no per-message receipt rows).

---

## 8. API Design

### 8.1 Surfaces

| Surface | Auth | Purpose |
|---|---|---|
| tRPC at `/trpc` | Session (cookie or bearer) | All user-facing app APIs |
| REST `/healthz`, `/readyz` | none | Liveness (process up) / readiness (DB+Redis+S3 ping) |
| REST `/admin/v1/*` | `Authorization: Bearer <OPERATOR_TOKEN>` | Operator admin (§13.7); never exposed to browsers (no CORS) |
| REST `/media/:attachmentId/:variant` | Session | Authorizes membership then 302 → presigned S3 GET (60 s) |
| socket.io at `/ws` | Session during handshake | Realtime events (§9) |

### 8.2 tRPC router map (procedures; `q` = query, `m` = mutation)

Namespaces and procedures (input/output zod schemas live in `packages/shared/src/api/*.ts`; issue drafts pin the exact shapes):

- `auth`: `m register` (invite-context only), `m login`, `m logout`, `m logoutAll`, `m requestPasswordReset`, `m resetPassword`, `m childLink` (code → device session), `q me`.
- `spaces`: `m create` (instance invite code), `q list` (my spaces + role), `q get`, `m updateSettings` (name/timezone/locale/moderation_mode/ng toggles), `m requestDeletion`, `m cancelDeletion`, `m transferOwnership`.
- `invites`: `m createSpaceInvite`, `q listSpaceInvites`, `m revokeSpaceInvite`, `m acceptSpaceInvite`, `q previewInvite` (code → space name/role, pre-auth).
- `children`: `m create`, `q list`, `m update` (name/avatar/birth_year), `m createLinkCode`, `q listDevices`, `m revokeDevice`, `m setPin`, `m remove`.
- `members`: `q list`, `m remove`, `m changeRole` (owner only, guardian↔adult), `m leave`.
- `rooms`: `q list` (with unread counts + last message), `m createGroup`, `m createDirect`, `m archive`, `m updateMembers`, `m setNotify`, `q get`.
- `messages`: `q list` (cursor), `m sendText`, `m sendImage` (attachmentId), `m delete`, `q around` (deep-link context).
- `receipts`: `m markRead` (roomId, messageId), `q roomReaders` (per-message read-by derivation).
- `attachments`: `m requestUpload` (mime,size → {attachmentId, presigned PUT}), `m finalize` (enqueues processing), `q get` (status).
- `board`: `q listPosts` (pinned first, cursor), `q getPost`, `m createPost`, `m updatePost` (author, 15-min window), `m deletePost`, `m pinPost`/`m unpinPost`, `m createComment`, `m deleteComment`, `q listComments`.
- `moderation`: `q flagQueue` (guardian), `m resolveHit` (approve/remove), `q listCustomWords`, `m addCustomWord`, `m removeCustomWord`.
- `reports`: `m create`, `q queue` (guardian), `m resolve`, `m dismiss`.
- `guardian`: `q dashboard` (space-level counts + per-child mini-status), `q childOverview` (rooms, devices, recent flags/reports per child), `m setQuietHours`, `q auditLog` (space-scoped, paged). (Message removal deliberately has no guardian alias — `messages.delete` with `message.deleteAny` is the single delete path.)
- `notifications`: `q feed` (cursor), `m markRead`, `m markAllRead`, `m registerPush` (webpush sub / expo token), `m unregisterPush`, `q unreadCounts` (notif + per-room).
- `exports`: `m request` (owner), `q status`, `q downloadUrl` (302-style presigned).

### 8.3 Error model

`packages/shared/src/errors.ts` defines `FamchatErrorCode` string union; the wire shape is: tRPC error `code` (transport class, e.g. `FORBIDDEN`) + `message` + `data: { cause: FamchatErrorCode, details?: Record<string, unknown> }` — clients branch on `error.data.cause`. Canonical codes (non-exhaustive): `AUTH_INVALID_CREDENTIALS`, `AUTH_SESSION_EXPIRED`, `INVITE_INVALID_OR_EXPIRED`, `PERMISSION_DENIED`, `NOT_A_MEMBER`, `ROOM_ARCHIVED`, `QUIET_HOURS_ACTIVE` (includes `until` timestamp), `CONTENT_BLOCKED_NG_WORD` (includes matched category count, never the matched terms), `UPLOAD_TOO_LARGE`, `UPLOAD_TYPE_UNSUPPORTED`, `ATTACHMENT_NOT_READY`, `RATE_LIMITED`, `LAST_GUARDIAN`, `VALIDATION_FAILED`. Client i18n maps codes → localized messages; server never localizes errors.

### 8.4 Conventions

- All list endpoints: cursor pagination `{ cursor?: string, limit?: number≤50 }` → `{ items, nextCursor? }`.
- All mutations idempotent where feasible (`sendText` accepts a client-generated `dedupeId` ULID; uniqueness of (sender, dedupeId) is permanent — client ULIDs make accidental reuse practically impossible, so no expiry window is needed).
- Server clock is authoritative; clients never write timestamps.
- Every mutation runs, in this exact order: session → membership/permission check (§13.1 matrix as code) → zod input → object-state checks (e.g. room archived, attachment ready) → quiet-hours gate (children) → moderation pipeline (content types) → write + audit (privileged actions) → post-commit WS emit + notification enqueue.

---

## 9. Realtime (socket.io)

### 9.1 Connection & auth

- Endpoint `/ws` on the api process. Handshake auth: web sends the session cookie automatically; mobile passes `auth: { token }`. Invalid/expired session ⇒ connection refused (`UNAUTHORIZED`).
- On connect, the server joins the socket to channels: `user:<userId>` and, for each active membership, `space:<spaceId>`; room-level fanout uses `room:<roomId>` joined lazily via `subscribeRoom` ack event (client subscribes to visible rooms).
- Children under active quiet hours: connection allowed but server emits `quietHours.state` and refuses room subscriptions; message events are not delivered to them until quiet hours end (client shows lock screen).

### 9.2 Event catalog (server → client)

| Event | Channel | Payload (zod in `packages/shared/src/ws.ts`) |
|---|---|---|
| `message.created` | room | `{ message: MessageDTO }` |
| `message.deleted` | room | `{ roomId, messageId }` |
| `message.moderated` | room (guardians) + sender | `{ roomId, messageId, moderationStatus }` |
| `receipt.updated` | room | `{ roomId, userId, lastReadMessageId }` |
| `room.updated` | space | `{ room: RoomDTO }` (create/archive/members) |
| `board.postCreated` / `board.commentCreated` | space | `{ postId }` / `{ postId, commentId }` |
| `notification.created` | user | `{ notification: NotificationDTO }` |
| `moderation.flagged` | space-guardians (`space:<id>:guardians`) | `{ hitId, contentType }` |
| `report.created` | space-guardians | `{ reportId }` |
| `quietHours.state` | user (child) | `{ active: boolean, until?: string }` |
| `member.updated` | space | `{ membership: MembershipDTO }` |
| `session.revoked` | user | `{}` — client must logout immediately |

Client → server: `subscribeRoom { roomId }` / `unsubscribeRoom` (ack with error code on permission failure). Everything else goes through tRPC (no business mutations over WS).

### 9.3 Delivery semantics

- WS is **best-effort notify**; the DB is the source of truth. On reconnect, client refetches: room list with unread counts, messages since last known id per open room, notification feed. No server-side event replay buffer in v1.
- Multi-instance readiness: all emits go through @socket.io/redis-adapter; v1 runs one api instance but nothing assumes it.

---

## 10. Messaging

### 10.1 Room rules

| Room type | Members | Creation | Naming |
|---|---|---|---|
| `family` | all active space members, auto-maintained | auto at space creation; cannot archive | localized "family" name client-side |
| `group` | chosen members (≥ 2) | guardian or adult; children cannot create | required 1–30 chars, moderation-checked |
| `direct` | exactly 2 members | any member incl. child (child→member of same space only) | derived from participants |

- Guardian oversight rule: guardians can open and read **any room that has ≥ 1 child member** without being a member (observer mode: read + delete powers, no notifications unless member). Adult-only rooms are private to members. This rule is enforced in room access checks, is displayed in child UI permanently ("おうちの人も 見られるよ"), and is listed in the space's member-facing safety description.
- A child cannot be in a room with zero guardians having oversight access — trivially true by the rule above (all child rooms are guardian-visible).
- Direct rooms are deduplicated per member pair (find-or-create).

### 10.2 Message lifecycle (state machine)

```
 client sendText/sendImage
   └─ validate (auth → membership → zod → room state/attachment ready → quiet hours)
        └─ moderation pipeline (§13.4)
             ├─ clean   → INSERT status=clean   → WS message.created → notify fanout
             ├─ flagged → INSERT status=flagged → WS message.created (all) +
             │            moderation_hits row → WS moderation.flagged (guardians) + guardian notification
             └─ blocked → (moderation_mode=block only) NO message row; moderation_hits row with
                          action=blocked; error CONTENT_BLOCKED_NG_WORD to sender; guardians notified
 delete (sender own / guardian any):
   visible|flagged → deleted (soft): body+attachment redacted in reads, WS message.deleted, audit log
 moderation resolve (guardian): flagged → approved (status→clean) | removed (soft delete)
```

Rendering rules: `flagged` messages render normally for everyone except guardians, who see a flag badge (child senders are not stigmatized in-room; review happens in console). Deleted messages render as localized tombstone ("メッセージは削除されました").

---

## 11. Bulletin Board

- One board per space (implicit; no boards table). Posts support title, body (plain text, newlines preserved), ≤ 4 images, pin/unpin (guardians; pinned sorted first by pinned_at DESC then created DESC).
- Comments: flat (no nesting), text only.
- Permissions per §13.1: all roles may post and comment by default (family notice-board semantics); guardians may delete any post/comment; authors may delete own; author may edit post body/title within 15 minutes of creation (no edit for comments; no edit indicator needed given the window).
- Moderation pipeline applies to title, body, and comments identically to messages.
- Board content emits `board.postCreated`/`board.commentCreated` and per-user notifications honoring per-room-style board preference (`spaces`-level board notify pref in v1: `memberships.board_notify enum all,none default all` — column on memberships).

---

## 12. Media & Attachments

### 12.1 Upload flow

```
client                          api                              worker                    S3
  │ attachments.requestUpload ──►│ validate mime∈{jpeg,png,webp,heic},
  │                              │ size ≤ UPLOAD_MAX_BYTES, quota (§19.5)
  │                              │ INSERT attachments(status=pending)
  │ ◄── {attachmentId, putUrl} ──│ presign PUT q/<space>/<id> (5-min, content-length-range)
  │ PUT bytes ─────────────────────────────────────────────────────────────────────────►│
  │ attachments.finalize ───────►│ status→processing, enqueue image job ───► consume
  │                              │                                  HEAD+GET original ◄──│
  │                              │                                  sniff magic bytes; decode via sharp;
  │                              │                                  auto-orient; strip ALL metadata (EXIF/GPS/XMP);
  │                              │                                  re-encode webp full (max 2048px) + thumb (512px);
  │                              │                                  PUT m/<space>/<id>/full.webp,thumb.webp ─────►│
  │                              │                                  status→ready (or rejected+reason);
  │                              │                                  DELETE original ────────────────────────────►│
  │ (poll attachments.get or send message referencing id; message send requires status=ready)
```

- Re-encoding is unconditional (defeats polyglot/steganographic-EXIF risks; guarantees GPS removal). Animated inputs are rejected in v1. Decode failures, dimension bombs (> 12k px side or > 80 MP), and mime/magic mismatches ⇒ `rejected`.
- Serving: only `m/` keys, only via `/media/:id/:variant` after membership check → 302 presigned GET (60 s). Bucket is private; no public ACLs; quarantine prefix never served.
- Attachments are single-use: one attachment belongs to one message or one board post (enforced on reference).

---

## 13. Safety System

### 13.1 Permission matrix (canonical; encoded in `packages/shared/src/authz.ts` as `PERMISSIONS` table; tRPC middleware `requirePermission(perm)` reads it)

| Capability (constant) | guardian | adult | child |
|---|:-:|:-:|:-:|
| `space.updateSettings` / `space.delete` / `space.transferOwner` | ✅ (delete/transfer: owner only) | ❌ | ❌ |
| `invite.adult.create` / `invite.guardian.create` / `invite.revoke` | ✅ | ❌ | ❌ |
| `child.create/update/linkDevice/revokeDevice/setPin/remove` | ✅ | ❌ | ❌ |
| `member.remove` / `member.changeRole` | ✅ (changeRole: owner) | ❌ | ❌ |
| `member.leave` | ✅ (not last guardian/owner) | ✅ | ❌ |
| `room.createGroup` | ✅ | ✅ | ❌ |
| `room.createDirect` | ✅ | ✅ | ✅ |
| `room.archive` / `room.updateMembers` | ✅ | own-created only | ❌ |
| `room.observe` (read child rooms w/o membership) | ✅ | ❌ | ❌ |
| `message.send` / `board.post` / `board.comment` | ✅ | ✅ | ✅ (quiet-hours gated) |
| `message.deleteOwn` / `board.deleteOwn` | ✅ | ✅ | ✅ |
| `message.deleteAny` / `board.deleteAny` | ✅ | ❌ | ❌ |
| `board.pin` | ✅ | ❌ | ❌ |
| `report.create` | ✅ | ✅ | ✅ |
| `report.review` / `moderation.review` / `moderation.customWords` | ✅ | ❌ | ❌ |
| `guardian.setQuietHours` / `guardian.viewAudit` / `guardian.childOverview` | ✅ | ❌ | ❌ |
| `export.request` | owner only | ❌ | ❌ |
| `notifications.managePush` (own) | ✅ | ✅ | ✅ |

Matrix is exhaustive in code: every tRPC mutation/query declares required permission(s); a unit test asserts every procedure is annotated (no unguarded procedure can ship).

### 13.2 Guardian oversight surfaces

Guardian console surfaces: per-child overview (devices, rooms, quiet hours, recent flags/reports involving the child), flag queue, report queue, member/invite management, space settings (moderation mode, NG toggles/custom words, timezone/locale), space-scoped audit log view. **Platform split in v1**: the web console provides everything; the mobile Guardian tab provides the monitoring + child-management subset (dashboard, queues, child overview incl. device link/revoke, quiet-hours editor) while member/invite management, space settings, and the audit view remain web-only (mobile parity is a v2 item).

### 13.3 Transparency to children

Child UI permanently shows an oversight notice in room headers and onboarding ("おうちの ひとも よめます / Your family grown-ups can read this"). Rendering of this notice is a hard acceptance criterion of the child UI issues — safety without covert surveillance.

### 13.4 NG-word filter pipeline (`packages/moderation`)

- **Normalization** (pure function `normalizeText`): Unicode NFKC → lowercase → katakana→hiragana fold → remove long-vowel marks/whitespace/zero-width/symbol separators between letters → basic leet fold (`0→o, 1→i, 3→e, 4→a, @→a, $→s`, plus letter fold `l→i` so `1`/`l` variants converge). Same function normalizes dictionary terms and inputs.
- **Matching**: Aho–Corasick automaton built per space (built-in ja list if `ng_builtin_ja`, built-in en list if `ng_builtin_en`, plus custom terms), cached in-process keyed by (spaceId, wordlist revision); substring match on normalized text.
- **Built-in lists**: `packages/moderation/lists/ja.txt`, `en.txt` — curated seed (~150–300 terms each: profanity, sexual, violence/self-harm, bullying phrases), maintained as reviewable plain text with category comments. Quality is a known unknown; lists are data, not code.
- **Actions**: space `moderation_mode = flag` (default): content is stored & delivered, `moderation_status=flagged`, hit recorded, guardians notified. `block`: content types `message|board_post|board_comment` are refused with `CONTENT_BLOCKED_NG_WORD`; hit recorded; guardians notified. Display names are always block-mode.
- Applies on create paths only (no retroactive scans in v1); custom word changes bump the space wordlist revision.

### 13.5 Quiet hours (time rules)

- Stored per child in `child_settings.quiet_hours` jsonb; zod schema:
  `{ enabled: boolean, rules: Array<{ days: (1..7)[], start: "HH:MM", end: "HH:MM" }> }` — `days` in ISO weekday numbering interpreted in the **space timezone**; windows may cross midnight (start > end ⇒ wraps to next day).
- Evaluation: pure function `isQuietNow(quietHours, spaceTz, now)` in `packages/shared`; unit-tested incl. midnight wrap and timezone edges (JST default; DST-bearing zones like `Europe/London` in tests).
- Enforcement points: (1) every child-authored content mutation; (2) message/room/board read queries return `QUIET_HOURS_ACTIVE`; (3) WS refuses room subscriptions and emits `quietHours.state`; (4) notification fanout to a quiet child follows §14.2 exactly — content-type notifications (message.new, board.*) are suppressed entirely (no push, no WS, no feed row; unread state derives from room read pointers §7.7), while child-directed system types still write feed rows with push deferred. **Exempt allowlist** (safety and basic account primitives trump time rules; exact constant `QUIET_EXEMPT_PROCEDURES` in `packages/shared`): `reports.create`, `auth.me`, `auth.quietState`, `auth.logout`, `auth.verifyChildPin`, `auth.updateLocale`, `notifications.feed`, `notifications.markRead`, `notifications.markAllRead`, `notifications.unreadCounts`, `spaces.list`, `spaces.get`.
- Client UX: lock screen with friendly countdown ("あさ 7:00 に あえるよ"); guardians see per-child schedule editor with per-day rows and copy-to-all.

### 13.6 Reporting flow

States: `open → resolved | dismissed` (guardian action with optional note). Reporter identity visible to reviewing guardians only; the reported member is **not** notified (child-protective default; prevents retaliation). **Reported-guardian exclusion**: when the report's target (or the targeted content's author) is themselves a guardian, that guardian is excluded from the queue, WS events, and notifications for that report — only the other guardians review it. If no other guardian exists, the report stays stored (visible to any future guardian) and the reporter's confirmation adds guidance to talk to another trusted adult; the operator does NOT gain content access (§13.7 boundary holds). Guardian notification on create. Reports reference the target id; if content is later deleted the report remains reviewable (tombstone rendering — no content snapshots are persisted).

### 13.7 Operator admin (instance level)

- REST `/admin/v1/*`, `Bearer OPERATOR_TOKEN`, IP-allowlist-able at Caddy, disabled if token unset. Endpoints: `POST /instance-invites`, `GET /spaces` (id, name, member counts, status — **no content access**), `POST /spaces/:id/suspend|unsuspend`, `POST /users/:id/suspend|unsuspend`, `GET /stats` (counts), `GET /audit` (instance-level).
- CLI wrapper `scripts/ops.mjs` (`pnpm ops <cmd>`) calls the API; all admin calls audited (`actor_kind=operator`).
- **Operator content-access policy**: no admin endpoint returns message/board/media content. Emergency content access = direct DB access on the host, which the policy doc (§22) requires to be logged manually and disclosed to the affected space. This is a governance control, not a technical one — documented honestly.

### 13.8 Audit log action catalog (constants in `packages/shared/src/audit.ts`)

`auth.login`, `auth.login_failed`, `auth.password_reset`, `session.revoke`, `space.create`, `space.settings_update`, `space.delete_request`, `space.delete_cancel`, `space.ownership_transfer`, `invite.create`, `invite.revoke`, `invite.accept`, `member.remove`, `member.role_change`, `member.leave`, `child.create`, `child.update`, `child.link_code_create`, `child.device_link`, `child.device_revoke`, `child.pin_set`, `child.remove`, `room.create`, `room.archive`, `room.members_update`, `message.delete_any`, `board.delete_any`, `board.pin`, `moderation.resolve`, `moderation.word_add`, `moderation.word_remove`, `report.resolve`, `report.dismiss`, `quiet_hours.update`, `export.request`, `export.download`, `admin.*` (per admin endpoint), `space.purged` (instance-level). Guardian-visible subset: space-scoped actions only. `auth.*` and `session.*` events are instance-level (operator-visible via §13.7) and are not exposed to end users in v1 (a "my security events" view is a v2 candidate).

---

## 14. Notifications

### 14.1 Pipeline

Mutation → `notifyService.enqueue(event)` (BullMQ) → worker composes per recipient: (1) INSERT `notifications` row → WS `notification.created`; (2) if user has push subscriptions and prefs allow: Web Push (web-push lib) and/or Expo push (expo-server-sdk). Push payloads contain **localized title/body + deep link**, composed server-side with i18next using each recipient's `users.locale`. Push text policy: room messages show sender display name + truncated body (30 chars) — acceptable because server-readable posture (ADR-003); flagged-content notifications never include matched terms.

### 14.2 Notification type catalog

| type | recipients | trigger | suppressed by |
|---|---|---|---|
| `message.new` | room members except sender | message.created (clean/flagged) | room notify=none; recipient child in quiet hours — **suppressed entirely** (no push, no WS, no feed row; unread state derives from room pointers §7.7, so nothing is lost and no content leaks through the lock) |
| `board.post.new` / `board.comment.new` | space members except author | board create | board_notify=none; child quiet hours — suppressed entirely (same rule) |
| `moderation.flagged` | space guardians | flag/block hit | never |
| `report.new` | space guardians | report.create | never |
| `child.device.linked` | space guardians | device link succeeds | never |
| `member.joined` | space members | invite accepted | none |
| `export.ready` | requesting owner | export job done | none |
| `quiet_hours.updated` | affected child | guardian saves schedule | none (delivered post-quiet-hours if locked) |

Expo push failure handling: `DeviceNotRegistered` ⇒ disable subscription; transient errors retry (BullMQ backoff, max 5). Web push 404/410 ⇒ disable.

---

## 15. Internationalization

- Catalogs in `packages/i18n/locales/{ja,en}/{common,auth,chat,board,guardian,safety,notifications,errors}.json`; keys are stable snake/dot case (`chat.composer.placeholder`). i18next instances: web (react-i18next + language detector order: user profile → cookie → Accept-Language), mobile (user profile → device locale), server (explicit locale per recipient for notifications/system messages/emails).
- **Completeness gate**: `scripts/check-i18n.mjs` fails CI if any key exists in one locale and not the other, or is referenced in code but missing (i18next-parser extraction). No untranslated fallback ships silently.
- **Child-register policy (ja)**: strings under `*.child.*` namespaces use kana-friendly wording; kanji beyond elementary grade 2 must carry ruby via the `<Furigana>` component (web/mobile) fed by `word|reading` markup in catalog values (`漢字|かんじ`). English child strings use simple words. Applied to all child-facing surfaces (chat, lock screen, onboarding, report dialog).
- Dates/numbers via `Intl` with space timezone; relative times ("さっき", "3分前") via a shared helper with unit tests per locale.

---

## 16. Web Application (Next.js)

- App Router, route groups: `(auth)` login/reset/invite-accept/child-link; `(app)` authed shell.
- Core routes: `/login`, `/reset`, `/invite/[code]`, `/link` (child device), `/s/[spaceId]` → room list (mobile-width) / split view (desktop), `/s/[spaceId]/r/[roomId]`, `/s/[spaceId]/board`, `/s/[spaceId]/board/[postId]`, `/s/[spaceId]/guardian/**` (dashboard, children/[id], moderation, reports, members, settings, audit), `/notifications`, `/settings` (profile/locale/push), space switcher in shell.
- Data layer: tRPC + TanStack Query; WS client updates query caches (message lists append, unread counters, notification feed). Optimistic send with `dedupeId`; failure → retry affordance.
- Kid mode: theme tokens (larger tap targets ≥ 44 px, larger type, high contrast, rounded), `<Furigana>` ruby rendering, simplified navigation when `membership.role=child` (rooms + board only; no settings beyond avatar/locale toggle if allowed), permanent oversight notice.
- PWA: manifest + icons + service worker (push + minimal offline shell showing "offline" state; no offline message queue in v1 — documented limitation).
- Accessibility: semantic landmarks, focus management in dialogs, WCAG AA contrast — acceptance criteria in UI issues.

## 17. Mobile Application (Expo)

- expo-router; screens mirror web routes: auth stack (login, invite accept, child QR link via expo-camera, PIN unlock), space tabs (Rooms, Board, Guardian [guardians only], Notifications, Settings).
- Session storage: expo-secure-store. WS via socket.io client. Push via expo-notifications (permission prompt after first successful chat, not at first launch). Deep links `famchat://` route to room/post/notification.
- Images: expo-image-picker (+ camera), upload with progress, retry.
- Child device UX: after device link the app stays signed in (180-day sliding session); PIN lock if set; quiet-hours lock screen matches web.
- Distribution v1: EAS Build with `development`, `preview` (internal: TestFlight / Play internal testing) profiles; runbook in repo. Store publication is v2 (ADR-004).

---

## 18. Deployment & Operations

### 18.1 Production topology (`infra/compose.prod.yml`)

Services: `caddy` (443/80, auto-TLS, reverse proxy: `/` → web:3000, `/trpc|/ws|/healthz|/readyz|/media|/admin` → api:8080), `web`, `api`, `worker`, `postgres` (volume), `redis` (volume, AOF), `minio` (volume), plus one-shot `migrate` service (`prisma migrate deploy`) run before api/worker start (compose `depends_on: condition: service_completed_successfully`).

- Images built by CI, published to GHCR (`ghcr.io/saber5656/famchat-{api,worker,web}`); compose pins a version tag; upgrade = edit tag → `docker compose pull && up -d` (migrate runs first).
- Healthchecks: containers define `HEALTHCHECK`; api `/readyz` gates caddy routing.
- Resource notes: single VPS ≥ 2 vCPU / 4 GB RAM target for beta (≤ ~20 spaces); sizing is a known unknown.
- `/admin/v1` additionally protected at Caddy with optional IP allowlist snippet.

### 18.2 Backup & restore

- Nightly cron (host): `scripts/backup.sh` = `pg_dump -Fc` + `mc mirror` (MinIO data) + retention (14 daily, 8 weekly) to a second disk/remote; secrets excluded.
- `scripts/restore.sh` + runbook `docs/ops/restore.md`; **restore is tested by a scripted drill** against a scratch compose (issue 50 acceptance).
- RPO 24 h / RTO 4 h documented for beta.

### 18.3 Self-host

`docs/selfhost.md`: prerequisites (VPS, domain, DNS A record, ports 80/443), install (clone → `.env` from example → generate secrets one-liner → `docker compose -f infra/compose.prod.yml up -d`), first operator steps (`pnpm ops instance-invite create`), upgrade, backup, troubleshooting, FAQ. English + Japanese (`docs/ja/selfhost.md`).

---

## 19. Security Model

### 19.1 Assets & actors

Assets: family message/media content (most sensitive: children's data), account credentials/sessions, invite/link codes, operator token, availability of the family's comms.
Threat actors: (T1) internet outsider; (T2) holder of a leaked invite/link code; (T3) malicious or compromised invited adult; (T4) curious/mischievous child within the space; (T5) stolen/lost child device; (T6) compromised guardian account; (T7) curious operator / compromised host; (T8) supply-chain (malicious dependency); (T9) abusive space creator (beta: operator-known invitees only).

### 19.2 Trust boundaries & controls

| Boundary | Controls |
|---|---|
| Internet → Caddy | TLS 1.2+ (auto certs), HSTS, 80→443 redirect; only 80/443 exposed; SSH hardening documented |
| Caddy → web/api | Internal compose network; no service publishes ports directly |
| Client → API | Session auth (§6.3), CSRF custom-header rule, strict CORS allowlist, zod validation on every input, output DTO mapping (no ORM objects serialized), rate limits (§19.5) |
| Client → WS | Session in handshake; per-room subscription permission checks; payloads zod-validated |
| API/worker → Postgres/Redis/MinIO | Compose-internal only; credentialed; least-privilege DB user (no SUPERUSER); Prisma parameterized queries (no raw SQL except reviewed migrations) |
| Upload → storage → serving | Quarantine prefix; magic-byte sniff; unconditional re-encode + metadata strip; size/dimension caps; private bucket; short-lived presigned GET; `Content-Disposition` + `X-Content-Type-Options: nosniff` |
| Push egress | Only to browser push endpoints / Expo API; no user-controlled URLs fetched by server (no unfurling ⇒ no SSRF surface) |
| Email egress | Only guardian password reset to stored address; token single-use |
| Admin REST | Bearer OPERATOR_TOKEN (≥ 32 bytes), constant-time compare, no CORS, optional IP allowlist, full audit |
| Operator ↔ content | No content endpoints (§13.7); governance policy in §22 |

### 19.3 Abuse cases → mitigations (selection; full table drives the issue 54 checklist)

| Abuse case | Mitigation |
|---|---|
| Invite code brute force | Codes = 128-bit random (URL) / 6-digit device codes are space+child-scoped, 10-min TTL, one-time, 5 attempts/15 min/IP rate limit, constant-time hash compare |
| Leaked child QR screenshot | One-time + 10-min expiry + guardian sees device list + `child.device.linked` notification to all guardians |
| Child spams 1000 messages | Per-user message rate limit (§19.5) + guardian visibility |
| Malicious image (polyglot, zip bomb, GPS leak) | Decode-and-re-encode only pipeline; dimension/size caps; metadata strip; originals destroyed |
| XSS via message/board text | No HTML rendering anywhere: plain text + newline only; React escaping; CSP (no `unsafe-inline` scripts); links not auto-linkified for children, confirm-dialog linkification for adults |
| CSRF | SameSite=Lax + custom header requirement + strict CORS |
| Session theft (XSS-independent) | HttpOnly cookies; tokens hashed at rest; revocation UI; rotation on login |
| Guardian account takeover | Strong password gate + reset revokes sessions + login-failure rate limits + audit `auth.login` with IP; TOTP is v2 |
| Operator overreach | No content APIs; audited admin; policy disclosure (§22); OSS = verifiable |
| Dependency compromise | pnpm lockfile, `pnpm audit` + osv-scanner in CI, Renovate weekly, minimal dependency policy (ADR-001), gitleaks secret scan in CI |
| DoS on VPS | Caddy + Fastify rate limits, body size caps, upload quota; accepted residual risk at beta scale |

### 19.4 Web platform hardening (issue 54 acceptance checklist source)

CSP (web): `default-src 'self'; script-src 'self'; img-src 'self' blob: https://<s3-host>; connect-src 'self' <api,ws origins>; frame-ancestors 'none'; base-uri 'none'; form-action 'self'` (+ Next.js-required nonces). Headers: HSTS (1y, preload-ready), `X-Content-Type-Options: nosniff`, `Referrer-Policy: same-origin`, `Permissions-Policy` minimal (camera only on link page). Cookies per §6.3.

### 19.5 Rate limit & quota policy (Redis-backed; per route defaults)

| Scope | Limit |
|---|---|
| `auth.login` / password reset request | 5 / 15 min / IP + 10 / h / account |
| invite preview/accept, child link attempts | 5 / 15 min / IP |
| `messages.sendText/sendImage` | 30 / min / user (burst 10/10 s) |
| board create/comment | 10 / min / user |
| `attachments.requestUpload` | 20 / h / user; space media quota 2 GiB (beta default, operator-adjustable) |
| reports | 10 / h / user |
| admin API | 60 / min / token |
| global HTTP | 300 / min / IP (Caddy) |

`RATE_LIMITED` responses include `details.retryAfterSec` (canonical field name). Limits are constants in `packages/shared/src/limits.ts`; keys enforced at the edge (Caddy) are marked `scope: 'edge'` and excluded from the API-side enforcement map. **Limiter availability policy**: if Redis is unavailable, rate limiting fails **closed** for credential/invite-class routes (login, reset, invite preview/accept, child link, PIN — request rejected `RATE_LIMITED`) and fails **open** for ordinary content routes — availability degradation is preferred over brute-force exposure on secrets, and accepted over availability on chat.

### 19.6 Secret handling rules

Secrets only via env (`.env` on host, chmod 600); `.env*` gitignored; `.env.example` placeholders only; no secret defaults in code; pino redaction list (`authorization`, `cookie`, `token`, `password`, `*_key`, `code`); secrets never in URLs (except one-time invite links which are credential-equivalent: stored hashed, short-lived, one-time); gitleaks in CI + pre-publication history scan (issue 54). Key generation documented one-liners (`openssl rand -base64 48`).

### 19.7 Vulnerability handling

`SECURITY.md`: private reporting via GitHub Security Advisories, 90-day coordinated disclosure, supported-version statement (latest minor), no bounty in beta. Dependabot/Renovate security PRs auto-raised.

---

## 20. Observability

- pino structured logs (request id, user id when authed — never message bodies), stdout → `docker compose logs` (+ optional loki hint in docs). Log levels by env.
- Metrics v1: lightweight `/admin/v1/stats` counters + healthchecks; Prometheus/Grafana is v2.
- Error tracking: optional self-host-friendly Sentry DSN env (off by default; documented privacy note). Worker job failures logged with job payload minus content.

## 21. Testing & Quality Strategy

| Layer | Tool | Scope | Gate |
|---|---|---|---|
| Unit | vitest | packages/* pure logic (authz matrix, moderation normalize/match, quiet hours, i18n helpers, env) | CI per PR |
| API integration | vitest + compose services (postgres/redis/minio) | routers with real DB; auth flows; permission denials; moderation pipeline; quotas | CI per PR |
| Web E2E | Playwright against compose stack | golden paths: onboard → invite → child link → chat → image → flag → report → quiet hours → board | CI (PR-labeled or nightly) |
| Mobile | jest-expo unit + manual smoke checklist per release | auth/link/chat/push happy paths | release runbook |
| i18n | `check-i18n.mjs` | catalog parity + unused/missing keys | CI per PR |
| Security | checklist issue 54 + gitleaks + pnpm audit/osv | §19 tables as acceptance items | pre-release gate |
| Restore drill | scripted | §18.2 | pre-release gate |

Per-issue Validation sections give exact commands; the definition of done for every implementation issue includes: typecheck, lint, unit tests green, and issue-specific validation steps.

## 22. Compliance Posture (Closed Beta)

- Basis: Japanese 個人情報保護法 (APPI) baseline; operator is an individual running an invited closed beta (families personally known to the operator). Public launch (v2) triggers a formal COPPA/GDPR/APPI re-assessment gate documented in ISSUE_PLAN §7.
- In-repo templates (`docs/legal/`, ja+en, plain-language, **not legal advice — owner reviews before use**): closed-beta Terms of Use, Privacy Notice (what is stored, that content is server-readable, operator access policy, retention, deletion rights, contact), Operator Data Access Policy (no casual access; emergency access logged + disclosed).
- Product-side commitments implemented in v1: guardian consent flow at signup (checkbox + links), data export (owner), space deletion with 7-day grace then purge, child data deletable by guardians, no third-party trackers/analytics, push text minimization option is v2.

## 23. Glossary & Conventions

| Term | Meaning |
|---|---|
| space | One family tenant |
| guardian / adult / child | Membership roles (§5.3) |
| owner | The single guardian membership with `is_owner` |
| device link | Child auth mechanism (§6.2) |
| flag / block | Moderation actions (§13.4) |
| quiet hours | Per-child time rules (§13.5) |
| quarantine / serve prefix | S3 `q/` unprocessed vs `m/` processed media |
| instance invite | Operator-issued space-creation code (beta gate) |

Naming: DB snake_case; TS camelCase; constants SCREAMING_SNAKE; i18n keys dot.case; branches `feat/NN-slug` matching issue numbers; commits Conventional Commits (`feat(api): …`).

