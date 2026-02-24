# Font Matching Debug Visualization — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** When `UNREDACT_DEBUG=1` is set, save visual debug images showing the top 5 font candidates per line with pixel overlap visualizations.

**Architecture:** New `font_debug.py` module handles all image generation. `_full_search()` in `font_detect.py` is modified to collect top-N candidates (not just the best). After search completes, debug functions re-render the top 5 and save composite images. Zero overhead when debug is off.

**Tech Stack:** PIL/Pillow for image generation, numpy for pixel operations (both already in use).

---

### Task 1: Add `debug/` to .gitignore

**Files:**
- Modify: `.gitignore`

**Step 1: Add debug directory to gitignore**

Append `debug/` to `.gitignore`.

**Step 2: Commit**

```bash
git add .gitignore
git commit -m "chore: gitignore debug/ directory"
```

---

### Task 2: Create `font_debug.py` with debug-enabled check and directory setup

**Files:**
- Create: `unredact/pipeline/font_debug.py`
- Test: `tests/test_font_debug.py`

**Step 1: Write tests for debug utilities**

```python
"""Tests for font matching debug utilities."""

import os
from pathlib import Path
from unittest.mock import patch

from unredact.pipeline.font_debug import debug_enabled, init_debug_dir


class TestDebugEnabled:
    def test_disabled_by_default(self):
        with patch.dict(os.environ, {}, clear=True):
            os.environ.pop("UNREDACT_DEBUG", None)
            assert debug_enabled() is False

    def test_enabled_with_env_var(self):
        with patch.dict(os.environ, {"UNREDACT_DEBUG": "1"}):
            assert debug_enabled() is True

    def test_disabled_with_zero(self):
        with patch.dict(os.environ, {"UNREDACT_DEBUG": "0"}):
            assert debug_enabled() is False


class TestInitDebugDir:
    def test_creates_timestamped_dir(self, tmp_path):
        result = init_debug_dir(base=tmp_path)
        assert result.exists()
        assert result.parent == tmp_path
        assert result.name.startswith("font-match-")

    def test_returns_path(self, tmp_path):
        result = init_debug_dir(base=tmp_path)
        assert isinstance(result, Path)
```

**Step 2: Run tests to verify they fail**

Run: `pytest tests/test_font_debug.py -v`
Expected: FAIL — module does not exist

**Step 3: Write minimal implementation**

```python
"""Font matching debug visualization.

When UNREDACT_DEBUG=1, saves visual comparisons of font candidates
to debug/font-match-<timestamp>/ in the project root.
"""

import os
from datetime import datetime
from pathlib import Path

PROJECT_ROOT = Path(__file__).parent.parent.parent


def debug_enabled() -> bool:
    return os.environ.get("UNREDACT_DEBUG", "0") == "1"


def init_debug_dir(base: Path | None = None) -> Path:
    if base is None:
        base = PROJECT_ROOT / "debug"
    stamp = datetime.now().strftime("%Y-%m-%dT%H-%M-%S")
    d = base / f"font-match-{stamp}"
    d.mkdir(parents=True, exist_ok=True)
    return d
```

**Step 4: Run tests to verify they pass**

Run: `pytest tests/test_font_debug.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/font_debug.py tests/test_font_debug.py
git commit -m "feat(debug): add font_debug module with env var check and dir setup"
```

---

### Task 3: Implement candidate image generation

**Files:**
- Modify: `unredact/pipeline/font_debug.py`
- Test: `tests/test_font_debug.py`

**Step 1: Write test for composite image generation**

Add to `tests/test_font_debug.py`:

```python
import numpy as np
from PIL import Image

from unredact.pipeline.font_debug import render_candidate_composite


class TestRenderCandidateComposite:
    def test_returns_rgb_image(self):
        page_bin = np.zeros((20, 100), dtype=bool)
        rendered_bin = np.zeros((20, 100), dtype=bool)
        result = render_candidate_composite(
            page_bin, rendered_bin,
            font_name="Arial", font_size=14, score=0.85, rank=1,
        )
        assert isinstance(result, Image.Image)
        assert result.mode == "RGB"

    def test_image_wider_than_crop(self):
        page_bin = np.zeros((20, 100), dtype=bool)
        rendered_bin = np.zeros((20, 100), dtype=bool)
        result = render_candidate_composite(
            page_bin, rendered_bin,
            font_name="Arial", font_size=14, score=0.85, rank=1,
        )
        # Must include header + 3 rows (page, rendered, overlap)
        assert result.height > 20 * 3

    def test_overlap_colors(self):
        # Create known overlap: both have ink at (5, 50)
        page_bin = np.zeros((20, 100), dtype=bool)
        rendered_bin = np.zeros((20, 100), dtype=bool)
        page_bin[5, 50] = True
        rendered_bin[5, 50] = True
        # Page-only ink at (10, 50)
        page_bin[10, 50] = True
        # Rendered-only ink at (15, 50)
        rendered_bin[15, 50] = True

        result = render_candidate_composite(
            page_bin, rendered_bin,
            font_name="Test", font_size=12, score=0.5, rank=1,
        )
        assert result is not None
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_font_debug.py::TestRenderCandidateComposite -v`
Expected: FAIL — function not defined

**Step 3: Implement `render_candidate_composite()`**

Add to `unredact/pipeline/font_debug.py`:

```python
import numpy as np
from PIL import Image, ImageDraw, ImageFont as PilFont


# Row label height + padding
_HEADER_H = 20
_LABEL_W = 60
_PAD = 2


def render_candidate_composite(
    page_bin: np.ndarray,
    rendered_bin: np.ndarray,
    font_name: str,
    font_size: int,
    score: float,
    rank: int,
) -> Image.Image:
    """Build a composite debug image for one font candidate.

    Layout (vertical stack):
    - Header: rank, font name, size, score
    - Row 1: page crop (white ink on black)
    - Row 2: rendered candidate (white ink on black)
    - Row 3: overlap map (green=both, red=page-only, blue=rendered-only)
    """
    h, w = page_bin.shape

    row_h = h + _PAD * 2
    total_h = _HEADER_H + row_h * 3
    img = Image.new("RGB", (w, total_h), (30, 30, 30))
    draw = ImageDraw.Draw(img)

    # Header
    header = f"#{rank}  {font_name}  {font_size}px  score={score:.3f}"
    draw.text((4, 2), header, fill=(255, 255, 255))

    # Row 1: page crop
    y_off = _HEADER_H + _PAD
    page_rgb = np.zeros((h, w, 3), dtype=np.uint8)
    page_rgb[page_bin] = [255, 255, 255]
    img.paste(Image.fromarray(page_rgb), (0, y_off))

    # Row 2: rendered candidate
    y_off += row_h
    rend_rgb = np.zeros((h, w, 3), dtype=np.uint8)
    rend_rgb[rendered_bin] = [255, 255, 255]
    img.paste(Image.fromarray(rend_rgb), (0, y_off))

    # Row 3: overlap map
    y_off += row_h
    overlap_rgb = np.zeros((h, w, 3), dtype=np.uint8)
    both = page_bin & rendered_bin
    page_only = page_bin & ~rendered_bin
    rend_only = ~page_bin & rendered_bin
    overlap_rgb[both] = [0, 200, 0]       # green = match
    overlap_rgb[page_only] = [220, 50, 50] # red = missing from render
    overlap_rgb[rend_only] = [50, 100, 220] # blue = extra in render
    img.paste(Image.fromarray(overlap_rgb), (0, y_off))

    return img
```

**Step 4: Run tests to verify they pass**

Run: `pytest tests/test_font_debug.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/font_debug.py tests/test_font_debug.py
git commit -m "feat(debug): implement candidate composite image generation"
```

---

### Task 4: Implement summary image (top 5 tiled)

**Files:**
- Modify: `unredact/pipeline/font_debug.py`
- Test: `tests/test_font_debug.py`

**Step 1: Write test for summary tiling**

Add to `tests/test_font_debug.py`:

```python
from unredact.pipeline.font_debug import render_summary_image


class TestRenderSummaryImage:
    def test_tiles_horizontally(self):
        imgs = [Image.new("RGB", (100, 80), (i * 50, 0, 0)) for i in range(3)]
        result = render_summary_image(imgs)
        assert result.width == 100 * 3 + 2 * 2  # 2px gap between
        assert result.height == 80

    def test_single_image(self):
        imgs = [Image.new("RGB", (100, 80))]
        result = render_summary_image(imgs)
        assert result.width == 100
        assert result.height == 80

    def test_handles_different_heights(self):
        imgs = [Image.new("RGB", (100, 60)), Image.new("RGB", (100, 80))]
        result = render_summary_image(imgs)
        assert result.height == 80  # tallest
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_font_debug.py::TestRenderSummaryImage -v`
Expected: FAIL

**Step 3: Implement `render_summary_image()`**

Add to `unredact/pipeline/font_debug.py`:

```python
_GAP = 2  # pixels between tiled images


def render_summary_image(composites: list[Image.Image]) -> Image.Image:
    """Tile candidate composites horizontally into a single summary image."""
    if not composites:
        return Image.new("RGB", (1, 1))
    max_h = max(img.height for img in composites)
    total_w = sum(img.width for img in composites) + _GAP * (len(composites) - 1)
    summary = Image.new("RGB", (total_w, max_h), (20, 20, 20))
    x = 0
    for img in composites:
        summary.paste(img, (x, 0))
        x += img.width + _GAP
    return summary
```

**Step 4: Run tests**

Run: `pytest tests/test_font_debug.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/font_debug.py tests/test_font_debug.py
git commit -m "feat(debug): implement summary image tiling"
```

---

### Task 5: Modify `_full_search()` to collect top-N candidates

**Files:**
- Modify: `unredact/pipeline/font_detect.py:206-245`

**Step 1: Refactor `_full_search()` to return a list of top candidates**

Currently `_full_search()` only tracks the single best match. Change it to collect the top N candidates using a sorted list. The function signature changes to return `list[FontMatch]` instead of `FontMatch | None`.

Modify `_full_search()` in `font_detect.py`:

```python
import heapq

def _full_search(
    line_h: int,
    scorer: Callable[[ImageFont.FreeTypeFont], float],
    top_n: int = 1,
) -> list[FontMatch]:
    """Full search across all candidate fonts and sizes.

    Args:
        line_h: OCR line height in pixels (used to bound the size search).
        scorer: Callable that takes a PIL font and returns a score (0.0-1.0).
        top_n: Number of top candidates to return (default 1 for backwards compat).

    Returns:
        List of up to top_n FontMatch results, sorted by score descending.
    """
    # Min-heap of (-score, FontMatch) — we negate score so heapq gives us lowest
    # (i.e. worst among our top-N) at index 0 for efficient replacement.
    heap: list[tuple[float, int, FontMatch]] = []
    counter = 0  # tiebreaker for heap ordering

    min_size = max(12, int(line_h * 0.6))
    max_size = min(120, int(line_h * 1.4))

    for font_name in CANDIDATE_FONTS:
        font_path = _find_font_path(font_name)
        if font_path is None:
            continue

        for size in range(min_size, max_size + 1):
            try:
                font = ImageFont.truetype(str(font_path), size)
            except Exception:
                continue

            score = scorer(font)
            match = FontMatch(
                font_name=font_name,
                font_path=font_path,
                font_size=size,
                score=score,
            )

            if len(heap) < top_n:
                heapq.heappush(heap, (score, counter, match))
            elif score > heap[0][0]:
                heapq.heapreplace(heap, (score, counter, match))
            counter += 1

    # Return sorted by score descending
    result = [m for (_, _, m) in sorted(heap, reverse=True)]
    return result
```

**Step 2: Update all callers to use `result[0]` instead of the raw return**

There are 3 callers of `_full_search()`:

1. `detect_font_for_line_from_crop()` (line 283):
   ```python
   results = _full_search(line.h, scorer)
   if not results:
       raise RuntimeError("No matching font found. Check system fonts.")
   return _fine_search(results[0], scorer)
   ```

2. `detect_font_masked()` (line 436):
   ```python
   results = _full_search(line.h, scorer)
   if not results:
       raise RuntimeError("No matching font found. Check system fonts.")
   return _fine_search(results[0], scorer)
   ```

3. `detect_font_for_line()` (line 482):
   ```python
   results = _full_search(line.h, scorer)
   if not results:
       if prior is not None:
           return prior
       raise RuntimeError("No matching font found. Check system fonts.")
   best = _fine_search(results[0], scorer)
   ```

**Step 3: Run all existing tests to verify nothing broke**

Run: `pytest tests/test_font_detect.py tests/test_pixel_scoring.py tests/test_pixel_full_search.py tests/test_font_masking.py tests/test_font_detect_real.py -v`
Expected: all PASS

**Step 4: Commit**

```bash
git add unredact/pipeline/font_detect.py
git commit -m "refactor: _full_search returns top-N candidates list"
```

---

### Task 6: Wire debug image saving into font detection

**Files:**
- Modify: `unredact/pipeline/font_debug.py`
- Modify: `unredact/pipeline/font_detect.py`

**Step 1: Add the `save_line_debug()` function to `font_debug.py`**

This is the main entry point called from `font_detect.py`. It re-renders the top candidates and saves everything.

```python
def save_line_debug(
    debug_dir: Path,
    line_idx: int,
    line_crop: np.ndarray,
    top_candidates: list,  # list[FontMatch] — avoid circular import
    scorer_data: dict,
) -> None:
    """Save debug images for one line's font matching.

    Args:
        debug_dir: Timestamped debug output directory.
        line_idx: Line index (for filenames).
        line_crop: Grayscale numpy crop of the line.
        top_candidates: Top-N FontMatch results from _full_search.
        scorer_data: Dict with keys needed to re-render:
            - "line" (OcrLine) and "type" = "line" for line scoring, OR
            - "char_runs", "line_x", "line_y" and "type" = "masked" for masked scoring.
    """
    from PIL import ImageFont as PilFont

    page_bin = line_crop < 128

    # Save raw crop
    crop_img = Image.fromarray(line_crop)
    crop_img.save(debug_dir / f"line-{line_idx:02d}_crop.png")

    composites = []
    for rank, match in enumerate(top_candidates, 1):
        try:
            font = PilFont.truetype(str(match.font_path), match.font_size)
        except Exception:
            continue

        # Re-render to get rendered_bin
        h, w = line_crop.shape
        rendered_img = Image.new("L", (w, h), 255)
        draw = ImageDraw.Draw(rendered_img)

        if scorer_data["type"] == "masked":
            for run in scorer_data["char_runs"]:
                text = "".join(c.text for c in run)
                if not text.strip():
                    continue
                cx = run[0].x - scorer_data["line_x"]
                cy = run[0].y - scorer_data["line_y"]
                bbox = font.getbbox(text)
                draw.text((cx - bbox[0], cy - bbox[1]), text, font=font, fill=0)
        else:
            line = scorer_data["line"]
            bbox = font.getbbox(line.text)
            draw.text((line.x - bbox[0], line.y - bbox[1]), line.text, font=font, fill=0)

        rendered_bin = np.array(rendered_img) < 128

        # Find best shift (same as scoring code)
        best_shifted = rendered_bin
        best_dice = 0.0
        page_ink = int(page_bin.sum())
        rend_ink = int(rendered_bin.sum())
        total = page_ink + rend_ink
        if total > 0:
            for dy in range(-3, 4):
                for dx in range(-3, 4):
                    shifted = _shift_2d(rendered_bin, dx, dy)
                    intersection = int((page_bin & shifted).sum())
                    dice = 2.0 * intersection / total
                    if dice > best_dice:
                        best_dice = dice
                        best_shifted = shifted

        comp = render_candidate_composite(
            page_bin, best_shifted,
            match.font_name, match.font_size, match.score, rank,
        )
        safe_name = match.font_name.replace(" ", "")
        comp.save(debug_dir / f"line-{line_idx:02d}_rank-{rank}_{safe_name}_{match.font_size}px_{match.score:.2f}.png")
        composites.append(comp)

    if composites:
        summary = render_summary_image(composites)
        summary.save(debug_dir / f"line-{line_idx:02d}_summary.png")
```

Note: import `_shift_2d` from `font_detect` at the top of `font_debug.py`:
```python
from unredact.pipeline.font_detect import _shift_2d
```

**Step 2: Wire debug into `detect_font_masked()`**

Modify `detect_font_masked()` in `font_detect.py` (around line 390-439).

After the `_full_search()` call, add:

```python
from unredact.pipeline.font_debug import debug_enabled, save_line_debug

# Inside detect_font_masked, after _full_search:
top_n = 5 if debug_enabled() else 1
results = _full_search(line.h, scorer, top_n=top_n)
if not results:
    raise RuntimeError("No matching font found. Check system fonts.")
best = _fine_search(results[0], scorer)

if debug_enabled() and hasattr(detect_font_masked, '_debug_dir'):
    save_line_debug(
        detect_font_masked._debug_dir,
        detect_font_masked._debug_line_idx,
        line_crop,
        results,
        {"type": "masked", "char_runs": char_runs, "line_x": line.x, "line_y": line.y},
    )
    detect_font_masked._debug_line_idx += 1
```

**Step 3: Wire debug into `detect_font_for_line()`**

Similarly modify `detect_font_for_line()` (around line 442-496).

After the `_full_search()` call:

```python
top_n = 5 if debug_enabled() else 1
results = _full_search(line.h, scorer, top_n=top_n)
# ... existing None check ...
best = _fine_search(results[0], scorer)

if debug_enabled() and hasattr(detect_font_for_line, '_debug_dir'):
    scoring_line_for_debug = scoring_line  # already defined above
    save_line_debug(
        detect_font_for_line._debug_dir,
        detect_font_for_line._debug_line_idx,
        line_crop,
        results,
        {"type": "line", "line": scoring_line_for_debug},
    )
    detect_font_for_line._debug_line_idx += 1
```

**Step 4: Initialize debug dir in `detect_fonts()` and `detect_font_masked()`**

At the top of `detect_fonts()` (line 499):
```python
if debug_enabled():
    from unredact.pipeline.font_debug import init_debug_dir
    detect_font_for_line._debug_dir = init_debug_dir()
    detect_font_for_line._debug_line_idx = 0
```

For `detect_font_masked()`, the debug dir should be initialized once per page analysis. The simplest approach: add the init at the top of `detect_font_masked()` if not already set:
```python
if debug_enabled():
    from unredact.pipeline.font_debug import init_debug_dir
    if not hasattr(detect_font_masked, '_debug_dir'):
        detect_font_masked._debug_dir = init_debug_dir()
        detect_font_masked._debug_line_idx = 0
```

**Step 5: Run all font tests**

Run: `pytest tests/test_font_detect.py tests/test_pixel_scoring.py tests/test_pixel_full_search.py tests/test_font_masking.py tests/test_font_detect_real.py -v`
Expected: all PASS (debug is off by default)

**Step 6: Manual smoke test with debug enabled**

```bash
UNREDACT_DEBUG=1 pytest tests/test_font_detect_real.py -v
ls debug/font-match-*/
```

Verify: debug images are created, summary images show tiled candidates.

**Step 7: Commit**

```bash
git add unredact/pipeline/font_debug.py unredact/pipeline/font_detect.py
git commit -m "feat(debug): save font matching debug images when UNREDACT_DEBUG=1"
```

---

### Task 7: Clean up — use a context manager instead of function attributes

**Files:**
- Modify: `unredact/pipeline/font_debug.py`
- Modify: `unredact/pipeline/font_detect.py`

The function-attribute approach in Task 6 is a quick hack. Replace it with a module-level context variable for cleanliness.

**Step 1: Add a simple debug context to `font_debug.py`**

```python
# Module-level debug state (set during a detection run)
_debug_ctx: dict | None = None


def start_debug_session() -> Path:
    """Start a debug session. Returns the debug directory."""
    global _debug_ctx
    d = init_debug_dir()
    _debug_ctx = {"dir": d, "line_idx": 0}
    return d


def end_debug_session():
    global _debug_ctx
    _debug_ctx = None


def get_debug_ctx() -> dict | None:
    return _debug_ctx


def next_line_idx() -> int:
    if _debug_ctx is None:
        return 0
    idx = _debug_ctx["line_idx"]
    _debug_ctx["line_idx"] = idx + 1
    return idx
```

**Step 2: Replace function attributes in `font_detect.py`**

In `detect_fonts()`:
```python
if debug_enabled():
    from unredact.pipeline.font_debug import start_debug_session
    start_debug_session()
# ... loop ...
if debug_enabled():
    from unredact.pipeline.font_debug import end_debug_session
    end_debug_session()
```

In `detect_font_masked()`:
```python
if debug_enabled():
    from unredact.pipeline.font_debug import get_debug_ctx, start_debug_session, next_line_idx
    if get_debug_ctx() is None:
        start_debug_session()
    # after search:
    save_line_debug(get_debug_ctx()["dir"], next_line_idx(), ...)
```

**Step 3: Remove all `_debug_dir` / `_debug_line_idx` function attributes**

**Step 4: Run tests**

Run: `pytest tests/test_font_detect.py tests/test_font_masking.py tests/test_font_detect_real.py -v`
Expected: all PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/font_debug.py unredact/pipeline/font_detect.py
git commit -m "refactor(debug): replace function attributes with module-level debug context"
```

---

### Task 8: End-to-end verification

**Step 1: Run the full test suite**

```bash
pytest tests/ -v
```

Expected: all existing tests PASS (debug disabled by default).

**Step 2: Run with debug enabled on real test data**

```bash
UNREDACT_DEBUG=1 pytest tests/test_font_detect_real.py -v
```

Verify:
- `debug/font-match-*` directory created
- Contains `line-NN_crop.png` files
- Contains `line-NN_rank-N_*.png` candidate composites
- Contains `line-NN_summary.png` tiled summaries
- Composite images show: header with font info, page ink, rendered ink, color overlap map

**Step 3: Clean up debug output**

```bash
rm -rf debug/
```
