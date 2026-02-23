# Word Dictionary Solve Mode — Design

## Overview

Add a new "word" solve mode that searches English nouns (single-word) and
adjective+noun / noun+noun phrases (two-word) against redaction pixel widths.
Both search phases run simultaneously when the user selects word mode.

## UI Changes

### Mode Dropdown

Add `"word"` option to the existing solve-mode `<select>` alongside
name / full_name / email / enumerate.

### New Control: Ensure Plural

- Checkbox, visible only when `mode=word`.
- When checked:
  - Single-word results must be plural nouns.
  - Two-word results: the last word (noun) must be a plural form.
- Default: off.

### Existing Controls (unchanged behavior)

- **Charset**: lowercase / uppercase / capitalized — applies casing to word
  candidates (same as name mode).
- **Tolerance**: pixel tolerance slider.
- **Known start / known end**: prefix/suffix constraints on the full phrase.
- **Word filter dropdown**: hidden for word mode (redundant since we're already
  searching a curated dictionary).

## Data Files

Three new files in `unredact/data/`:

### `nouns.txt` (~40K entries)

Source: WordNet via NLTK (`nltk.corpus.wordnet`), filtered to noun synsets.
One lowercase word per line. Excludes entries containing digits, underscores,
or slashes.

### `nouns_plural.txt`

Plural forms of the nouns. Generated via:

1. **Wiktionary mappings** from
   [djstrong/nouns-with-plurals](https://github.com/djstrong/nouns-with-plurals)
   — covers irregular plurals (children, mice, etc.).
2. **English pluralisation rules** for nouns not in the Wiktionary mapping:
   - Ends in s/x/z/sh/ch → append `es`
   - Ends in consonant+y → replace `y` with `ies`
   - Ends in f → replace `f` with `ves`
   - Ends in fe → replace `fe` with `ves`
   - Otherwise → append `s`

### `adjectives.txt` (~20–30K entries)

Source: WordNet via NLTK, filtered to adjective synsets. Same exclusion rules
as nouns.

## Backend

### New Function: `solve_word_dictionary()`

Location: `unredact/pipeline/dictionary.py`

```
solve_word_dictionary(
    font, target_width, tolerance,
    left_context, right_context,
    casing, known_start, known_end,
    ensure_plural,
) -> Generator[SolveResult]
```

Runs two phases, yielding results as they're found (for SSE streaming):

#### Phase 1 — Single-Word Noun Search

1. Load `nouns.txt` (or `nouns_plural.txt` if `ensure_plural`).
2. Apply casing transform to each noun.
3. Filter by `known_start` / `known_end`.
4. Measure width with kerning context (same as existing `solve_dictionary`).
5. Yield matches within tolerance.

#### Phase 2 — Two-Word Search

Word pools:
- **Word1**: adjectives ∪ nouns (combined ~60K entries).
- **Word2**: nouns (or plural nouns if `ensure_plural`).

Algorithm (sorted binary search, O(n log n)):

1. Apply casing to all word2 candidates.
2. Measure each word2's width with `font.getlength()`.
3. Sort word2 list by width → array of `(width, word)`.
4. For each word1 candidate:
   a. Filter by `known_start` if set.
   b. Measure "left portion" = width of `left_context + cased_word1 + " "`,
      minus `left_context` width.
   c. Compute `remaining = target_width - left_portion`.
   d. Binary search sorted word2 array for candidates in range
      `[remaining - tolerance - 3px, remaining + tolerance + 3px]`
      (3px kerning margin).
   e. For each candidate word2:
      - Filter by `known_end` if set.
      - Verify exact combined width:
        `font.getlength(left_ctx + word1 + " " + word2 + right_ctx)`
        minus context widths.
      - If error ≤ tolerance, yield as `SolveResult`.

### API Changes

`SolveRequest` (in `app.py`) gains:

```python
ensure_plural: bool = False
```

Route `mode="word"` to `solve_word_dictionary()`.

### Word List Loading

Add to `word_filter.py` (or a new loader module):

- `_get_nouns() -> list[str]` — lazy-loaded, cached.
- `_get_nouns_plural() -> list[str]` — lazy-loaded, cached.
- `_get_adjectives() -> list[str]` — lazy-loaded, cached.

## Frontend Changes

### `index.html`

- Add `<option value="word">word</option>` to `#solve-mode`.
- Add `<label id="plural-label"><input type="checkbox" id="solve-plural"> Ensure plural</label>`.

### `popover.js`

- Show/hide `#plural-label` based on `solveMode.value === "word"`.
- Hide word-filter dropdown when mode is `"word"`.

### `solver.js`

- Include `ensure_plural` in the `POST /api/solve` body when mode is `"word"`.

## Performance Estimate

- Phase 1 (single word): ~40K width measurements ≈ <1 second.
- Phase 2 (two words): ~60K word1 × binary search over ~40K sorted word2
  ≈ 2–5 seconds depending on font/tolerance.
- Total: results start streaming within ~1 second; full search completes in
  under 10 seconds.

## Word List Generation

One-time script (`scripts/generate_word_lists.py` or similar):

1. Use NLTK to extract nouns and adjectives from WordNet.
2. Download djstrong Wiktionary noun.csv for irregular plural mappings.
3. Generate `nouns_plural.txt` using Wiktionary + rules.
4. Write `nouns.txt`, `nouns_plural.txt`, `adjectives.txt` to `unredact/data/`.
5. Commit generated files to the repo (they're small enough).
