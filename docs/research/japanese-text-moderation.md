# Research: Japanese text moderation — normalization pitfalls and approach

Status: desk research (agent knowledge, 2026-07). Design impact: DESIGN.md
§13.4; `packages/moderation` (issues 25–26).

## Why naive word matching fails in Japanese

| Pitfall | Example | Countermeasure in famchat |
|---|---|---|
| Script variance: same word in hiragana/katakana/kanji/romaji | ばか / バカ / 馬鹿 / baka | Dictionary lists all script variants explicitly for kanji/romaji; katakana→hiragana fold handles kana variance mechanically |
| Full/half width variance | ﾊﾞｶ vs バカ, ＡＢＣ vs ABC | Unicode NFKC normalization first |
| Separator evasion | ば　か, ば.か, ば🌟か | Strip whitespace/symbols/emoji between letters before matching |
| Long-vowel / small-kana play | ばーか, ばぁか | Remove long-vowel marks; fold small kana to base kana |
| Leet/lookalike | b4ka, ba_ka | Basic leet fold map (0→o,1→i/l,3→e,4→a,@→a,$→s) |
| No word boundaries in ja | substring matching required | Aho–Corasick substring scan (not word-boundary regex) |
| False positives from substrings | しね inside 施設(しせつ)? — kana-folded text can over-match | Curate list terms ≥ 2 chars with care; flag-not-block default makes false positives low-cost (guardian review, not censorship) |

## Approach chosen

1. Single pure `normalizeText()` (NFKC → lowercase → katakana→hiragana →
   strip separators/long-vowels → leet fold) applied identically to
   dictionary terms and inputs — golden-file tested.
2. Aho–Corasick automaton over normalized text; per-space automaton cached
   and rebuilt on wordlist revision bump. At family scale (≤ few hundred
   terms, ≤ 4000-char inputs) performance is trivial.
3. Base lists are **data files with category comments**, ~150–300 terms per
   locale, seeded from: JIS-common profanity, school-bullying phrase lists
   (死ね/きもい/うざい class), sexual/violence vocabulary. Quality is a
   **known unknown** — lists ship as reviewable plain text, guardians extend
   per space, and false-positive tuning is expected during beta.
4. Explicit non-goals of v1 matching: ML toxicity scoring, context awareness
   (sarcasm, quoting), image-text OCR, kanji-lookalike (氏ね) beyond listed
   variants — each listed variant must be in the dictionary explicitly.
   AI-assisted moderation is a v2 candidate.

## Rejected alternatives

| Alternative | Why rejected for v1 |
|---|---|
| MeCab/kuromoji morphological analysis | Native/heavy deps, marginal gain for deny-list matching on kids' chat |
| Perspective API / hosted moderation | Sends children's messages to a third party; breaks self-host; contradicts data-minimization posture |
| Local LLM moderation | Ops footprint (GPU/RAM) incompatible with 4 GB VPS target; v2 research item |
