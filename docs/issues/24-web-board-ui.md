# Issue 24: Web bulletin board UI

## Summary

Build the bulletin board web screens: pinned-first post list, post detail
with comments, post composer with up to four images, pin/edit/delete
controls per role, and board realtime updates.

## Context

DESIGN §11 (board rules), §16 (routes `/s/[spaceId]/board` and
`/s/[spaceId]/board/[postId]`). Reuses the 23 uploader and the 20
shell/i18n foundations. The board is the "family noticeboard" —
optimized for readability by children and grandparents alike.

## Scope

In scope: board list/detail/composer/comment UIs, pin controls, edit
window UX, WS updates, empty states.
Out of scope: moderation badge queue (31), report action on posts (33),
mobile (45), notifications (37/39).

## Detailed Requirements

1. `/s/[spaceId]/board`: card list via `board.listPosts` — pinned section
   (📌 header, distinct background) then chronological; card = title,
   author chip, relative time, first-image thumbnail, comment count,
   body 2-line clamp; infinite scroll; kid mode enlarges cards/typography.
2. `/s/[spaceId]/board/[postId]`: full post (title, author, absolute +
   relative time, body with preserved newlines, image grid 1–4 with
   lightbox reuse), comments ascending with composer at bottom (2000 cap),
   deleted tombstones for post/comments.
3. Composer (`/s/[spaceId]/board/new` or modal): title (100 cap), body
   (8000 cap, auto-grow), up to 4 images via the 23 uploader component
   (grid preview, reorder by drag optional-skip, remove), submit disabled
   until all attachments ready; edit mode for own posts within the 15-min
   window (countdown chip "編集できるのは あと n 分"), text-only edit
   (attachments locked per 19).
4. Controls by role (matrix-driven, hidden not disabled): pin/unpin
   (guardian), delete-any (guardian, confirm dialog), delete-own, edit-own
   (window), comment (all).
5. Realtime: `board.postCreated`/`board.commentCreated` → cache patches
   (list prepend under pinned section, comment append, counts); reconnect
   invalidate.
6. Empty/edge states ja+en (+furigana): empty board CTA ("最初のお知らせを
   書いてみよう"), edit window expired, post deleted while viewing.
6b. Text rendering safety (DESIGN §19.3): post titles/bodies/comments
   render as plain text with newline preservation ONLY — no HTML, no
   Markdown, no auto-linkification for children; adult linkification (if
   rendered at all on board surfaces) reuses 22's scheme-allowlisted
   confirm-dialog component. React escaping assumed; a hostile-fixture
   component test proves `<img onerror>`-class strings render inert.
6c. Accessibility acceptance (DESIGN §16): list/detail use semantic
   landmarks and heading order; dialogs trap focus; interactive controls
   meet WCAG AA contrast and ≥ 44 px kid-mode targets — verified by an
   axe-core Playwright pass on the three board screens.
7. Tests: component — pin ordering render, edit countdown behavior with
   fake timers, role-based control visibility (guardian/adult/child
   fixtures); Playwright: create post with 2 images → second browser sees
   it live → comment → pin as guardian reorders list → delete tombstones.

## Acceptance Criteria

- [ ] All DESIGN §11 behaviors visible and role-gated correctly in UI.
- [ ] Live updates without reload; images via `/media` variants only.
- [ ] ja/en complete incl. child furigana on board chrome.
- [ ] Edit window UX matches server enforcement (no false affordances).

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/web test -- -t board
pnpm --filter @famchat/web exec playwright test --grep @board
```

## Dependencies

19, 21, 23 (uploader reuse).

## Non-goals

Reactions, rich text, board search, multiple boards, moderation queue UI
(31).

## Design References

- DESIGN §11 (board), §16 (web), §13.1 (permissions), §12 (media)
