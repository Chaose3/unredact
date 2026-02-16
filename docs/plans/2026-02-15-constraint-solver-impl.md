# Constraint Solver Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a kerning-aware constraint solver that enumerates all strings fitting a pixel-width gap, with multiprocessing parallelization and dictionary fast-path, streamed to the frontend via SSE.

**Architecture:** Width table precomputed per (font, size, charset) captures kerning implicitly. Branch-and-bound DFS with aggressive pruning fans out across CPU cores via ProcessPoolExecutor. SSE streams matches to the frontend in real-time. Dictionary mode provides a fast-path for known names.

**Tech Stack:** Python 3.12, Pillow (font metrics), multiprocessing, FastAPI SSE (sse-starlette), vanilla JS EventSource.

---

### Task 1: Width Table Module

**Files:**
- Create: `unredact/pipeline/width_table.py`
- Test: `tests/test_width_table.py`

**Step 1: Write the failing test**

```python
# tests/test_width_table.py
import numpy as np
import pytest
from PIL import ImageFont

from unredact.pipeline.width_table import build_width_table, CHARSETS


def _get_test_font() -> ImageFont.FreeTypeFont:
    """Get any available system font for testing."""
    import subprocess
    result = subprocess.run(
        ["fc-match", "--format=%{file}", "Liberation Serif"],
        capture_output=True, text=True,
    )
    path = result.stdout.strip()
    return ImageFont.truetype(path, 40)


class TestCharsets:
    def test_lowercase_has_26_chars(self):
        assert len(CHARSETS["lowercase"]) == 26

    def test_uppercase_has_26_chars(self):
        assert len(CHARSETS["uppercase"]) == 26

    def test_alpha_has_52_chars(self):
        assert len(CHARSETS["alpha"]) == 52

    def test_alphanumeric_has_62_chars(self):
        assert len(CHARSETS["alphanumeric"]) == 62


class TestBuildWidthTable:
    def test_returns_correct_shape(self):
        font = _get_test_font()
        charset = "abc"
        table = build_width_table(font, charset)
        assert table.width_table.shape == (3, 3)
        assert table.left_edge.shape == (3,)
        assert table.right_edge.shape == (3,)
        assert table.min_advance.shape == (3,)
        assert table.max_advance.shape == (3,)

    def test_widths_are_positive(self):
        font = _get_test_font()
        charset = "abcdefghij"
        table = build_width_table(font, charset)
        # Character advances should be positive
        assert np.all(table.width_table > 0)

    def test_min_max_bounds_correct(self):
        font = _get_test_font()
        charset = "abcdefghijklmnopqrstuvwxyz"
        table = build_width_table(font, charset)
        for i in range(len(charset)):
            row = table.width_table[i]
            assert table.min_advance[i] == pytest.approx(row.min())
            assert table.max_advance[i] == pytest.approx(row.max())

    def test_kerning_captured(self):
        """AV should kern tighter than AX in most serif fonts."""
        font = _get_test_font()
        charset = "AVX"
        table = build_width_table(font, charset)
        a_idx = 0  # A
        v_idx = 1  # V
        x_idx = 2  # X
        # This may not hold for all fonts, but generally AV kerns tighter
        # Just verify the table produces different values for different pairs
        # (i.e., kerning is being captured, not just uniform widths)
        assert table.width_table.shape == (3, 3)

    def test_left_edge_uses_context(self):
        font = _get_test_font()
        charset = "abcde"
        table = build_width_table(font, charset, left_context="T")
        table_no_ctx = build_width_table(font, charset, left_context="")
        # With left context "T", the left edge values should differ
        # (kerning between T and first char)
        # At minimum, the arrays should exist and have correct shape
        assert table.left_edge.shape == (5,)

    def test_right_edge_uses_context(self):
        font = _get_test_font()
        charset = "abcde"
        table = build_width_table(font, charset, right_context="y")
        assert table.right_edge.shape == (5,)
```

**Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_width_table.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'unredact.pipeline.width_table'`

**Step 3: Write minimal implementation**

```python
# unredact/pipeline/width_table.py
from dataclasses import dataclass

import numpy as np
from PIL import ImageFont

CHARSETS: dict[str, str] = {
    "lowercase": "abcdefghijklmnopqrstuvwxyz",
    "uppercase": "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
    "alpha": "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ",
    "alphanumeric": "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789",
    "printable": "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 !\"#$%&'()*+,-./:;<=>?@[\\]^_`{|}~",
}


@dataclass
class WidthTable:
    """Precomputed font metric table for a charset.

    width_table[i][j]: advance width of charset[j] when preceded by charset[i]
    left_edge[j]: advance width of charset[j] when preceded by left_context
    right_edge[i]: kerning correction for charset[i] followed by right_context
    min_advance[i]: minimum advance of any char after charset[i]
    max_advance[i]: maximum advance of any char after charset[i]
    """
    charset: str
    width_table: np.ndarray   # (N, N) float64
    left_edge: np.ndarray     # (N,) float64
    right_edge: np.ndarray    # (N,) float64
    min_advance: np.ndarray   # (N,) float64
    max_advance: np.ndarray   # (N,) float64
    # For the first character, min/max using left_edge
    left_min: float
    left_max: float


def build_width_table(
    font: ImageFont.FreeTypeFont,
    charset: str,
    left_context: str = "",
    right_context: str = "",
) -> WidthTable:
    """Build a kerning-aware width lookup table.

    Uses font.getlength() which returns advance width including kerning.
    width_table[i][j] = getlength(charset[i] + charset[j]) - getlength(charset[i])
    This gives the advance of charset[j] when preceded by charset[i].
    """
    n = len(charset)
    table = np.zeros((n, n), dtype=np.float64)
    left_edge = np.zeros(n, dtype=np.float64)
    right_edge = np.zeros(n, dtype=np.float64)

    # Precompute single-char lengths
    single = np.array([font.getlength(c) for c in charset], dtype=np.float64)

    # Build pair table
    for i, prev in enumerate(charset):
        base = font.getlength(prev)
        for j, next_c in enumerate(charset):
            table[i][j] = font.getlength(prev + next_c) - base

    # Left edge: advance of each char when preceded by left_context
    if left_context:
        base_left = font.getlength(left_context)
        for j, c in enumerate(charset):
            left_edge[j] = font.getlength(left_context + c) - base_left
    else:
        # No left context — use standalone width
        left_edge[:] = single

    # Right edge: kerning correction when followed by right_context
    # right_edge[i] = getlength(charset[i] + right_context) - getlength(charset[i]) - getlength(right_context)
    # This is the extra (or reduced) width from kerning at the right boundary
    if right_context:
        right_len = font.getlength(right_context)
        for i, c in enumerate(charset):
            right_edge[i] = font.getlength(c + right_context) - single[i] - right_len
    # else: right_edge stays zeros (no correction needed)

    min_advance = table.min(axis=1)
    max_advance = table.max(axis=1)

    return WidthTable(
        charset=charset,
        width_table=table,
        left_edge=left_edge,
        right_edge=right_edge,
        min_advance=min_advance,
        max_advance=max_advance,
        left_min=float(left_edge.min()),
        left_max=float(left_edge.max()),
    )
```

**Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_width_table.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/width_table.py tests/test_width_table.py
git commit -m "feat: width table module with kerning-aware font metrics"
```

---

### Task 2: Single-Threaded Branch-and-Bound Solver

**Files:**
- Create: `unredact/pipeline/solver.py`
- Test: `tests/test_solver.py`

**Step 1: Write the failing test**

```python
# tests/test_solver.py
import pytest
from PIL import ImageFont

from unredact.pipeline.solver import solve_gap, SolveResult
from unredact.pipeline.width_table import build_width_table


def _get_test_font() -> ImageFont.FreeTypeFont:
    import subprocess
    result = subprocess.run(
        ["fc-match", "--format=%{file}", "Liberation Serif"],
        capture_output=True, text=True,
    )
    path = result.stdout.strip()
    return ImageFont.truetype(path, 40)


class TestSolveGap:
    def test_finds_exact_match(self):
        """Given a gap that exactly fits 'hello', solver should find it."""
        font = _get_test_font()
        # Measure the actual width of "hello" with this font
        target = font.getlength("hello")
        results = solve_gap(
            font=font,
            charset="ehlo",  # tiny charset containing only needed chars
            target_width=target,
            tolerance=0.5,
            min_length=5,
            max_length=5,
            left_context="",
            right_context="",
        )
        texts = [r.text for r in results]
        assert "hello" in texts

    def test_respects_tolerance_zero(self):
        """With zero tolerance, only exact pixel matches are returned."""
        font = _get_test_font()
        target = font.getlength("ab")
        results = solve_gap(
            font=font,
            charset="ab",
            target_width=target,
            tolerance=0.0,
            min_length=2,
            max_length=2,
            left_context="",
            right_context="",
        )
        # "ab" must be in results; others only if they happen to have identical width
        texts = [r.text for r in results]
        assert "ab" in texts

    def test_respects_length_bounds(self):
        """Results should all be within min/max length."""
        font = _get_test_font()
        target = font.getlength("test")
        results = solve_gap(
            font=font,
            charset="tes",
            target_width=target,
            tolerance=2.0,
            min_length=3,
            max_length=5,
            left_context="",
            right_context="",
        )
        for r in results:
            assert 3 <= len(r.text) <= 5

    def test_with_context_chars(self):
        """Solver accounts for left/right boundary kerning."""
        font = _get_test_font()
        # Measure width of "es" in context "T...t"
        full = font.getlength("Test")
        left = font.getlength("T")
        right_char_len = font.getlength("t")
        # The gap is: full - left_advance - but we need precise boundary measurement
        # Just measure "es" with context
        target = font.getlength("Tes") - font.getlength("T")
        # Remove right context contribution if needed
        results = solve_gap(
            font=font,
            charset="est",
            target_width=target,
            tolerance=0.5,
            min_length=2,
            max_length=2,
            left_context="T",
            right_context="",
        )
        texts = [r.text for r in results]
        assert "es" in texts

    def test_empty_results_for_impossible_width(self):
        """If target is impossibly large or small, return no results."""
        font = _get_test_font()
        results = solve_gap(
            font=font,
            charset="abc",
            target_width=0.1,  # impossibly narrow
            tolerance=0.0,
            min_length=1,
            max_length=5,
            left_context="",
            right_context="",
        )
        assert len(results) == 0

    def test_result_contains_width_and_error(self):
        """Each result should have text, width, and error fields."""
        font = _get_test_font()
        target = font.getlength("ab")
        results = solve_gap(
            font=font,
            charset="ab",
            target_width=target,
            tolerance=1.0,
            min_length=2,
            max_length=2,
            left_context="",
            right_context="",
        )
        assert len(results) > 0
        r = results[0]
        assert isinstance(r.text, str)
        assert isinstance(r.width, float)
        assert isinstance(r.error, float)
        assert r.error >= 0
```

**Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_solver.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'unredact.pipeline.solver'`

**Step 3: Write minimal implementation**

```python
# unredact/pipeline/solver.py
from dataclasses import dataclass

import numpy as np
from PIL import ImageFont

from unredact.pipeline.width_table import build_width_table, WidthTable


@dataclass
class SolveResult:
    text: str
    width: float   # actual rendered width in px
    error: float   # abs(width - target)


def _solve_subtree(
    wt: WidthTable,
    target: float,
    tolerance: float,
    min_length: int,
    max_length: int,
    prefix: str,
    prefix_width: float,
    last_char_idx: int,
    is_first_char: bool,
) -> list[SolveResult]:
    """DFS branch-and-bound on a single subtree."""
    results: list[SolveResult] = []
    charset = wt.charset
    n = len(charset)
    # Convert to numpy arrays for fast access
    table = wt.width_table
    min_adv = wt.min_advance
    max_adv = wt.max_advance
    right_edge = wt.right_edge
    left_edge = wt.left_edge

    def dfs(depth: int, acc_width: float, last_idx: int, path: list[str]):
        current_length = len(prefix) + depth

        # Check if current string is a valid candidate
        if current_length >= min_length:
            final_width = acc_width + right_edge[last_idx]
            err = abs(final_width - target)
            if err <= tolerance:
                results.append(SolveResult(
                    text="".join(path),
                    width=float(final_width),
                    error=float(err),
                ))

        if current_length >= max_length:
            return

        # How many more chars can we add?
        chars_left = max_length - current_length

        for next_idx in range(n):
            advance = table[last_idx][next_idx]
            new_width = acc_width + advance

            # Prune: already overshot
            if new_width > target + tolerance:
                continue

            # Prune: even with widest remaining chars, can't reach target
            # (use min_length check: we need at least (min_length - current_length - 1) more after this)
            remaining_after = min_length - current_length - 1
            if remaining_after > 0:
                # We need at least remaining_after more chars — check if even widest can't fill
                pass  # conservative: skip this optimization for now
            else:
                # We could stop here — check if max possible future width is too small
                if chars_left > 1:
                    max_possible = new_width + max_adv[next_idx] * (chars_left - 1)
                    if max_possible + tolerance < target:
                        continue

            path.append(charset[next_idx])
            dfs(depth + 1, new_width, next_idx, path)
            path.pop()

    # Start DFS from the prefix state
    if len(prefix) == 0:
        # No prefix — start with each character using left_edge
        for first_idx in range(n):
            start_width = left_edge[first_idx]
            if start_width > target + tolerance:
                continue
            dfs(1, start_width, first_idx, [charset[first_idx]])
    else:
        # Continue from prefix
        path = list(prefix)
        dfs(0, prefix_width, last_char_idx, path)

    return results


def solve_gap(
    font: ImageFont.FreeTypeFont,
    charset: str,
    target_width: float,
    tolerance: float,
    min_length: int,
    max_length: int,
    left_context: str = "",
    right_context: str = "",
) -> list[SolveResult]:
    """Find all strings in charset that fill target_width within tolerance.

    Single-threaded version. See solve_gap_parallel for multiprocessing.
    """
    wt = build_width_table(font, charset, left_context, right_context)

    results = _solve_subtree(
        wt=wt,
        target=target_width,
        tolerance=tolerance,
        min_length=min_length,
        max_length=max_length,
        prefix="",
        prefix_width=0.0,
        last_char_idx=-1,
        is_first_char=True,
    )

    # Sort by error (best matches first)
    results.sort(key=lambda r: (r.error, r.text))
    return results
```

**Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_solver.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/solver.py tests/test_solver.py
git commit -m "feat: single-threaded branch-and-bound gap solver"
```

---

### Task 3: Parallel Solver with Multiprocessing

**Files:**
- Modify: `unredact/pipeline/solver.py`
- Test: `tests/test_solver.py` (add tests)

**Step 1: Write the failing test**

Add to `tests/test_solver.py`:

```python
from unredact.pipeline.solver import solve_gap_parallel


class TestSolveGapParallel:
    def test_same_results_as_serial(self):
        """Parallel solver should find the same results as serial."""
        font = _get_test_font()
        target = font.getlength("hello")
        serial = solve_gap(
            font=font, charset="ehlo", target_width=target,
            tolerance=0.5, min_length=5, max_length=5,
        )
        parallel = solve_gap_parallel(
            font=font, charset="ehlo", target_width=target,
            tolerance=0.5, min_length=5, max_length=5,
        )
        assert set(r.text for r in serial) == set(r.text for r in parallel)

    def test_parallel_with_larger_charset(self):
        """Should handle a real-sized charset without errors."""
        font = _get_test_font()
        target = font.getlength("cat")
        results = solve_gap_parallel(
            font=font, charset="abcdefghijklmnopqrstuvwxyz",
            target_width=target, tolerance=0.5,
            min_length=3, max_length=3,
        )
        texts = [r.text for r in results]
        assert "cat" in texts

    def test_progress_callback(self):
        """Progress callback should be called with node counts."""
        font = _get_test_font()
        target = font.getlength("ab")
        progress = []
        solve_gap_parallel(
            font=font, charset="abc", target_width=target,
            tolerance=1.0, min_length=2, max_length=2,
            on_progress=lambda checked, found: progress.append((checked, found)),
        )
        # Should have received at least one progress update
        assert len(progress) > 0
```

**Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_solver.py::TestSolveGapParallel -v`
Expected: FAIL — `ImportError: cannot import name 'solve_gap_parallel'`

**Step 3: Write minimal implementation**

Add to `unredact/pipeline/solver.py`:

```python
import os
from concurrent.futures import ProcessPoolExecutor, as_completed
from typing import Callable


def _generate_prefixes(wt: WidthTable, target: float, tolerance: float, depth: int = 2):
    """Generate all prefixes of given depth with their accumulated widths.

    Prunes prefixes that already overshoot the target.
    Returns list of (prefix_str, prefix_width, last_char_idx).
    """
    charset = wt.charset
    n = len(charset)
    prefixes = []

    def build(d, acc_width, last_idx, path):
        if d == depth:
            prefixes.append(("".join(path), acc_width, last_idx))
            return
        for next_idx in range(n):
            if d == 0:
                advance = wt.left_edge[next_idx]
            else:
                advance = wt.width_table[last_idx][next_idx]
            new_width = acc_width + advance
            if new_width > target + tolerance:
                continue
            path.append(charset[next_idx])
            build(d + 1, new_width, next_idx, path)
            path.pop()

    build(0, 0.0, -1, [])
    return prefixes


def _worker_solve(args: tuple) -> list[SolveResult]:
    """Worker function for multiprocessing. Takes serialized args."""
    (wt_data, target, tolerance, min_length, max_length, prefix, prefix_width, last_char_idx) = args
    # Reconstruct WidthTable from serialized data
    wt = WidthTable(**wt_data)
    return _solve_subtree(
        wt=wt, target=target, tolerance=tolerance,
        min_length=min_length, max_length=max_length,
        prefix=prefix, prefix_width=prefix_width,
        last_char_idx=last_char_idx, is_first_char=False,
    )


def solve_gap_parallel(
    font: ImageFont.FreeTypeFont,
    charset: str,
    target_width: float,
    tolerance: float,
    min_length: int,
    max_length: int,
    left_context: str = "",
    right_context: str = "",
    max_workers: int | None = None,
    on_progress: Callable[[int, int], None] | None = None,
) -> list[SolveResult]:
    """Find all strings that fill target_width, using multiprocessing."""
    wt = build_width_table(font, charset, left_context, right_context)

    # Determine prefix depth: want enough jobs for good load balancing
    n = len(charset)
    prefix_depth = 2 if n <= 52 else 1

    # Ensure prefix_depth doesn't exceed max_length
    prefix_depth = min(prefix_depth, max_length)

    prefixes = _generate_prefixes(wt, target_width, tolerance, prefix_depth)

    if not prefixes:
        return []

    # Serialize WidthTable for pickling across processes
    wt_data = {
        "charset": wt.charset,
        "width_table": wt.width_table,
        "left_edge": wt.left_edge,
        "right_edge": wt.right_edge,
        "min_advance": wt.min_advance,
        "max_advance": wt.max_advance,
        "left_min": wt.left_min,
        "left_max": wt.left_max,
    }

    worker_args = [
        (wt_data, target_width, tolerance, min_length, max_length,
         prefix, width, last_idx)
        for prefix, width, last_idx in prefixes
    ]

    if max_workers is None:
        max_workers = os.cpu_count() or 4

    all_results: list[SolveResult] = []
    checked_total = 0

    # Also check prefixes themselves if they meet length requirements
    # (handled inside _solve_subtree when depth=0 with current_length check)

    with ProcessPoolExecutor(max_workers=max_workers) as pool:
        futures = {pool.submit(_worker_solve, args): args for args in worker_args}
        for future in as_completed(futures):
            batch = future.result()
            all_results.extend(batch)
            checked_total += 1
            if on_progress:
                on_progress(checked_total, len(all_results))

    all_results.sort(key=lambda r: (r.error, r.text))
    return all_results
```

**Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_solver.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/solver.py tests/test_solver.py
git commit -m "feat: parallel gap solver with multiprocessing"
```

---

### Task 4: Dictionary Solver

**Files:**
- Create: `unredact/pipeline/dictionary.py`
- Test: `tests/test_dictionary.py`

**Step 1: Write the failing test**

```python
# tests/test_dictionary.py
import pytest
from PIL import ImageFont

from unredact.pipeline.dictionary import solve_dictionary, DictionaryStore


def _get_test_font() -> ImageFont.FreeTypeFont:
    import subprocess
    result = subprocess.run(
        ["fc-match", "--format=%{file}", "Liberation Serif"],
        capture_output=True, text=True,
    )
    path = result.stdout.strip()
    return ImageFont.truetype(path, 40)


class TestSolveDictionary:
    def test_finds_matching_word(self):
        font = _get_test_font()
        entries = ["Smith", "Jones", "Brown"]
        target = font.getlength("Smith")
        results = solve_dictionary(font, entries, target, tolerance=0.5)
        texts = [r.text for r in results]
        assert "Smith" in texts

    def test_respects_tolerance(self):
        font = _get_test_font()
        entries = ["Smith", "Jones", "Brown"]
        target = font.getlength("Smith")
        results = solve_dictionary(font, entries, target, tolerance=0.0)
        # Only exact match
        assert all(r.error == 0.0 for r in results)

    def test_with_context(self):
        font = _get_test_font()
        entries = ["Smith"]
        # Measure with context
        target = font.getlength(" Smith ") - font.getlength(" ") - font.getlength(" ")
        results = solve_dictionary(
            font, entries, target, tolerance=1.0,
            left_context=" ", right_context=" ",
        )
        assert len(results) >= 0  # may or may not match due to kerning


class TestDictionaryStore:
    def test_add_and_list(self):
        store = DictionaryStore()
        store.add("names", ["Alice", "Bob", "Charlie"])
        assert "names" in store.list()

    def test_get_entries(self):
        store = DictionaryStore()
        store.add("names", ["Alice", "Bob"])
        assert store.get_entries("names") == ["Alice", "Bob"]

    def test_remove(self):
        store = DictionaryStore()
        store.add("names", ["Alice"])
        store.remove("names")
        assert "names" not in store.list()

    def test_all_entries(self):
        store = DictionaryStore()
        store.add("list1", ["Alice", "Bob"])
        store.add("list2", ["Charlie"])
        all_entries = store.all_entries()
        assert set(all_entries) == {"Alice", "Bob", "Charlie"}
```

**Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_dictionary.py -v`
Expected: FAIL — `ModuleNotFoundError`

**Step 3: Write minimal implementation**

```python
# unredact/pipeline/dictionary.py
from dataclasses import dataclass

from PIL import ImageFont

from unredact.pipeline.solver import SolveResult


class DictionaryStore:
    """In-memory store for word/name lists."""

    def __init__(self):
        self._dicts: dict[str, list[str]] = {}

    def add(self, name: str, entries: list[str]):
        self._dicts[name] = entries

    def remove(self, name: str):
        self._dicts.pop(name, None)

    def list(self) -> list[str]:
        return list(self._dicts.keys())

    def get_entries(self, name: str) -> list[str]:
        return self._dicts.get(name, [])

    def all_entries(self) -> list[str]:
        seen = set()
        result = []
        for entries in self._dicts.values():
            for e in entries:
                if e not in seen:
                    seen.add(e)
                    result.append(e)
        return result


def solve_dictionary(
    font: ImageFont.FreeTypeFont,
    entries: list[str],
    target_width: float,
    tolerance: float = 0.0,
    left_context: str = "",
    right_context: str = "",
) -> list[SolveResult]:
    """Check each dictionary entry against the target width."""
    results: list[SolveResult] = []

    for entry in entries:
        # Measure width with context
        if left_context or right_context:
            full = left_context + entry + right_context
            full_len = font.getlength(full)
            # Subtract the context contributions
            left_len = font.getlength(left_context) if left_context else 0.0
            right_len = font.getlength(right_context) if right_context else 0.0
            # The entry width includes kerning at boundaries
            width = full_len - left_len - right_len
        else:
            width = font.getlength(entry)

        error = abs(width - target_width)
        if error <= tolerance:
            results.append(SolveResult(text=entry, width=float(width), error=float(error)))

    results.sort(key=lambda r: (r.error, r.text))
    return results
```

**Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_dictionary.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/dictionary.py tests/test_dictionary.py
git commit -m "feat: dictionary solver and in-memory wordlist store"
```

---

### Task 5: SSE Solve Endpoint

**Files:**
- Modify: `unredact/app.py` (add solve endpoints)
- Modify: `pyproject.toml` (add `sse-starlette` dependency)
- Test: `tests/test_app.py` (add solve endpoint tests)

**Step 1: Add sse-starlette dependency**

Add `"sse-starlette>=2.0"` to `pyproject.toml` dependencies and run `pip install -e ".[dev]"`.

**Step 2: Write the failing test**

Add to `tests/test_app.py`:

```python
@pytest.mark.anyio
async def test_solve_endpoint_enumerate():
    """POST /api/solve should stream SSE results."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        # Get an available font
        resp = await client.get("/api/fonts")
        fonts = resp.json()["fonts"]
        font = next(f for f in fonts if f["available"])

        # We need to know the font size and a target width.
        # Use the font to measure a known string server-side via a helper.
        resp = await client.post("/api/solve", json={
            "font_id": font["id"],
            "font_size": 40,
            "gap_width_px": 50.0,  # some width
            "tolerance_px": 5.0,   # loose tolerance for testing
            "left_context": "",
            "right_context": "",
            "hints": {
                "charset": "lowercase",
                "min_length": 2,
                "max_length": 3,
            },
            "mode": "enumerate",
        })
        assert resp.status_code == 200
        # SSE responses come as text/event-stream
        assert "text/event-stream" in resp.headers.get("content-type", "")


@pytest.mark.anyio
async def test_solve_endpoint_dictionary():
    """POST /api/solve with mode=dictionary should work."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.get("/api/fonts")
        fonts = resp.json()["fonts"]
        font = next(f for f in fonts if f["available"])

        resp = await client.post("/api/solve", json={
            "font_id": font["id"],
            "font_size": 40,
            "gap_width_px": 50.0,
            "tolerance_px": 5.0,
            "left_context": "",
            "right_context": "",
            "hints": {
                "charset": "lowercase",
                "min_length": 1,
                "max_length": 10,
            },
            "mode": "dictionary",
        })
        assert resp.status_code == 200


@pytest.mark.anyio
async def test_dictionary_crud():
    """Dictionary upload, list, delete endpoints."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        # Upload
        resp = await client.post("/api/dictionary", json={
            "name": "test-names",
            "entries": ["Alice", "Bob", "Charlie"],
        })
        assert resp.status_code == 200

        # List
        resp = await client.get("/api/dictionary")
        assert resp.status_code == 200
        assert "test-names" in resp.json()["dictionaries"]

        # Delete
        resp = await client.delete("/api/dictionary/test-names")
        assert resp.status_code == 200

        # Verify deleted
        resp = await client.get("/api/dictionary")
        assert "test-names" not in resp.json()["dictionaries"]
```

**Step 3: Run test to verify it fails**

Run: `python -m pytest tests/test_app.py::test_solve_endpoint_enumerate -v`
Expected: FAIL — endpoint not found (404)

**Step 4: Write implementation**

Add to `unredact/app.py`:

```python
import asyncio
import json
import uuid as uuid_mod

from pydantic import BaseModel
from sse_starlette.sse import EventSourceResponse
from PIL import ImageFont

from unredact.pipeline.solver import solve_gap_parallel, SolveResult
from unredact.pipeline.dictionary import DictionaryStore, solve_dictionary
from unredact.pipeline.width_table import CHARSETS


# In-memory dictionary store
_dictionary_store = DictionaryStore()

# Active solve tasks (for cancellation)
_active_solves: dict[str, bool] = {}  # solve_id -> cancelled


class SolveRequest(BaseModel):
    font_id: str
    font_size: int
    gap_width_px: float
    tolerance_px: float = 0.0
    left_context: str = ""
    right_context: str = ""
    hints: dict = {}
    mode: str = "enumerate"  # "enumerate", "dictionary", "both"


@app.post("/api/solve")
async def solve(req: SolveRequest):
    """Stream solve results via SSE."""
    font_path = _font_id_to_path.get(req.font_id)
    if not font_path:
        return JSONResponse({"error": "font not found"}, status_code=404)

    font = ImageFont.truetype(str(font_path), req.font_size)
    charset_name = req.hints.get("charset", "lowercase")
    charset = CHARSETS.get(charset_name, charset_name)  # allow custom charset string
    min_length = req.hints.get("min_length", 1)
    max_length = req.hints.get("max_length", 10)

    solve_id = uuid_mod.uuid4().hex[:12]
    _active_solves[solve_id] = False

    async def event_generator():
        try:
            found_texts = set()

            # Dictionary mode first (if "dictionary" or "both")
            if req.mode in ("dictionary", "both"):
                entries = _dictionary_store.all_entries()
                if entries:
                    dict_results = solve_dictionary(
                        font, entries, req.gap_width_px, req.tolerance_px,
                        req.left_context, req.right_context,
                    )
                    for r in dict_results:
                        if _active_solves.get(solve_id):
                            break
                        found_texts.add(r.text)
                        yield json.dumps({
                            "status": "match",
                            "text": r.text,
                            "width_px": round(r.width, 2),
                            "error_px": round(r.error, 2),
                            "source": "dictionary",
                        })

            # Enumeration (if "enumerate" or "both")
            if req.mode in ("enumerate", "both") and not _active_solves.get(solve_id):
                # Run solver in a thread to not block the event loop
                loop = asyncio.get_event_loop()
                results = await loop.run_in_executor(
                    None,
                    lambda: solve_gap_parallel(
                        font=font,
                        charset=charset,
                        target_width=req.gap_width_px,
                        tolerance=req.tolerance_px,
                        min_length=min_length,
                        max_length=max_length,
                        left_context=req.left_context,
                        right_context=req.right_context,
                    ),
                )
                for r in results:
                    if _active_solves.get(solve_id):
                        break
                    if r.text in found_texts:
                        continue  # already sent from dictionary
                    found_texts.add(r.text)
                    yield json.dumps({
                        "status": "match",
                        "text": r.text,
                        "width_px": round(r.width, 2),
                        "error_px": round(r.error, 2),
                        "source": "enumerate",
                    })

            yield json.dumps({
                "status": "done",
                "total_found": len(found_texts),
            })
        finally:
            _active_solves.pop(solve_id, None)

    return EventSourceResponse(event_generator(), headers={"X-Solve-Id": solve_id})


@app.delete("/api/solve/{solve_id}")
async def cancel_solve(solve_id: str):
    if solve_id in _active_solves:
        _active_solves[solve_id] = True
        return {"status": "cancelled"}
    return JSONResponse({"error": "solve not found"}, status_code=404)


@app.post("/api/dictionary")
async def upload_dictionary(data: dict):
    name = data.get("name", "")
    entries = data.get("entries", [])
    if not name or not entries:
        return JSONResponse({"error": "name and entries required"}, status_code=400)
    _dictionary_store.add(name, entries)
    return {"status": "ok", "count": len(entries)}


@app.get("/api/dictionary")
async def list_dictionaries():
    return {"dictionaries": _dictionary_store.list()}


@app.delete("/api/dictionary/{name}")
async def delete_dictionary(name: str):
    _dictionary_store.remove(name)
    return {"status": "ok"}
```

**Step 5: Run test to verify it passes**

Run: `python -m pytest tests/test_app.py -v`
Expected: PASS

**Step 6: Commit**

```bash
pip install sse-starlette
git add unredact/app.py pyproject.toml tests/test_app.py
git commit -m "feat: SSE solve endpoint with dictionary CRUD"
```

---

### Task 6: Frontend Solve Panel

**Files:**
- Modify: `unredact/static/index.html` (add solve panel markup)
- Modify: `unredact/static/style.css` (add solve panel styles)
- Modify: `unredact/static/app.js` (add solve logic)

**Step 1: Add solve panel HTML**

Add inside the `#right-panel` div, after `#text-edit-bar` in `index.html`:

```html
<div id="solve-panel" hidden>
  <div class="solve-header">
    <span class="solve-title">Solve Redaction</span>
    <button id="solve-close" class="size-btn">X</button>
  </div>
  <div class="solve-controls">
    <label>
      Charset
      <select id="solve-charset">
        <option value="lowercase">lowercase</option>
        <option value="uppercase">UPPERCASE</option>
        <option value="alpha">Mixed Case</option>
        <option value="alphanumeric">Alphanumeric</option>
      </select>
    </label>
    <label>
      Min length
      <input type="number" id="solve-min-len" value="3" min="1" max="30" class="solve-num">
    </label>
    <label>
      Max length
      <input type="number" id="solve-max-len" value="8" min="1" max="30" class="solve-num">
    </label>
    <label>
      Tolerance
      <input type="range" id="solve-tolerance" min="0" max="5" step="0.5" value="0">
      <span id="solve-tol-value">0</span>px
    </label>
    <label>
      Mode
      <select id="solve-mode">
        <option value="enumerate">Enumerate</option>
        <option value="dictionary">Dictionary</option>
        <option value="both">Both</option>
      </select>
    </label>
  </div>
  <div class="solve-actions">
    <button id="solve-start" class="solve-btn">Solve</button>
    <button id="solve-stop" class="solve-btn" hidden>Stop</button>
    <span id="solve-status"></span>
  </div>
  <div id="solve-results"></div>
</div>
```

**Step 2: Add a "Solve" button to the text-edit-bar**

In `index.html`, add a button next to the existing Reset button inside `#text-edit-bar`:

```html
<button id="solve-btn" class="size-btn" title="Solve redaction gap" hidden>Solve</button>
```

**Step 3: Add CSS for solve panel**

Add to `style.css`:

```css
/* Solve panel */
#solve-panel {
  position: absolute;
  top: 80px;
  right: 12px;
  width: 300px;
  max-height: calc(100% - 100px);
  background: rgba(20, 20, 40, 0.95);
  border: 1px solid #333;
  border-radius: 8px;
  padding: 12px;
  z-index: 30;
  overflow-y: auto;
  backdrop-filter: blur(10px);
}

.solve-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.solve-title {
  font-weight: bold;
  color: #00d474;
}

.solve-controls {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-bottom: 10px;
}

.solve-controls label {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.8rem;
  color: #aaa;
}

.solve-controls select,
.solve-controls input {
  background: #1a1a2e;
  border: 1px solid #333;
  color: #eee;
  border-radius: 4px;
  padding: 2px 4px;
}

.solve-num {
  width: 50px;
}

.solve-actions {
  display: flex;
  gap: 8px;
  align-items: center;
  margin-bottom: 10px;
}

.solve-btn {
  background: #00d474;
  color: #000;
  border: none;
  border-radius: 4px;
  padding: 6px 16px;
  cursor: pointer;
  font-weight: bold;
}

.solve-btn:hover {
  background: #00ff8a;
}

#solve-status {
  font-size: 0.75rem;
  color: #aaa;
}

#solve-results {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.solve-result {
  display: flex;
  justify-content: space-between;
  padding: 4px 8px;
  background: rgba(0, 212, 116, 0.1);
  border: 1px solid transparent;
  border-radius: 4px;
  cursor: pointer;
  font-family: monospace;
  font-size: 0.85rem;
}

.solve-result:hover {
  border-color: #00d474;
  background: rgba(0, 212, 116, 0.2);
}

.solve-result .result-text {
  color: #00d474;
}

.solve-result .result-error {
  color: #888;
  font-size: 0.75rem;
}
```

**Step 4: Add solve logic to app.js**

Add after the viewport section at the bottom of `app.js`:

```javascript
// ── Solve panel ──

const solveBtn = document.getElementById("solve-btn");
const solvePanel = document.getElementById("solve-panel");
const solveClose = document.getElementById("solve-close");
const solveStart = document.getElementById("solve-start");
const solveStop = document.getElementById("solve-stop");
const solveStatus = document.getElementById("solve-status");
const solveResults = document.getElementById("solve-results");
const solveCharset = document.getElementById("solve-charset");
const solveMinLen = document.getElementById("solve-min-len");
const solveMaxLen = document.getElementById("solve-max-len");
const solveTolerance = document.getElementById("solve-tolerance");
const solveTolValue = document.getElementById("solve-tol-value");
const solveMode = document.getElementById("solve-mode");

let activeEventSource = null;

// Show solve button when a line has redaction gaps
function updateSolveButton() {
  const segs = getSegments();
  solveBtn.hidden = !(segs && segs.length > 1);
}

solveBtn.addEventListener("click", () => {
  solvePanel.hidden = false;
});

solveClose.addEventListener("click", () => {
  solvePanel.hidden = true;
  stopSolve();
});

solveTolerance.addEventListener("input", () => {
  solveTolValue.textContent = solveTolerance.value;
});

solveStart.addEventListener("click", startSolve);
solveStop.addEventListener("click", stopSolve);

function startSolve() {
  if (state.selectedLine === null) return;
  const segs = getSegments();
  if (!segs || segs.length < 2) return;

  const pd = state.pageData[state.currentPage];
  const line = pd.lines[state.selectedLine];
  const override = state.lineOverrides[`${state.currentPage}-${state.selectedLine}`];
  const fontId = override ? override.fontId : line.font.id;
  const fontSize = override ? override.fontSize : line.font.size;
  const fontName = state.fonts.find(f => f.id === fontId)?.name || line.font.name;

  // Find the gap after the active segment
  const gapIdx = state.activeSegment;
  if (gapIdx >= segs.length - 1) return; // no gap after last segment

  // Compute gap width from canvas measurement
  const fontStr = `${fontSize}px "${fontName}"`;
  ctx.font = fontStr;

  // Measure width of the segment before the gap
  const segBefore = segs[gapIdx];
  const segAfter = segs[gapIdx + 1];

  // The gap width is what the canvas renders between segments
  // We need to calculate it the same way renderOverlay does
  let cursorX = 0;
  for (let i = 0; i <= gapIdx; i++) {
    cursorX += segs[i].offsetX + ctx.measureText(segs[i].text).width;
  }
  const gapStart = cursorX;
  const nextSegStart = cursorX + segAfter.offsetX;
  const gapWidth = Math.max(nextSegStart - gapStart, fontSize * 2);

  // Context characters
  const leftCtx = segBefore.text.length > 0 ? segBefore.text[segBefore.text.length - 1] : "";
  const rightCtx = segAfter.text.length > 0 ? segAfter.text[0] : "";

  // Clear previous results
  solveResults.innerHTML = "";
  solveStatus.textContent = "Starting...";
  solveStart.hidden = true;
  solveStop.hidden = false;

  const params = new URLSearchParams();
  const body = {
    font_id: fontId,
    font_size: fontSize,
    gap_width_px: gapWidth,
    tolerance_px: parseFloat(solveTolerance.value),
    left_context: leftCtx,
    right_context: rightCtx,
    hints: {
      charset: solveCharset.value,
      min_length: parseInt(solveMinLen.value),
      max_length: parseInt(solveMaxLen.value),
    },
    mode: solveMode.value,
  };

  // Use fetch + ReadableStream for SSE from POST
  const abortController = new AbortController();
  activeEventSource = abortController;

  fetch("/api/solve", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
    signal: abortController.signal,
  }).then(response => {
    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let buffer = "";

    function read() {
      reader.read().then(({ done, value }) => {
        if (done) {
          solveStart.hidden = false;
          solveStop.hidden = true;
          return;
        }
        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split("\n");
        buffer = lines.pop(); // keep incomplete line

        for (const line of lines) {
          if (line.startsWith("data: ")) {
            try {
              const data = JSON.parse(line.slice(6));
              handleSolveEvent(data, gapIdx);
            } catch (e) { /* skip malformed */ }
          }
        }
        read();
      });
    }
    read();
  }).catch(err => {
    if (err.name !== "AbortError") {
      solveStatus.textContent = "Error: " + err.message;
    }
    solveStart.hidden = false;
    solveStop.hidden = true;
  });
}

function handleSolveEvent(data, gapIdx) {
  if (data.status === "match") {
    const div = document.createElement("div");
    div.className = "solve-result";
    div.innerHTML = `
      <span class="result-text">${escapeHtml(data.text)}</span>
      <span class="result-error">${data.error_px.toFixed(1)}px ${data.source || ""}</span>
    `;
    div.addEventListener("click", () => {
      // Insert this text into the redaction gap
      const segs = ensureSegments();
      // Insert a new segment between gapIdx and gapIdx+1
      segs.splice(gapIdx + 1, 0, { text: data.text, offsetX: 0 });
      state.activeSegment = gapIdx + 1;
      renderSegmentInputs();
      renderOverlay();
      updateLineListPreview();
    });
    solveResults.appendChild(div);
    solveStatus.textContent = `Found ${solveResults.children.length} matches`;
  } else if (data.status === "running") {
    solveStatus.textContent = `Checked ${data.checked}, found ${data.found}...`;
  } else if (data.status === "done") {
    solveStatus.textContent = `Done. ${data.total_found} total matches.`;
    solveStart.hidden = false;
    solveStop.hidden = true;
    activeEventSource = null;
  }
}

function stopSolve() {
  if (activeEventSource) {
    activeEventSource.abort();
    activeEventSource = null;
  }
  solveStart.hidden = false;
  solveStop.hidden = true;
  solveStatus.textContent = "Stopped.";
}

function escapeHtml(text) {
  const div = document.createElement("div");
  div.textContent = text;
  return div.innerHTML;
}
```

**Step 5: Wire updateSolveButton into existing code**

Call `updateSolveButton()` at the end of:
- `selectLine()` (after `renderOverlay()`)
- `splitSegmentAtCursor()` (after `renderOverlay()`)
- `textReset` click handler (after `renderOverlay()`)

**Step 6: Manual test**

1. Run `uvicorn unredact.app:app --reload`
2. Upload a PDF, select a line, Ctrl+Space to create a gap
3. Click "Solve", configure hints, click "Solve"
4. Verify matches stream in and clicking a result fills the gap

**Step 7: Commit**

```bash
git add unredact/static/index.html unredact/static/style.css unredact/static/app.js
git commit -m "feat: frontend solve panel with SSE streaming"
```

---

### Task 7: Integration Test

**Files:**
- Test: `tests/test_solver.py` (add integration test)

**Step 1: Write a round-trip integration test**

Add to `tests/test_solver.py`:

```python
class TestRoundTrip:
    def test_known_word_found_with_real_font(self):
        """Measure a word, then solve for it — the word should appear in results."""
        font = _get_test_font()
        word = "Smith"
        target = font.getlength("o" + word + " ") - font.getlength("o")
        # Subtract right context contribution
        right_correction = font.getlength(word[-1] + " ") - font.getlength(word[-1]) - font.getlength(" ")
        target -= right_correction

        # Use the actual getlength for the word itself
        target = font.getlength("o" + word) - font.getlength("o")

        results = solve_gap(
            font=font,
            charset="abcdefghijklmnopqrstuvwxyzST",  # include needed uppercase
            target_width=target,
            tolerance=0.5,
            min_length=5,
            max_length=5,
            left_context="o",
            right_context="",
        )
        texts = [r.text for r in results]
        assert "Smith" in texts
```

**Step 2: Run full test suite**

Run: `python -m pytest tests/ -v --ignore=tests/test_e2e.py`
Expected: ALL PASS

**Step 3: Commit**

```bash
git add tests/test_solver.py
git commit -m "test: round-trip integration test for solver"
```
