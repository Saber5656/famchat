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
   **between word characters** — precise predicate: a word character is
   any code point whose Unicode general category is Letter (L*) or
   Number (N*) (covers kana/kanji/latin/digits; use the `\p{L}\p{N}`
   property escapes, NOT `\w`); a separator is any character in
   categories P*, S*, Z*, or Cf (incl. emoji and zero-width U+200B–D,
   U+FEFF) occurring between two word characters; 7. leet fold —
   exactly the canonical DESIGN §13.4 map: digits `0→o 1→i 3→e 4→a` and
   symbols `@→a $→s`, plus the letter fold `l→i` (this implements the
   research doc's `1→i/l` rule deterministically: both `1` and `l`
   normalize to `i`; dictionary terms pass through the same fold so
   matching stays consistent). No other leet mappings (`5,7,8` are NOT
   folded — canonical map only). Export the map/steps as named constants
   so tests reference them.
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
5. `src/matcher.ts` — exported API surface (exact signatures):
   - `buildMatcher(config: { builtinJa: boolean, builtinEn: boolean,
     customTerms: string[] }): Matcher` where `Matcher = { match(text:
     string): Hit[] }` and `Hit = { term: string, source: 'builtin_ja'|
     'builtin_en'|'custom' }` (deduped by term; `match` normalizes its
     input internally).
   - `matchText(matcher: Matcher, text: string): Hit[]` — thin
     convenience wrapper (kept so call sites read declaratively).
   - `createMatcherCache(limit = 200): MatcherCache` with
     `get(key: { spaceId, wordlistRevision, builtinJa, builtinEn },
     build: () => Matcher): Matcher` — LRU; the API owns one instance
     process-wide (27).
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
pnpm -w lint
pnpm --filter @famchat/moderation typecheck
pnpm --filter @famchat/moderation test
```

## Dependencies

03 (constants/types only).

## Non-goals

Context awareness, ML scoring, OCR, kanji-lookalike generation beyond
listed variants, retroactive scans, list localization beyond ja/en.

## Design References

- DESIGN §13.4 (pipeline), research/japanese-text-moderation.md (all
  pitfalls + approach)
