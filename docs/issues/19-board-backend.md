# Issue 19: Bulletin board backend

## Summary

Implement the space bulletin board: posts (title/body/≤4 images/pinning),
flat comments, the author edit window, board notification preference
column, WS events, and the `board` router.

## Context

DESIGN §11: one implicit board per space; posts are family notices —
pinned-first ordering, guardian pin control, everyone may post/comment by
default. Moderation hooks pass-through until 27.

## Scope

In scope: migration (board_posts, board_post_attachments, board_comments,
`memberships.board_notify`), `board` router, attachment claiming for
posts, WS events, audit, tests.
Out of scope: web/mobile UI (24/45), moderation behavior (27), notification
delivery (37).

## Detailed Requirements

1. Migration per DESIGN §7.3 + `board_notify enum(all,none) default all`
   on memberships. `board_post_attachments` composite PK (post_id,
   attachment_id) + `position int` (0–3) **+ unique index on
   `attachment_id` alone** — combined with `messages.attachment_id`
   uniqueness (14/17), this makes the DESIGN §12.1 single-use invariant
   database-enforced in both directions; the claim algorithm must check
   `messages.attachment_id IS NULL` for the attachment before inserting
   the join row inside one transaction (and vice versa in 14's claim).
2. `board.createPost({ spaceId, title, body, attachmentIds?[≤4] })` —
   permission `board.post` (all roles); pipeline order per §8.4 (quiet-
   hours hook, moderation hook on title+body); attachments claimed per
   the req-1 transactional algorithm (cross-claim tested both
   directions); post-commit `onBoardPostCreated` (WS `board.postCreated`
   to space + notify stub). **Quiet-hours scope note**: every
   child-authored `board.*` mutation in this issue (createPost,
   updatePost, deletePost, createComment, deleteComment, setNotify)
   calls the `assertNotQuiet` hook — issue 29 activates it and its
   procedure list must match this set.
3. `board.updatePost({ spaceId, postId, title?, body? })` — author only,
   within 15 min of createdAt (`VALIDATION_FAILED details.editWindow`
   after); re-runs moderation hook; attachments immutable after create
   (v1 simplification).
4. `board.deletePost({ spaceId, postId })` — own or `board.deleteAny`
   (guardian; audited `board.delete_any`); soft delete. Deleted-post
   semantics: excluded from `listPosts`; `getPost({ spaceId, postId })`
   returns a tombstone DTO (`{ id, deleted: true, author: null, title:
   null, body: null }`); `listComments` for a tombstoned post remains
   available (existing comments render under the tombstone header —
   deep-linked notifications stay resolvable).
5. `board.pinPost` / `unpinPost` — guardian (`board.pin`); sets/clears
   pinnedAt; audit `board.pin`.
6. `board.listPosts({ spaceId, cursor? })` — members; pinned first
   (pinnedAt DESC) then createdAt DESC; DTO includes author, counts
   (comments), thumbnails; deleted posts excluded from list (tombstone only
   on direct `getPost` for deep links).
7. `board.listComments({ spaceId, postId, cursor? })` (ascending, cursor =
   last comment id, 50/page), `board.createComment({ spaceId, postId,
   body })` (pipeline hooks; `board.commentCreated` WS + notify stub),
   `board.deleteComment({ spaceId, commentId })` — own, or guardian
   (guardian path audited `board.delete_any` with targetType
   `board_comment`). DTO contracts pinned in
   `packages/shared/src/api/board.ts`: `BoardPostDTO = { id, author:
   MemberChipDTO|null, title, body, pinnedAt, images: [{ attachmentId,
   position }], commentCount, createdAt, updatedAt, deleted }`,
   `BoardCommentDTO = { id, postId, author, body, createdAt, deleted }`
   — mapper redacts deleted rows exactly like 14's message mapper (no
   ORM objects serialized; snapshot-tested).
8. Board notify pref: `board.setNotify({ spaceId, notify })` writes
   membership column (consumed by 37's fanout).
9. Tests: pin ordering; edit window boundary (14:59 ok, 15:01 rejected via
   injected clock); cross-claim of attachments blocked both directions;
   soft-deleted post/comment redaction; permission rows (child can post/
   comment, only guardian pins/deletes-any); comment pagination; WS events
   observed; cross-tenant sweep; rate limit `boardWrite` (10/min)
   registered in the 12 map and tripped in test.

## Acceptance Criteria

- [ ] Ordering, edit-window, and pin semantics exactly per DESIGN §11.
- [ ] Attachment single-use invariant holds across messages and posts.
- [ ] All DESIGN §13.1 board permission rows test-covered.
- [ ] WS events + audit rows emitted.

## Validation

```bash
pnpm --filter @famchat/db db:migrate && pnpm --filter @famchat/api test -- --grep board
```

## Dependencies

13 (space/membership context), 15 (WS emits), 18 (ready attachments for
image posts; text posts testable with 17 alone).

## Non-goals

Nested comments, reactions, rich text, board search, multiple boards per
space (all v1 non-goals / v2).

## Design References

- DESIGN §7.3 (schema), §11 (board rules), §8.2 (`board` router), §13.1
  (permissions)
