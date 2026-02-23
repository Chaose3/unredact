# Word Dictionary Solve Mode — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a "word" solve mode that searches English nouns (single-word) and adjective/noun + noun phrases (two-word) against redaction pixel widths, with optional plural filtering.

**Architecture:** New `solve_word_dictionary()` generator in `dictionary.py` runs two phases — single-noun width matching, then two-word binary search. Word lists (nouns, adjectives, plurals) are generated from WordNet/Wiktionary via a build script and committed as static data files.

**Tech Stack:** Python (PIL for font metrics, bisect for binary search), NLTK WordNet for word list extraction, Wiktionary CSV for irregular plurals.

---

### Task 1: Word List Generation Script

**Files:**
- Create: `scripts/build_word_lists.py`
- Create: `unredact/data/nouns.txt`
- Create: `unredact/data/nouns_plural.txt`
- Create: `unredact/data/adjectives.txt`

**Step 1: Write the build script**

```python
# scripts/build_word_lists.py
"""Generate noun, adjective, and plural word lists from WordNet + Wiktionary.

Usage:
    python scripts/build_word_lists.py [--data-dir PATH]

Requires: nltk (with wordnet corpus downloaded)
    pip install nltk
    python -c "import nltk; nltk.download('wordnet')"

For irregular plurals, downloads noun.csv from djstrong/nouns-with-plurals.
"""

import argparse
import csv
import io
import re
import urllib.request
from pathlib import Path

OUTPUT_DIR = Path(__file__).parent.parent / "unredact" / "data"

WIKTIONARY_CSV_URL = (
    "https://raw.githubusercontent.com/djstrong/nouns-with-plurals/master/noun.csv"
)

# Pattern: only simple lowercase alpha words (no digits, hyphens, spaces, etc.)
WORD_RE = re.compile(r"^[a-z]+$")


def extract_wordnet_words(pos: str) -> list[str]:
    """Extract unique lemma names from WordNet for a given POS tag.

    pos: 'n' for nouns, 'a' for adjectives (includes 's' satellite adjectives).
    """
    from nltk.corpus import wordnet as wn

    words = set()
    pos_tags = [pos] if pos != "a" else ["a", "s"]
    for tag in pos_tags:
        for synset in wn.all_synsets(tag):
            for lemma in synset.lemmas():
                name = lemma.name().lower()
                if WORD_RE.match(name):
                    words.add(name)
    return sorted(words)


def download_wiktionary_plurals() -> dict[str, str]:
    """Download noun.csv from djstrong/nouns-with-plurals.

    Returns dict mapping singular -> first plural form.
    """
    response = urllib.request.urlopen(WIKTIONARY_CSV_URL)
    text = response.read().decode("utf-8")
    mapping = {}
    reader = csv.reader(io.StringIO(text))
    for row in reader:
        if len(row) >= 2:
            singular = row[0].strip().lower()
            plural = row[1].strip().lower()
            if WORD_RE.match(singular) and WORD_RE.match(plural):
                mapping[singular] = plural
    return mapping


def pluralize(word: str) -> str:
    """Apply standard English pluralization rules."""
    if word.endswith(("s", "x", "z")):
        return word + "es"
    if word.endswith(("sh", "ch")):
        return word + "es"
    if word.endswith("y") and len(word) > 1 and word[-2] not in "aeiou":
        return word[:-1] + "ies"
    if word.endswith("f"):
        return word[:-1] + "ves"
    if word.endswith("fe"):
        return word[:-2] + "ves"
    return word + "s"


def main():
    parser = argparse.ArgumentParser(description="Generate word lists from WordNet")
    parser.add_argument("--data-dir", type=Path, default=OUTPUT_DIR)
    args = parser.parse_args()
    data_dir = args.data_dir
    data_dir.mkdir(parents=True, exist_ok=True)

    print("Extracting nouns from WordNet...")
    nouns = extract_wordnet_words("n")
    print(f"  Found {len(nouns)} nouns")

    print("Extracting adjectives from WordNet...")
    adjectives = extract_wordnet_words("a")
    print(f"  Found {len(adjectives)} adjectives")

    print("Downloading Wiktionary irregular plurals...")
    wiktionary = download_wiktionary_plurals()
    print(f"  Found {len(wiktionary)} irregular plural mappings")

    print("Generating plural forms...")
    noun_set = set(nouns)
    plurals = []
    for noun in nouns:
        if noun in wiktionary:
            plurals.append(wiktionary[noun])
        else:
            plurals.append(pluralize(noun))
    print(f"  Generated {len(plurals)} plural forms")

    nouns_path = data_dir / "nouns.txt"
    nouns_path.write_text("\n".join(nouns) + "\n")
    print(f"Wrote {nouns_path} ({len(nouns)} entries)")

    plurals_path = data_dir / "nouns_plural.txt"
    plurals_path.write_text("\n".join(plurals) + "\n")
    print(f"Wrote {plurals_path} ({len(plurals)} entries)")

    adj_path = data_dir / "adjectives.txt"
    adj_path.write_text("\n".join(adjectives) + "\n")
    print(f"Wrote {adj_path} ({len(adjectives)} entries)")


if __name__ == "__main__":
    main()
```

**Step 2: Install NLTK and download WordNet data**

Run:
```bash
.venv/bin/pip install nltk
.venv/bin/python -c "import nltk; nltk.download('wordnet')"
```

**Step 3: Run the build script**

Run: `.venv/bin/python scripts/build_word_lists.py`
Expected: Three new files created in `unredact/data/` with counts printed.

**Step 4: Verify output**

Run:
```bash
wc -l unredact/data/nouns.txt unredact/data/nouns_plural.txt unredact/data/adjectives.txt
head -20 unredact/data/nouns.txt
head -20 unredact/data/nouns_plural.txt
head -20 unredact/data/adjectives.txt
```
Expected: ~40K nouns, ~40K plurals, ~20-30K adjectives. All lowercase alpha words.

**Step 5: Add Makefile target**

In `Makefile`, after the `build-emails` target (line 161), add:

```makefile
build-word-lists:
	$(PYTHON) scripts/build_word_lists.py
```

Update the `.PHONY` line at top to include `build-word-lists`.

**Step 6: Commit**

```bash
git add scripts/build_word_lists.py unredact/data/nouns.txt unredact/data/nouns_plural.txt unredact/data/adjectives.txt Makefile
git commit -m "feat: add word list generation script (nouns, adjectives, plurals from WordNet)"
```

---

### Task 2: Word List Loaders

**Files:**
- Modify: `unredact/pipeline/word_filter.py`
- Test: `tests/test_word_lists.py`

**Step 1: Write the failing tests**

Create `tests/test_word_lists.py`:

```python
# tests/test_word_lists.py
"""Tests for word list loading (nouns, adjectives, plurals)."""

from unredact.pipeline.word_filter import (
    _get_nouns,
    _get_nouns_plural,
    _get_adjectives,
)


class TestWordListLoaders:
    def test_nouns_loads(self):
        nouns = _get_nouns()
        assert len(nouns) > 1000
        assert isinstance(nouns, list)
        assert "dog" in nouns

    def test_nouns_plural_loads(self):
        plurals = _get_nouns_plural()
        assert len(plurals) > 1000
        assert isinstance(plurals, list)
        assert "dogs" in plurals

    def test_adjectives_loads(self):
        adjectives = _get_adjectives()
        assert len(adjectives) > 1000
        assert isinstance(adjectives, list)
        assert "large" in adjectives

    def test_nouns_and_plurals_same_length(self):
        """Each noun has exactly one plural form at the same index."""
        nouns = _get_nouns()
        plurals = _get_nouns_plural()
        assert len(nouns) == len(plurals)

    def test_all_words_lowercase_alpha(self):
        """All words should be lowercase alphabetic only."""
        import re
        pattern = re.compile(r"^[a-z]+$")
        for word in _get_nouns()[:100]:
            assert pattern.match(word), f"Non-alpha noun: {word}"
        for word in _get_adjectives()[:100]:
            assert pattern.match(word), f"Non-alpha adjective: {word}"
```

**Step 2: Run tests to verify they fail**

Run: `.venv/bin/python -m pytest tests/test_word_lists.py -v`
Expected: FAIL — `_get_nouns`, `_get_nouns_plural`, `_get_adjectives` don't exist yet.

**Step 3: Add loaders to word_filter.py**

In `unredact/pipeline/word_filter.py`, after the `_get_associate_variants` function (after line 92), add:

```python
_nouns: list[str] | None = None
_nouns_plural: list[str] | None = None
_adjectives: list[str] | None = None


def _load_list(filename: str) -> list[str]:
    path = DATA_DIR / filename
    if not path.exists():
        return []
    return [line.strip() for line in path.read_text().splitlines() if line.strip()]


def _get_nouns() -> list[str]:
    global _nouns
    if _nouns is None:
        _nouns = _load_list("nouns.txt")
    return _nouns


def _get_nouns_plural() -> list[str]:
    global _nouns_plural
    if _nouns_plural is None:
        _nouns_plural = _load_list("nouns_plural.txt")
    return _nouns_plural


def _get_adjectives() -> list[str]:
    global _adjectives
    if _adjectives is None:
        _adjectives = _load_list("adjectives.txt")
    return _adjectives
```

**Step 4: Run tests to verify they pass**

Run: `.venv/bin/python -m pytest tests/test_word_lists.py -v`
Expected: All 5 tests PASS.

**Step 5: Commit**

```bash
git add unredact/pipeline/word_filter.py tests/test_word_lists.py
git commit -m "feat: add noun/adjective/plural word list loaders"
```

---

### Task 3: Single-Word Noun Solver

**Files:**
- Modify: `unredact/pipeline/dictionary.py`
- Test: `tests/test_word_solve.py`

**Step 1: Write the failing tests**

Create `tests/test_word_solve.py`:

```python
# tests/test_word_solve.py
"""Tests for word dictionary solve mode."""

import subprocess
import pytest
from PIL import ImageFont

from unredact.pipeline.dictionary import solve_word_dictionary


def _get_test_font() -> ImageFont.FreeTypeFont:
    result = subprocess.run(
        ["fc-match", "--format=%{file}", "Liberation Serif"],
        capture_output=True, text=True,
    )
    path = result.stdout.strip()
    return ImageFont.truetype(path, 40)


class TestSolveWordDictionarySingleWord:
    def test_finds_exact_noun(self):
        font = _get_test_font()
        target = font.getlength("house")
        results = list(solve_word_dictionary(
            font, target, tolerance=0.5,
            casing="lowercase",
        ))
        texts = [r.text for r in results]
        assert "house" in texts

    def test_plural_mode_filters(self):
        font = _get_test_font()
        target = font.getlength("houses")
        results = list(solve_word_dictionary(
            font, target, tolerance=0.5,
            casing="lowercase", ensure_plural=True,
        ))
        texts = [r.text for r in results]
        # "houses" should be found in plural mode
        assert "houses" in texts

    def test_casing_capitalized(self):
        font = _get_test_font()
        target = font.getlength("House")
        results = list(solve_word_dictionary(
            font, target, tolerance=0.5,
            casing="capitalized",
        ))
        texts = [r.text for r in results]
        assert "House" in texts

    def test_casing_uppercase(self):
        font = _get_test_font()
        target = font.getlength("HOUSE")
        results = list(solve_word_dictionary(
            font, target, tolerance=0.5,
            casing="uppercase",
        ))
        texts = [r.text for r in results]
        assert "HOUSE" in texts

    def test_known_start_filters(self):
        font = _get_test_font()
        target = font.getlength("house")
        results = list(solve_word_dictionary(
            font, target, tolerance=0.5,
            casing="lowercase", known_start="ho",
        ))
        texts = [r.text for r in results]
        assert "house" in texts
        # No results starting with other letters
        assert all(t.lower().startswith("ho") for t in texts)

    def test_known_end_filters(self):
        font = _get_test_font()
        target = font.getlength("house")
        results = list(solve_word_dictionary(
            font, target, tolerance=0.5,
            casing="lowercase", known_end="se",
        ))
        texts = [r.text for r in results]
        assert "house" in texts
        assert all(t.lower().endswith("se") for t in texts)

    def test_returns_generator(self):
        """solve_word_dictionary should be a generator for SSE streaming."""
        font = _get_test_font()
        target = font.getlength("dog")
        gen = solve_word_dictionary(font, target, tolerance=0.5, casing="lowercase")
        # Should be a generator, not a list
        import types
        assert isinstance(gen, types.GeneratorType)
```

**Step 2: Run tests to verify they fail**

Run: `.venv/bin/python -m pytest tests/test_word_solve.py::TestSolveWordDictionarySingleWord -v`
Expected: FAIL — `solve_word_dictionary` doesn't exist yet.

**Step 3: Implement solve_word_dictionary (phase 1 only)**

In `unredact/pipeline/dictionary.py`, add at the bottom:

```python
from typing import Generator


def solve_word_dictionary(
    font: ImageFont.FreeTypeFont,
    target_width: float,
    tolerance: float = 0.0,
    left_context: str = "",
    right_context: str = "",
    casing: str = "lowercase",
    known_start: str = "",
    known_end: str = "",
    ensure_plural: bool = False,
) -> Generator[SolveResult, None, None]:
    """Search English nouns (single-word) and adj+noun phrases (two-word).

    Yields results as found for SSE streaming. Phase 1 searches single nouns,
    phase 2 searches two-word combinations using binary search.
    """
    from unredact.pipeline.word_filter import (
        _get_nouns,
        _get_nouns_plural,
    )

    # Phase 1: single-word noun search
    nouns = _get_nouns()
    plurals = _get_nouns_plural()

    if ensure_plural:
        word_list = plurals
    else:
        word_list = nouns

    ks_lower = known_start.lower()
    ke_lower = known_end.lower()
    seen: set[str] = set()

    for word in word_list:
        if ks_lower and not word.startswith(ks_lower):
            continue
        if ke_lower and not word.endswith(ke_lower):
            continue

        display = _apply_casing(word, casing)
        if display in seen:
            continue
        seen.add(display)

        if left_context or right_context:
            full = left_context + display + right_context
            full_len = font.getlength(full)
            left_len = font.getlength(left_context) if left_context else 0.0
            right_len = font.getlength(right_context) if right_context else 0.0
            width = full_len - left_len - right_len
        else:
            width = font.getlength(display)

        error = abs(width - target_width)
        if error <= tolerance:
            yield SolveResult(text=display, width=float(width), error=float(error))
```

Note: `_apply_casing` already exists in `dictionary.py` (line 65).

**Step 4: Run tests to verify they pass**

Run: `.venv/bin/python -m pytest tests/test_word_solve.py::TestSolveWordDictionarySingleWord -v`
Expected: All 7 tests PASS.

**Step 5: Commit**

```bash
git add unredact/pipeline/dictionary.py tests/test_word_solve.py
git commit -m "feat: add single-word noun solver (phase 1 of word mode)"
```

---

### Task 4: Two-Word Binary Search Solver (Phase 2)

**Files:**
- Modify: `unredact/pipeline/dictionary.py`
- Modify: `tests/test_word_solve.py`

**Step 1: Write the failing tests**

Append to `tests/test_word_solve.py`:

```python
class TestSolveWordDictionaryTwoWord:
    def test_finds_two_word_phrase(self):
        font = _get_test_font()
        target = font.getlength("large house")
        results = list(solve_word_dictionary(
            font, target, tolerance=1.0,
            casing="lowercase",
        ))
        texts = [r.text for r in results]
        assert "large house" in texts

    def test_two_word_capitalized(self):
        font = _get_test_font()
        target = font.getlength("Large House")
        results = list(solve_word_dictionary(
            font, target, tolerance=1.0,
            casing="capitalized",
        ))
        texts = [r.text for r in results]
        assert "Large House" in texts

    def test_two_word_plural(self):
        font = _get_test_font()
        target = font.getlength("large houses")
        results = list(solve_word_dictionary(
            font, target, tolerance=1.0,
            casing="lowercase", ensure_plural=True,
        ))
        texts = [r.text for r in results]
        assert "large houses" in texts

    def test_two_word_known_start(self):
        font = _get_test_font()
        target = font.getlength("large house")
        results = list(solve_word_dictionary(
            font, target, tolerance=1.0,
            casing="lowercase", known_start="la",
        ))
        texts = [r.text for r in results]
        assert "large house" in texts
        assert all(t.lower().startswith("la") for t in texts)

    def test_two_word_known_end(self):
        font = _get_test_font()
        target = font.getlength("large house")
        results = list(solve_word_dictionary(
            font, target, tolerance=1.0,
            casing="lowercase", known_end="se",
        ))
        texts = [r.text for r in results]
        assert "large house" in texts
        assert all(t.lower().endswith("se") for t in texts)

    def test_no_duplicates_across_phases(self):
        """Single-word and two-word results should not have duplicates."""
        font = _get_test_font()
        target = font.getlength("dog")
        results = list(solve_word_dictionary(
            font, target, tolerance=1.0,
            casing="lowercase",
        ))
        texts = [r.text for r in results]
        assert len(texts) == len(set(texts))
```

**Step 2: Run tests to verify new ones fail**

Run: `.venv/bin/python -m pytest tests/test_word_solve.py::TestSolveWordDictionaryTwoWord -v`
Expected: FAIL — two-word phrases won't be found (only phase 1 implemented).

**Step 3: Add phase 2 to solve_word_dictionary**

At the end of the `solve_word_dictionary` function in `dictionary.py`, after the phase 1 loop, add the phase 2 code. The full function body after phase 1 continues:

```python
    # Phase 2: two-word search (word1 + " " + noun)
    import bisect

    from unredact.pipeline.word_filter import _get_adjectives

    adjectives = _get_adjectives()

    # Word1 pool: adjectives + nouns (deduplicated)
    word1_set = set(adjectives) | set(nouns)
    word1_list = sorted(word1_set)

    # Word2 pool: nouns or plural nouns
    word2_list = list(plurals if ensure_plural else nouns)

    # Pre-measure word2 widths and sort
    word2_measured: list[tuple[float, str]] = []
    for w2 in word2_list:
        display2 = _apply_casing(w2, casing)
        w = font.getlength(display2)
        word2_measured.append((w, display2, w2))

    word2_measured.sort(key=lambda x: x[0])
    word2_widths = [x[0] for x in word2_measured]

    KERNING_MARGIN = 3.0  # extra margin for kerning between words

    for w1 in word1_list:
        if ks_lower and not w1.startswith(ks_lower):
            continue

        display1 = _apply_casing(w1, casing)

        # Measure word1 + space width (with left context kerning)
        w1_with_space = display1 + " "
        if left_context:
            w1_portion = (
                font.getlength(left_context + w1_with_space)
                - font.getlength(left_context)
            )
        else:
            w1_portion = font.getlength(w1_with_space)

        remaining = target_width - w1_portion
        if remaining < 0:
            continue

        # Binary search for word2 candidates
        lo = bisect.bisect_left(word2_widths, remaining - tolerance - KERNING_MARGIN)
        hi = bisect.bisect_right(word2_widths, remaining + tolerance + KERNING_MARGIN)

        for i in range(lo, hi):
            _, display2, w2_raw = word2_measured[i]
            if ke_lower and not w2_raw.endswith(ke_lower):
                continue

            phrase = display1 + " " + display2
            if phrase in seen:
                continue

            # Verify exact combined width with full kerning context
            if left_context or right_context:
                full = left_context + phrase + right_context
                full_len = font.getlength(full)
                left_len = font.getlength(left_context) if left_context else 0.0
                right_len = font.getlength(right_context) if right_context else 0.0
                exact_width = full_len - left_len - right_len
            else:
                exact_width = font.getlength(phrase)

            error = abs(exact_width - target_width)
            if error <= tolerance:
                seen.add(phrase)
                yield SolveResult(
                    text=phrase, width=float(exact_width), error=float(error)
                )
```

**Step 4: Run tests to verify they pass**

Run: `.venv/bin/python -m pytest tests/test_word_solve.py -v`
Expected: All tests PASS (both single-word and two-word).

**Step 5: Commit**

```bash
git add unredact/pipeline/dictionary.py tests/test_word_solve.py
git commit -m "feat: add two-word binary search solver (phase 2 of word mode)"
```

---

### Task 5: API Wiring

**Files:**
- Modify: `unredact/app.py:277-289` (SolveRequest model)
- Modify: `unredact/app.py:354-489` (solve endpoint)

**Step 1: Add `ensure_plural` to SolveRequest**

In `unredact/app.py`, find the `SolveRequest` class (line 277-288). Add after line 288 (`known_end: str = ""`):

```python
    ensure_plural: bool = False
```

**Step 2: Add word mode handler to the solve endpoint**

In the `event_generator` function inside `solve()` (around line 365-485), add a new block after the email mode block (after line 434) and before the enumerate mode block (line 437). Insert:

```python
            # Word mode: English nouns + adjective/noun phrases
            if req.mode == "word" and not _active_solves.get(solve_id):
                from unredact.pipeline.dictionary import solve_word_dictionary
                for r in solve_word_dictionary(
                    font, req.gap_width_px, req.tolerance_px,
                    req.left_context, req.right_context,
                    casing=charset_name,
                    known_start=req.known_start,
                    known_end=req.known_end,
                    ensure_plural=req.ensure_plural,
                ):
                    if _active_solves.get(solve_id):
                        break
                    if r.text in found_texts:
                        continue
                    found_texts.add(r.text)
                    yield json.dumps({
                        "status": "match",
                        "text": r.text,
                        "width_px": round(r.width, 2),
                        "error_px": round(r.error, 2),
                        "source": "words",
                    })
```

**Step 3: Run existing tests to check nothing broke**

Run: `.venv/bin/python -m pytest tests/test_dictionary.py tests/test_word_solve.py -v`
Expected: All PASS.

**Step 4: Commit**

```bash
git add unredact/app.py
git commit -m "feat: wire word mode and ensure_plural to /api/solve endpoint"
```

---

### Task 6: Frontend — HTML + DOM

**Files:**
- Modify: `unredact/static/index.html:98-103` (mode dropdown)
- Modify: `unredact/static/index.html:118-126` (add plural checkbox near filter label)
- Modify: `unredact/static/dom.js`

**Step 1: Add word option to mode dropdown**

In `index.html`, find the mode `<select>` (lines 98-103). Add a new option after `enumerate`:

```html
                  <option value="word">Word</option>
```

So it becomes:
```html
                <select id="solve-mode">
                  <option value="name" selected>Name</option>
                  <option value="full_name">Full Name</option>
                  <option value="email">Email</option>
                  <option value="enumerate">Enumerate</option>
                  <option value="word">Word</option>
                </select>
```

**Step 2: Add plural checkbox**

In `index.html`, after the filter label block (line 126), add:

```html
              <label id="plural-label" hidden>
                <input type="checkbox" id="solve-plural"> Ensure plural
              </label>
```

**Step 3: Add DOM references**

In `unredact/static/dom.js`, add after the `filterLabel` export (line 31):

```javascript
export const pluralLabel = document.getElementById("plural-label");
export const solvePlural = /** @type {HTMLInputElement} */ (document.getElementById("solve-plural"));
```

**Step 4: Verify by inspection**

Open the app in the browser and confirm:
- "Word" appears in the mode dropdown
- Plural checkbox is hidden by default

**Step 5: Commit**

```bash
git add unredact/static/index.html unredact/static/dom.js
git commit -m "feat: add Word mode option and plural checkbox to UI"
```

---

### Task 7: Frontend — Popover Logic + Solver Integration

**Files:**
- Modify: `unredact/static/popover.js:198-200` (mode change handler)
- Modify: `unredact/static/solver.js:40-54` (request body)

**Step 1: Update popover mode change handler**

In `unredact/static/popover.js`, find the `solveMode` change handler (lines 198-200):

```javascript
  solveMode.addEventListener("change", () => {
    filterLabel.hidden = solveMode.value !== "enumerate";
  });
```

Replace with:

```javascript
  solveMode.addEventListener("change", () => {
    filterLabel.hidden = solveMode.value !== "enumerate";
    pluralLabel.hidden = solveMode.value !== "word";
  });
```

Add `pluralLabel` to the import from `./dom.js` at the top of the file (line 14). Change:

```javascript
  solveMode, filterLabel,
```

to:

```javascript
  solveMode, filterLabel, pluralLabel, solvePlural,
```

**Step 2: Update solver.js to send ensure_plural**

In `unredact/static/solver.js`, find the import line (lines 6-10). Add `solvePlural` to the import from `./dom.js`. The imports should include `solvePlural`:

```javascript
import {
  solveCharset, solveTolerance, solveMode, solveFilter,
  solveKnownStart, solveKnownEnd, solvePlural,
  solveResults, solveStatus, solveStart, solveStop,
  solveAccept, redactionMarker, escapeHtml,
} from './dom.js';
```

Then in the `body` object (lines 40-54), add after `known_end`:

```javascript
    ensure_plural: solvePlural.checked,
```

**Step 3: Verify end-to-end**

Start the app with `make run`. Open a document with a redaction. Select "Word" mode. Click Solve. Confirm:
- Single-word nouns appear as results.
- Two-word phrases (adjective + noun) appear.
- Toggling "Ensure plural" filters to plural forms.

**Step 4: Commit**

```bash
git add unredact/static/popover.js unredact/static/solver.js
git commit -m "feat: connect word mode UI controls to solver backend"
```

---

### Task 8: Add .gitignore Entry + Final Verification

**Files:**
- Modify: `.gitignore` (if needed — ensure data files aren't gitignored)

**Step 1: Verify data files are tracked**

Run:
```bash
git ls-files unredact/data/nouns.txt unredact/data/nouns_plural.txt unredact/data/adjectives.txt
```

If empty (files are gitignored), check `.gitignore` and add an exception if needed.

**Step 2: Run full test suite**

Run: `.venv/bin/python -m pytest tests/test_dictionary.py tests/test_word_lists.py tests/test_word_solve.py -v`
Expected: All tests PASS.

**Step 3: Final commit (if any changes)**

```bash
git add -A && git commit -m "chore: ensure word list data files are tracked"
```
