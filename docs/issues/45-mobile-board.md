# Issue 45: Mobile bulletin board

## Summary

Implement the mobile Board tab: pinned-first post list, post detail with
comments, post composer with images (reusing the 44 uploader), and
role-gated pin/edit/delete controls — parity with web board (24).

## Context

DESIGN §11 (board rules), §17. The board is the grandparent-friendliest
surface; large type and simple navigation matter. 24 is the reference
implementation for interaction rules.

## Scope

In scope: board list/detail/composer/comments screens, pin/edit/delete
controls, realtime updates, kid/grandparent-friendly presentation.
Out of scope: report action on board content (47 wires the shared
dialog), notifications (46), web (24).

## Detailed Requirements

1. Board tab list: cards per 24's anatomy (pinned section with 📌 header,
   title, author chip, relative time, first-image thumb, comment count,
   2-line body clamp); FlashList + infinite scroll + pull-to-refresh;
   compose FAB (all roles per matrix).
2. Post detail: full content, image grid (1–4) with the 44 lightbox,
   comments ascending, comment composer (2000 cap), deleted tombstones;
   kebab menu per role: pin/unpin (guardian), edit (own, 15-min
   countdown chip), delete (own / guardian confirm).
3. Composer screen: title (100) / body (8000, auto-grow) / up to 4
   images via the 44 uploader (grid preview, remove); submit gated on
   all-ready; edit mode text-only per 19.
4. Realtime: `board.postCreated` / `board.commentCreated` via the 43
   socket hook → cache patches; reconnect invalidation.
5. Presentation: base type scale bumped on this tab (grandparent
   consideration — 17 pt body minimum adult mode; kid mode inherits
   larger); date display absolute + relative per 40 helpers.
6. Empty/edge states ja/en+furigana matching 24's keys (shared
   namespace — no new strings unless mobile-specific).
7. Tests: component — card variants, role-gated menus (guardian/adult/
   child fixtures), edit countdown fake-timer; manual checklist: create
   post with 2 photos on device → web sees it live; pin on device
   reorders web list; comment round-trip; child posts allowed, pin
   hidden.

## Acceptance Criteria

- [ ] Full board loop device↔web live (checklist evidence).
- [ ] All DESIGN §11 role rules mirrored (matrix-driven component
      tests).
- [ ] Uploader reuse — no duplicated upload logic (import graph check).
- [ ] Typography meets the enlarged-scale spec on this tab.

## Validation

```bash
pnpm --filter @famchat/mobile test -- --grep board
# manual: device↔web board checklist
```

## Dependencies

42 (auth), 44 (uploader), 19 (API). Reference: 24.

## Non-goals

Reactions, rich text, board search, report entry point (47), multiple
boards.

## Design References

- DESIGN §11 (board), §17 (mobile), §13.1 (permissions), §15 (registers)
