# Issue 27: Moderation pipeline integration

## Summary

Wire `@famchat/moderation` into every content write path (messages, board
posts/comments, display names, room names), implement flag/block actions
with `moderation_hits`, guardian realtime + notification events, the
guardian flag queue and resolution APIs, and per-space custom words.

## Context

DESIGN §13.4 (actions), §10.2 (flagged lifecycle). Default space mode is
`flag` (deliver + notify guardians); `block` refuses content; display
names and room names are always block-mode. This issue replaces the
pass-through `moderateContent` hooks left by 14/19/13/25.

## Scope

In scope: migration (`moderation_hits`, `custom_ng_words`), moderation
service in api, hook replacements, `moderation` router (flagQueue,
resolveHit, custom words CRUD), wordlist revision bump, WS + notify
events, audit, tests.
Out of scope: guardian UI (31), report flow (28), base-list curation
changes (26 owns lists).

## Detailed Requirements

1. Migration per DESIGN §7.5: `moderation_hits`, `custom_ng_words`
   (unique (spaceId, normalizedTerm)). `spaces.wordlistRevision` already
   exists (04).
2. `apps/api/src/services/moderation.ts`:
   - `getSpaceMatcher(space)` — loads custom terms, consults the 26 cache
     keyed by (spaceId, wordlistRevision, toggles).
   - `moderateContent(ctx, { contentType, text, mode: 'space'|'block' })`
     → `{ status: 'clean' } | { status: 'flagged'|'blocked', hits }`
     applying: space mode for message/board types; forced block for
     `display_name` (and room names — treated as display_name type).
   - On flagged/blocked: insert `moderation_hits` per DESIGN §7.5
     (matched_terms with sources; `content_id` null for blocked-never-
     created content; `metadata.textSha256` for blocked attempts — raw
     attempted text is never persisted, canonical privacy rule), emit WS
     `moderation.flagged` to `space:<id>:guardians`, enqueue notify
     `moderation.flagged` to guardians — **the enqueued event carries
     category counts only, never matched terms** (DESIGN §14.1; 37
     renders from counts; a test asserts term absence from the event
     payload).
3. Replace hooks: 14 sendText/sendImage caption; 19 createPost/updatePost/
   createComment; 13 group-room name; 25 displayName. Blocked ⇒
   `CONTENT_BLOCKED_NG_WORD` with `details.categoryCount` only (never the
   matched terms — DESIGN §8.3).
4. Flagged message/post/comment rows get `moderation_status='flagged'`
   and a hit row referencing content_id; content stays visible per §10.2
   (non-guardians still see it rendered normally; DTO hides the status
   from them — already enforced in 14's mapper, extend to board DTOs).
5. `moderation` router (guardian-gated):
   - `flagQueue({ spaceId, resolution? (default 'pending'), cursor? })` —
     ordered id DESC, cursor = last hit id, 50/page; DTO
     `ModerationHitDTO = { id, contentType, action, resolution,
     matchedTerms (guardian-visible — full terms with sources, per
     §13.2), createdAt, content: LiveContentDTO | { deleted: true } |
     { blocked: true } , author: MemberChipDTO | null, link }` (pinned in
     `packages/shared/src/api/moderation.ts`; live content joined at
     read time — no snapshots persisted).
   - `resolveHit({ spaceId, hitId, resolution: 'approved'|'removed',
     note? })` — for hits **with** a content row: `approved` ⇒ content
     moderation_status→clean; `removed` ⇒ soft-delete the content (reuse
     14/19 delete services, attributed to the guardian). For hits
     **without** content (blocked messages/posts, display-name blocks):
     either resolution simply closes the hit (records reviewer +
     timestamp; there is nothing to mutate) — explicit rule + tests for
     both branches. Audit `moderation.resolve`.
   - `listCustomWords`, `addCustomWord({ term })` (validated: 1–50 chars,
     normalizes to ≥ 2 chars, per-space cap 500), `removeCustomWord` —
     each mutation bumps `wordlistRevision` in the same transaction;
     audit `moderation.word_add/remove`.
6. Space settings dependency: `moderation_mode` + builtin toggles are
   edited via `spaces.updateSettings` (30 exposes them; the service reads
   them now from the space row).
7. Tests: flag mode e2e (message with ja list term → delivered + hit row +
   guardian WS + notify stub called; sender sees normal send); block mode
   (space flipped → refused, no content row, hit row with text-hash only);
   display-name always blocked; custom word add → immediate effect (cache
   invalidated via revision — prove with before/after sends); resolveHit
   approved/removed paths incl. audit + content state; category-count-only
   error detail (no term leakage — snapshot test); non-guardian denied all
   queue/CRUD; cross-tenant sweep; evasion smoke (ﾊﾞｶ variant flagged
   end-to-end through the API path).

## Acceptance Criteria

- [ ] Every content write path moderated (meta-test: grep-style check that
      all `moderateOrPass` call sites are gone and replaced).
- [ ] Flag vs block vs always-block semantics exactly per DESIGN §13.4.
- [ ] Matched terms never leak to non-guardians (error details + DTO
      snapshot tests).
- [ ] Revision-keyed cache invalidation proven live.

## Validation

```bash
pnpm --filter @famchat/db db:migrate
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/api test -- -t moderation
```

## Dependencies

26, 14, 19 (13/25 hook sites). Guardian notification delivery completes
with 37.

## Non-goals

Report queue (28), guardian UI (31), retroactive re-scans on list change,
image moderation (v2).

## Design References

- DESIGN §13.4 (filter), §10.2 (flagged lifecycle), §7.5 (schema), §8.3
  (error detail policy), §9.2 (`moderation.flagged`)
