# Issue 26: `@famchat/moderation` — normalization, matcher, base lists

## Summary

Create the pure moderation package: the canonical `normalizeText` function,
an in-package Aho–Corasick matcher, curated ja/en base word lists as data
files, and the per-config compiled matcher with revision-keyed caching —
all side-effect-free and golden-tested.

## Context

DESIGN §13.4 and `docs/research/japanese-text-moderation.md` define the
normalization pipeline, evasion cases, and list philosophy (flag-not-block
default makes false positives low-cost). This package has zero runtime
dependencies and is consumed by the API in 27.

## Scope

In scope: package scaffold, `normalizeText`, Aho–Corasick implementation,
list files + loader, `buildMatcher`/`matchText` API, cache keying, golden +
property tests, micro-benchmark sanity.
Out of scope: API integration, per-space config storage, custom-word CRUD
(all 27), ML anything (v2).

## Detailed Requirements

1. Scaffold `packages/moderation` (`@famchat/moderation`), no runtime deps.
2. `src/normalize.ts` — `normalizeText(input: string): string`, exactly
   this order (research doc):
   1. Unicode NFKC; 2. `toLowerCase()`; 3. katakana→hiragana (U+30A1–
   U+30F6 → −0x60; also `ヴ`→`ゔ`); 4. remove long-vowel marks (ー U+30FC)
   and iteration marks (ゝゞ々); 5. fold small kana to base
   (ぁぃぅぇぉゃゅょっ→あいうえおやゆよつ); 6. strip separator characters
   **between** word characters: whitespace, zero-width (U+200B–D, U+FEFF),
   and Unicode categories P/S (punctuation/symbols incl. emoji); 7. leet
   fold map `0→o 1→i 3→e 4→a 5→s 7→t 8→b @→a $→s`. Export the map/steps
   as named constants so tests reference them.
3. `src/ahocorasick.ts` — classic trie + BFS failure links + output links;
   `build(terms: string[])` → automaton; `scan(text)` → array of
   `{ term, start, end }`; wholly deterministic; ~150 lines; include the
   textbook algorithm description in JSDoc for maintainers.
4. Lists: `lists/ja.txt`, `lists/en.txt` — one term per line, `#` comments
   delimit categories: profanity / sexual / violence-self-harm /
   bullying-exclusion. Target 150–300 terms per locale. Curate from
   commonly known Japanese school-bullying and profanity vocabulary and
   standard English profanity lists; **include script variants explicitly**
   (kanji/kana/romaji rows) since normalization only folds within-script
   variance. Terms must be ≥ 2 normalized chars. Loader
   `loadBuiltinList(locale)` parses, normalizes each term, dedupes, and
   throws on any term that normalizes to < 2 chars (list hygiene at build
   time).
5. `src/matcher.ts` — `buildMatcher(config: { builtinJa: boolean,
   builtinEn: boolean, customTerms: string[] })` → `{ match(text):
   Hit[] }` where `Hit = { term, source: 'builtin_ja'|'builtin_en'|
   'custom' }` (deduped by term); `createMatcherCache()` keyed by
   `(spaceId, wordlistRevision, builtinJa, builtinEn)` with LRU (≤ 200
   entries) — cache object is exported for the API to own instance-wide.
6. Golden tests (`test/golden.test.ts`): a fixture table of ≥ 40 cases
   pairing raw input → expected hit terms, covering every research-doc
   evasion row (ﾊﾞｶ, ば　か, ばーか, ば🌟か, b4ka, mixed-width, katakana/
   hiragana/kanji variants) and ≥ 10 must-NOT-match cases (benign words
   containing risky substrings; e.g. normalized boundaries respected per
   list curation). Property test: `matchText(normalizeText(x))` never
   throws on 10k random unicode strings (fast-check or hand-rolled fuzz).
7. Performance sanity test: 4000-char mixed-script input × 300-term
   automaton scans in < 5 ms median over 100 runs (soft-assert with log,
   hard-fail > 50 ms).

## Acceptance Criteria

- [ ] `normalizeText` implements the exact step order; each step
      unit-tested in isolation.
- [ ] All golden evasion cases pass; must-not-match cases pass.
- [ ] Lists load clean (hygiene checks), both locales ≥ 150 terms across
      all four categories.
- [ ] Zero runtime dependencies (`package.json` deps empty; test-only dev
      deps allowed).

## Validation

```bash
pnpm --filter @famchat/moderation test
pnpm --filter @famchat/moderation typecheck
```

## Dependencies

03 (constants/types only).

## Non-goals

Context awareness, ML scoring, OCR, kanji-lookalike generation beyond
listed variants, retroactive scans, list localization beyond ja/en.

## Design References

- DESIGN §13.4 (pipeline), research/japanese-text-moderation.md (all
  pitfalls + approach)
