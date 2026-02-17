# Pipeline Refactor Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace per-redaction OCR + blind OpenCV with a unified pipeline: full-page OCR → LLM-based redaction detection → guided OpenCV → masked font detection, all pre-computed during upload with SSE progress streaming.

**Architecture:** Upload triggers a per-page pipeline that runs OCR once, sends text to Claude Haiku to identify redactions (broken text patterns), uses the LLM output to guide OpenCV for precise bounding boxes, then masks redactions for font detection. Results are stored and served from a data endpoint. Old click-to-analyze endpoints are removed.

**Tech Stack:** Python 3.14, FastAPI, Anthropic SDK (Haiku), Tesseract OCR, OpenCV, Pillow, NumPy, SSE-Starlette

---

### Task 1: LLM Redaction Detection Module

**Files:**
- Create: `unredact/pipeline/llm_detect.py`
- Create: `tests/test_llm_detect.py`

**Context:** This module receives OCR lines and calls Claude Haiku to identify where redactions are. The LLM spots broken text patterns (artifacts like `[`, `|`, garbled chars) and returns the clean boundary words around each redaction.

**Step 1: Write the test**

```python
# tests/test_llm_detect.py
"""Tests for LLM-based redaction detection."""
import json
from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from unredact.pipeline.llm_detect import (
    LlmRedaction,
    detect_redactions_llm,
    _build_prompt,
    _parse_response,
)
from unredact.pipeline.ocr import OcrChar, OcrLine


def _make_line(text: str, x: int = 0, y: int = 0, char_w: int = 10, h: int = 30) -> OcrLine:
    """Helper to build an OcrLine from plain text."""
    chars = []
    cx = x
    for ch in text:
        chars.append(OcrChar(text=ch, x=cx, y=y, w=char_w, h=h, conf=95.0))
        cx += char_w
    return OcrLine(chars=chars, x=x, y=y, w=len(text) * char_w, h=h)


class TestBuildPrompt:
    def test_includes_line_text(self):
        lines = [_make_line("Hello world", y=100)]
        prompt = _build_prompt(lines)
        assert "Hello world" in prompt
        assert "Line 0" in prompt

    def test_multiple_lines(self):
        lines = [
            _make_line("First line", y=100),
            _make_line("Second line", y=150),
        ]
        prompt = _build_prompt(lines)
        assert "Line 0" in prompt
        assert "Line 1" in prompt


class TestParseResponse:
    def test_parses_single_redaction(self):
        lines = [_make_line("If you can let [|| or ||| know", y=100)]
        tool_input = {
            "redactions": [
                {
                    "line_index": 0,
                    "left_word": "let",
                    "right_word": "or",
                }
            ]
        }
        results = _parse_response(tool_input, lines)
        assert len(results) == 1
        r = results[0]
        assert r.line_index == 0
        assert r.left_word == "let"
        assert r.right_word == "or"
        # left_x should be the right edge of "let" (char 't' at index 14)
        # right_x should be the left edge of "or" (char 'o' at some position)
        assert r.left_x > 0
        assert r.right_x > r.left_x

    def test_parses_multiple_redactions_same_line(self):
        lines = [_make_line("let ||| or ||| know what", y=100)]
        tool_input = {
            "redactions": [
                {"line_index": 0, "left_word": "let", "right_word": "or"},
                {"line_index": 0, "left_word": "or", "right_word": "know"},
            ]
        }
        results = _parse_response(tool_input, lines)
        assert len(results) == 2
        assert results[0].right_x <= results[1].left_x

    def test_no_redactions(self):
        lines = [_make_line("Clean line with no redactions")]
        tool_input = {"redactions": []}
        results = _parse_response(tool_input, lines)
        assert results == []

    def test_line_at_start(self):
        """Redaction at the very start of a line (no left word)."""
        lines = [_make_line("||| wrote: hello", y=100)]
        tool_input = {
            "redactions": [
                {"line_index": 0, "left_word": "", "right_word": "wrote:"},
            ]
        }
        results = _parse_response(tool_input, lines)
        assert len(results) == 1
        assert results[0].left_x == 0

    def test_line_at_end(self):
        """Redaction at the very end of a line (no right word)."""
        lines = [_make_line("hello wrote: |||", y=100)]
        tool_input = {
            "redactions": [
                {"line_index": 0, "left_word": "wrote:", "right_word": ""},
            ]
        }
        results = _parse_response(tool_input, lines)
        assert len(results) == 1


@pytest.mark.asyncio
async def test_detect_redactions_llm_integration():
    """Test the full async function with a mocked Anthropic client."""
    lines = [
        _make_line("Hello world", y=100),
        _make_line("If you can let ||| know", y=150),
    ]
    mock_response = MagicMock()
    mock_response.stop_reason = "tool_use"
    mock_block = MagicMock()
    mock_block.type = "tool_use"
    mock_block.input = {
        "redactions": [
            {"line_index": 1, "left_word": "let", "right_word": "know"},
        ]
    }
    mock_response.content = [mock_block]

    with patch("unredact.pipeline.llm_detect._get_client") as mock_client_fn:
        mock_client = AsyncMock()
        mock_client.messages.create = AsyncMock(return_value=mock_response)
        mock_client_fn.return_value = mock_client

        results = await detect_redactions_llm(lines)
        assert len(results) == 1
        assert results[0].left_word == "let"
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_llm_detect.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'unredact.pipeline.llm_detect'`

**Step 3: Implement the module**

```python
# unredact/pipeline/llm_detect.py
"""LLM-based redaction detection from OCR text.

Sends OCR text to Claude Haiku to identify redactions by spotting
broken text patterns (artifacts like [, |, garbled chars). Returns
the boundary words and approximate x-positions for each redaction.
"""
from __future__ import annotations

import os
from dataclasses import dataclass

import anthropic

from unredact.pipeline.ocr import OcrLine


@dataclass
class LlmRedaction:
    """A redaction identified by the LLM."""
    line_index: int
    left_word: str       # last clean word before the redaction
    right_word: str      # first clean word after the redaction
    left_x: int          # right edge of left_word (pixels)
    right_x: int         # left edge of right_word (pixels)
    line_y: int           # line Y position (pixels)
    line_h: int           # line height (pixels)


_TOOL = {
    "name": "report_redactions",
    "description": "Report redacted sections found in the OCR text. A redaction is a gap in the text where content has been blacked out. Identify each redaction by the last clean word before it and the first clean word after it.",
    "input_schema": {
        "type": "object",
        "properties": {
            "redactions": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "line_index": {
                            "type": "integer",
                            "description": "Zero-based index of the line containing this redaction.",
                        },
                        "left_word": {
                            "type": "string",
                            "description": "The last clean word before the redaction. Empty string if redaction is at line start.",
                        },
                        "right_word": {
                            "type": "string",
                            "description": "The first clean word after the redaction. Empty string if redaction is at line end.",
                        },
                    },
                    "required": ["line_index", "left_word", "right_word"],
                },
            },
        },
        "required": ["redactions"],
    },
}

_MODEL = os.environ.get("UNREDACT_LLM_MODEL", "claude-haiku-4-5-20251001")

# Cache the client so we don't recreate it for every call
_client: anthropic.AsyncAnthropic | None = None


def _get_client() -> anthropic.AsyncAnthropic:
    global _client
    if _client is None:
        _client = anthropic.AsyncAnthropic()
    return _client


def _build_prompt(lines: list[OcrLine]) -> str:
    """Build the prompt for the LLM from OCR lines."""
    text_lines = []
    for i, line in enumerate(lines):
        text_lines.append(f"Line {i} (y={line.y}, h={line.h}): \"{line.text}\"")

    return (
        "Analyze this OCR text from a scanned document page. "
        "Identify redacted sections where text has been blacked out. "
        "Redactions appear in OCR output as broken text: artifacts like [, ], |, "
        "garbled characters, or unnatural gaps between words.\n\n"
        "For each redaction, report the last clean word before it and the first "
        "clean word after it. If a redaction is at the start of a line, left_word "
        "is empty. If at the end, right_word is empty.\n\n"
        "If a line has no redactions (all text is clean and natural), do not "
        "include it.\n\n"
        + "\n".join(text_lines)
    )


def _find_word_in_chars(
    line: OcrLine,
    word: str,
    search_from: int = 0,
    from_right: bool = False,
) -> tuple[int, int] | None:
    """Find a word in the line's chars, return (start_x, end_x).

    Args:
        line: The OCR line to search.
        word: The word to find.
        search_from: Start searching from this char index.
        from_right: If True, find the rightmost occurrence.

    Returns:
        (left_edge_x, right_edge_x) of the word, or None if not found.
    """
    text = line.text
    if from_right:
        idx = text.rfind(word, search_from)
    else:
        idx = text.find(word, search_from)
    if idx < 0:
        return None
    chars = line.chars
    if idx >= len(chars) or idx + len(word) - 1 >= len(chars):
        return None
    start_char = chars[idx]
    end_char = chars[idx + len(word) - 1]
    return (start_char.x, end_char.x + end_char.w)


def _parse_response(
    tool_input: dict,
    lines: list[OcrLine],
) -> list[LlmRedaction]:
    """Parse the LLM tool response into LlmRedaction objects with positions."""
    results: list[LlmRedaction] = []

    for r in tool_input.get("redactions", []):
        line_idx = r["line_index"]
        if line_idx < 0 or line_idx >= len(lines):
            continue
        line = lines[line_idx]
        left_word = r.get("left_word", "")
        right_word = r.get("right_word", "")

        # Find left boundary
        if left_word:
            pos = _find_word_in_chars(line, left_word, from_right=False)
            left_x = pos[1] if pos else line.x  # right edge of left word
        else:
            left_x = line.x

        # Find right boundary (search after left_x)
        if right_word:
            # Search starting from approximately where left_x is in the char list
            search_from = 0
            for ci, c in enumerate(line.chars):
                if c.x >= left_x:
                    search_from = ci
                    break
            pos = _find_word_in_chars(line, right_word, search_from=search_from)
            right_x = pos[0] if pos else line.x + line.w  # left edge of right word
        else:
            right_x = line.x + line.w

        results.append(LlmRedaction(
            line_index=line_idx,
            left_word=left_word,
            right_word=right_word,
            left_x=left_x,
            right_x=right_x,
            line_y=line.y,
            line_h=line.h,
        ))

    return results


async def detect_redactions_llm(lines: list[OcrLine]) -> list[LlmRedaction]:
    """Detect redactions by analyzing OCR text with Claude Haiku.

    Args:
        lines: OCR lines from a full page.

    Returns:
        List of identified redactions with boundary words and positions.
    """
    if not lines:
        return []

    prompt = _build_prompt(lines)
    client = _get_client()

    response = await client.messages.create(
        model=_MODEL,
        max_tokens=1024,
        tools=[_TOOL],
        tool_choice={"type": "tool", "name": "report_redactions"},
        messages=[{"role": "user", "content": prompt}],
    )

    # Extract tool use result
    for block in response.content:
        if block.type == "tool_use":
            return _parse_response(block.input, lines)

    return []
```

**Step 4: Run tests**

Run: `pytest tests/test_llm_detect.py -v`
Expected: All PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/llm_detect.py tests/test_llm_detect.py
git commit -m "feat: add LLM-based redaction detection module"
```

---

### Task 2: Guided OpenCV Search

**Files:**
- Modify: `unredact/pipeline/detect_redactions.py`
- Create: `tests/test_guided_detect.py`

**Context:** Add a function that searches for a black rectangle within a specific region (guided by LLM output), instead of scanning the whole page blindly. The existing `detect_redactions()` and `spot_redaction()` functions stay for now as fallbacks.

**Step 1: Write the test**

```python
# tests/test_guided_detect.py
"""Tests for guided redaction detection (searching within a region)."""
import numpy as np
from PIL import Image, ImageDraw

from unredact.pipeline.detect_redactions import find_redaction_in_region, Redaction


def _make_page_with_redaction(
    page_w: int = 800,
    page_h: int = 400,
    rx: int = 300,
    ry: int = 150,
    rw: int = 120,
    rh: int = 20,
) -> Image.Image:
    """Create a white page with a black rectangle (simulated redaction)."""
    img = Image.new("RGB", (page_w, page_h), (255, 255, 255))
    draw = ImageDraw.Draw(img)
    draw.rectangle([rx, ry, rx + rw, ry + rh], fill=(0, 0, 0))
    return img


def test_finds_redaction_in_region():
    img = _make_page_with_redaction(rx=300, ry=150, rw=120, rh=20)
    result = find_redaction_in_region(
        img, search_x1=280, search_y1=140, search_x2=440, search_y2=180,
    )
    assert result is not None
    assert abs(result.x - 300) <= 3
    assert abs(result.y - 150) <= 3
    assert abs(result.w - 120) <= 3
    assert abs(result.h - 20) <= 3


def test_returns_none_when_no_redaction():
    img = Image.new("RGB", (800, 400), (255, 255, 255))
    result = find_redaction_in_region(
        img, search_x1=100, search_y1=100, search_x2=300, search_y2=200,
    )
    assert result is None


def test_does_not_find_outside_region():
    """Redaction exists but outside the search region."""
    img = _make_page_with_redaction(rx=600, ry=150, rw=120, rh=20)
    result = find_redaction_in_region(
        img, search_x1=100, search_y1=140, search_x2=300, search_y2=180,
    )
    assert result is None


def test_finds_small_redaction():
    """Finds a redaction smaller than the old MIN_AREA=500 threshold."""
    img = _make_page_with_redaction(rx=300, ry=150, rw=40, rh=15)
    result = find_redaction_in_region(
        img, search_x1=280, search_y1=140, search_x2=360, search_y2=180,
    )
    assert result is not None
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_guided_detect.py -v`
Expected: FAIL with `ImportError: cannot import name 'find_redaction_in_region'`

**Step 3: Implement**

Add to `unredact/pipeline/detect_redactions.py`:

```python
def find_redaction_in_region(
    image: Image.Image,
    search_x1: int,
    search_y1: int,
    search_x2: int,
    search_y2: int,
    padding: int = 10,
) -> Redaction | None:
    """Find a single black rectangle within a specific region.

    Searches only within the given bounds (plus padding). Used for
    guided detection where the LLM has identified the approximate
    location of a redaction.

    Args:
        image: Full page image.
        search_x1, search_y1: Top-left of search region.
        search_x2, search_y2: Bottom-right of search region.
        padding: Extra pixels to add around the search region.

    Returns:
        Redaction with page-relative coordinates, or None.
    """
    # Clamp and pad the search region
    x1 = max(0, search_x1 - padding)
    y1 = max(0, search_y1 - padding)
    x2 = min(image.width, search_x2 + padding)
    y2 = min(image.height, search_y2 + padding)

    crop = image.crop((x1, y1, x2, y2))
    gray = np.array(crop.convert("L"))

    # Threshold: dark pixels are potential redaction
    _, binary = cv2.threshold(gray, 40, 255, cv2.THRESH_BINARY_INV)

    # Morphological close to merge fragments
    kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (5, 3))
    closed = cv2.morphologyEx(binary, cv2.MORPH_CLOSE, kernel)

    contours, _ = cv2.findContours(closed, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    if not contours:
        return None

    # Pick the largest contour (most likely the redaction)
    best = max(contours, key=cv2.contourArea)
    area = cv2.contourArea(best)
    if area < 100:
        return None

    bx, by, bw, bh = cv2.boundingRect(best)

    return Redaction(
        id=uuid.uuid4().hex[:8],
        x=x1 + bx,
        y=y1 + by,
        w=bw,
        h=bh,
    )
```

**Step 4: Run tests**

Run: `pytest tests/test_guided_detect.py -v`
Expected: All PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/detect_redactions.py tests/test_guided_detect.py
git commit -m "feat: add guided redaction search within a region"
```

---

### Task 3: Font Detection with Masking

**Files:**
- Modify: `unredact/pipeline/font_detect.py`
- Create: `tests/test_font_masking.py`

**Context:** Add a function that masks redaction boxes to white before running font detection on the full line. This replaces the fragile crop-left/crop-right/neighbor-line approaches.

**Step 1: Write the test**

```python
# tests/test_font_masking.py
"""Tests for font detection with redaction masking."""
import numpy as np
from PIL import Image, ImageDraw, ImageFont

from unredact.pipeline.font_detect import (
    detect_font_masked,
    _find_font_path,
)
from unredact.pipeline.ocr import OcrChar, OcrLine


def _render_line_with_redaction(
    font_name: str,
    font_size: int,
    text: str,
    redact_x: int,
    redact_w: int,
) -> tuple[OcrLine, Image.Image, list[tuple[int, int, int, int]]]:
    """Render text with a black box over part of it.

    Returns (ocr_line, page_image, [(rx, ry, rw, rh)]).
    """
    font_path = _find_font_path(font_name)
    assert font_path is not None
    font = ImageFont.truetype(str(font_path), font_size)

    bbox = font.getbbox(text)
    text_w = bbox[2] - bbox[0]
    text_h = bbox[3] - bbox[1]

    pad = 20
    img_w = text_w + pad * 2
    img_h = text_h + pad * 2
    img = Image.new("L", (img_w, img_h), 255)
    draw = ImageDraw.Draw(img)
    draw.text((pad - bbox[0], pad - bbox[1]), text, font=font, fill=0)

    # Draw a black redaction box
    ry = pad - 2
    rh = text_h + 4
    draw.rectangle([redact_x, ry, redact_x + redact_w, ry + rh], fill=0)

    # Build OcrLine (approximate chars)
    chars = []
    char_w = text_w / max(len(text), 1)
    for i, ch in enumerate(text):
        chars.append(OcrChar(
            text=ch, x=int(pad + i * char_w), y=pad,
            w=max(1, int(char_w)), h=text_h, conf=95.0,
        ))
    line = OcrLine(chars=chars, x=pad, y=pad, w=text_w, h=text_h)

    redactions = [(redact_x, ry, redact_w, rh)]

    return line, img, redactions


def test_masked_detection_finds_correct_font():
    """Font detection with masking should find TNR even with a redaction box."""
    line, img, redactions = _render_line_with_redaction(
        "Times New Roman", 50,
        "On Aug 27 2012 at 12:52 PM wrote ot",
        redact_x=300, redact_w=100,
    )

    match = detect_font_masked(line, img, redactions)
    assert "Times" in match.font_name or "Liberation Serif" in match.font_name
    assert 47 <= match.font_size <= 53


def test_masked_detection_not_confused_by_redaction():
    """Without masking, the black box would add spurious ink. With masking, it shouldn't."""
    line, img, redactions = _render_line_with_redaction(
        "Arial", 40,
        "Hello world this is a test sentence",
        redact_x=200, redact_w=80,
    )

    match = detect_font_masked(line, img, redactions)
    assert "Arial" in match.font_name or "Liberation Sans" in match.font_name
    assert 37 <= match.font_size <= 43
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_font_masking.py -v`
Expected: FAIL with `ImportError: cannot import name 'detect_font_masked'`

**Step 3: Implement**

Add to `unredact/pipeline/font_detect.py`:

```python
def detect_font_masked(
    line: OcrLine,
    page_image: Image.Image,
    redaction_boxes: list[tuple[int, int, int, int]],
) -> FontMatch:
    """Detect font for a line by masking redaction boxes to white.

    Args:
        line: OCR line with char positions (page-relative coordinates).
        page_image: Grayscale page image (or will be converted).
        redaction_boxes: List of (x, y, w, h) redaction boxes to mask.

    Returns:
        Best FontMatch for this line.
    """
    # Crop line region
    line_crop = page_image.convert("L").crop(
        (line.x, line.y, line.x + line.w, line.y + line.h)
    )
    line_arr = np.array(line_crop)

    # Mask each redaction box to white (255) in the crop
    for rx, ry, rw, rh in redaction_boxes:
        # Convert page-relative box to crop-relative
        cx = rx - line.x
        cy = ry - line.y
        x1 = max(0, cx)
        y1 = max(0, cy)
        x2 = min(line_arr.shape[1], cx + rw)
        y2 = min(line_arr.shape[0], cy + rh)
        if x1 < x2 and y1 < y2:
            line_arr[y1:y2, x1:x2] = 255

    scoring_line = OcrLine(
        chars=line.chars,
        x=0, y=0,
        w=line.w, h=line.h,
    )

    best = _full_search(scoring_line, line_arr)
    if best is None:
        raise RuntimeError("No matching font found. Check system fonts.")
    return _fine_search(scoring_line, line_arr, best)
```

**Step 4: Run tests**

Run: `pytest tests/test_font_masking.py -v`
Expected: All PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/font_detect.py tests/test_font_masking.py
git commit -m "feat: add masked font detection for lines with redactions"
```

---

### Task 4: Page Analysis Pipeline

**Files:**
- Create: `unredact/pipeline/analyze_page.py`
- Create: `tests/test_analyze_page.py`

**Context:** Orchestrates the full per-page pipeline: OCR → LLM → guided OpenCV → font detection → analysis objects. This is a new module that replaces the inline `_run_analysis()` in app.py.

**Step 1: Write the test**

```python
# tests/test_analyze_page.py
"""Tests for the page analysis pipeline."""
from unittest.mock import AsyncMock, MagicMock, patch

import pytest
from PIL import Image, ImageDraw, ImageFont

from unredact.pipeline.analyze_page import analyze_page, PageAnalysis, RedactionAnalysis
from unredact.pipeline.font_detect import _find_font_path


def _make_test_page() -> Image.Image:
    """Create a test page with text and a black redaction box."""
    img = Image.new("RGB", (800, 400), (255, 255, 255))
    draw = ImageDraw.Draw(img)

    font_path = _find_font_path("Times New Roman")
    if font_path:
        font = ImageFont.truetype(str(font_path), 30)
    else:
        font = ImageFont.load_default()

    draw.text((50, 100), "Hello world this is clean text", font=font, fill=0)
    draw.text((50, 160), "If you can let", font=font, fill=0)
    # Black redaction box
    draw.rectangle([260, 155, 360, 185], fill=0)
    draw.text((370, 160), "know what to do", font=font, fill=0)

    return img


@pytest.mark.asyncio
async def test_analyze_page_returns_results():
    """Integration test: pipeline produces RedactionAnalysis objects."""
    page = _make_test_page()

    mock_llm_redactions = [
        MagicMock(
            line_index=1,
            left_word="let",
            right_word="know",
            left_x=250,
            right_x=370,
            line_y=160,
            line_h=30,
        )
    ]

    with patch("unredact.pipeline.analyze_page.detect_redactions_llm",
               new_callable=AsyncMock, return_value=mock_llm_redactions):
        with patch("unredact.pipeline.analyze_page.find_redaction_in_region") as mock_cv:
            from unredact.pipeline.detect_redactions import Redaction
            mock_cv.return_value = Redaction(id="abc", x=260, y=155, w=100, h=30)

            result = await analyze_page(page)

    assert isinstance(result, PageAnalysis)
    assert len(result.redactions) == 1
    r = result.redactions[0]
    assert r.box.x == 260
    assert r.font is not None
    assert r.left_text is not None


@pytest.mark.asyncio
async def test_analyze_page_no_redactions():
    """Page with no redactions returns empty list."""
    page = Image.new("RGB", (800, 400), (255, 255, 255))

    with patch("unredact.pipeline.analyze_page.detect_redactions_llm",
               new_callable=AsyncMock, return_value=[]):
        result = await analyze_page(page)

    assert result.redactions == []
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_analyze_page.py -v`
Expected: FAIL with `ModuleNotFoundError`

**Step 3: Implement**

```python
# unredact/pipeline/analyze_page.py
"""Full page analysis pipeline: OCR → LLM → guided OpenCV → font detection."""
from __future__ import annotations

import asyncio
from dataclasses import dataclass, field

from PIL import Image

from unredact.pipeline.ocr import ocr_page, OcrLine
from unredact.pipeline.llm_detect import detect_redactions_llm, LlmRedaction
from unredact.pipeline.detect_redactions import find_redaction_in_region, Redaction
from unredact.pipeline.font_detect import (
    detect_font_masked,
    align_text_to_page,
    FontMatch,
)

import numpy as np


@dataclass
class RedactionAnalysis:
    """Complete analysis for a single redaction."""
    box: Redaction
    line: OcrLine
    font: FontMatch
    left_text: str
    right_text: str
    offset_x: float
    offset_y: float


@dataclass
class PageAnalysis:
    """Analysis results for a full page."""
    lines: list[OcrLine]
    redactions: list[RedactionAnalysis] = field(default_factory=list)


def _find_line_for_redaction(
    lines: list[OcrLine],
    llm_red: LlmRedaction,
) -> OcrLine | None:
    """Find the OCR line that matches this LLM-identified redaction."""
    if 0 <= llm_red.line_index < len(lines):
        return lines[llm_red.line_index]
    return None


def _extract_segments(
    line: OcrLine,
    box: Redaction,
) -> tuple[str, str]:
    """Extract clean left/right text around a redaction using center-point filtering."""
    left_chars = [c for c in line.chars if c.x + c.w / 2 < box.x]
    right_chars = [c for c in line.chars if c.x + c.w / 2 > box.x + box.w]
    left_text = "".join(c.text for c in left_chars).rstrip()
    right_text = "".join(c.text for c in right_chars).lstrip()
    return left_text, right_text


def _compute_alignment(
    left_text: str,
    font_match: FontMatch,
    line: OcrLine,
    page_image: Image.Image,
    box: Redaction,
) -> tuple[float, float]:
    """Compute pixel-aligned offset for the left text segment."""
    if not left_text:
        return 0.0, 0.0

    pil_font = font_match.to_pil_font()
    # Crop region around the left text
    text_region_x1 = max(0, line.x - 20)
    text_region_x2 = min(page_image.width, box.x + 20)
    text_region_y1 = max(0, line.y - 10)
    text_region_y2 = min(page_image.height, line.y + line.h + 10)
    text_crop = np.array(page_image.convert("L").crop(
        (text_region_x1, text_region_y1, text_region_x2, text_region_y2)
    ))
    align_dx, align_dy = align_text_to_page(left_text, pil_font, text_crop)
    offset_x = float(text_region_x1 + align_dx - line.x)
    offset_y = float(text_region_y1 + align_dy - line.y)
    return round(offset_x, 1), round(offset_y, 1)


async def analyze_page(
    page_image: Image.Image,
    on_progress: callable | None = None,
) -> PageAnalysis:
    """Run the full analysis pipeline on a single page.

    Args:
        page_image: Full page image (PIL).
        on_progress: Optional callback for progress events.
            Called with (event_type: str, data: dict).

    Returns:
        PageAnalysis with all detected redactions analyzed.
    """
    # Step 1: OCR
    lines = await asyncio.to_thread(ocr_page, page_image)
    if on_progress:
        on_progress("ocr_done", {"line_count": len(lines)})

    if not lines:
        return PageAnalysis(lines=lines)

    # Step 2: LLM detection
    llm_redactions = await detect_redactions_llm(lines)
    if on_progress:
        on_progress("redactions_found", {"count": len(llm_redactions)})

    if not llm_redactions:
        return PageAnalysis(lines=lines)

    # Step 3: Guided OpenCV + font detection + alignment for each redaction
    results: list[RedactionAnalysis] = []

    # Group redactions by line for shared font detection
    line_fonts: dict[int, FontMatch] = {}

    for llm_red in llm_redactions:
        line = _find_line_for_redaction(lines, llm_red)
        if line is None:
            continue

        # Guided OpenCV: find precise bounding box
        box = await asyncio.to_thread(
            find_redaction_in_region,
            page_image,
            llm_red.left_x,
            llm_red.line_y,
            llm_red.right_x,
            llm_red.line_y + llm_red.line_h,
        )
        if box is None:
            continue

        # Font detection (cached per line)
        if llm_red.line_index not in line_fonts:
            # Collect all redaction boxes on this line for masking
            line_boxes = [
                (b.x, b.y, b.w, b.h) for b in
                [r for r in results if r.line is line]
            ]
            line_boxes.append((box.x, box.y, box.w, box.h))
            font_match = await asyncio.to_thread(
                detect_font_masked, line, page_image, line_boxes,
            )
            line_fonts[llm_red.line_index] = font_match
        font_match = line_fonts[llm_red.line_index]

        # Extract text segments
        left_text, right_text = _extract_segments(line, box)

        # Compute alignment
        offset_x, offset_y = await asyncio.to_thread(
            _compute_alignment, left_text, font_match, line, page_image, box,
        )

        results.append(RedactionAnalysis(
            box=box,
            line=line,
            font=font_match,
            left_text=left_text,
            right_text=right_text,
            offset_x=offset_x,
            offset_y=offset_y,
        ))

    if on_progress:
        on_progress("analysis_complete", {"count": len(results)})

    return PageAnalysis(lines=lines, redactions=results)
```

**Step 4: Run tests**

Run: `pytest tests/test_analyze_page.py -v`
Expected: All PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/analyze_page.py tests/test_analyze_page.py
git commit -m "feat: add page analysis pipeline (OCR → LLM → OpenCV → font)"
```

---

### Task 5: Refactor app.py — Upload Pipeline with SSE

**Files:**
- Modify: `unredact/app.py`

**Context:** Replace the old per-redaction endpoints with a new upload flow that pre-computes all analysis during upload via SSE streaming. Remove `/api/redaction/spot` and `/api/redaction/analyze`. Update `/api/upload` to trigger the pipeline. Update `/api/doc/{id}/page/{p}/data` to return pre-computed results.

**Step 1: Update imports at top of app.py**

Replace the old font_detect imports and add the new pipeline import:

```python
# Remove these imports:
# from unredact.pipeline.font_detect import detect_font_for_line, detect_font_for_line_from_crop, align_text_to_page, CANDIDATE_FONTS, _find_font_path
# from unredact.pipeline.ocr import ocr_page, OcrLine

# Replace with:
from unredact.pipeline.font_detect import CANDIDATE_FONTS, _find_font_path
from unredact.pipeline.analyze_page import analyze_page, RedactionAnalysis
```

**Step 2: Replace upload endpoint**

The new upload returns `doc_id` immediately, then starts SSE streaming for analysis progress:

```python
@app.post("/api/upload")
async def upload_pdf(file: UploadFile):
    content = await file.read()
    doc_id = uuid.uuid4().hex[:12]

    tmp = TemporaryDirectory()
    pdf_path = Path(tmp.name) / "doc.pdf"
    pdf_path.write_bytes(content)

    pages = rasterize_pdf(pdf_path)

    page_data = {}
    for i, page_img in enumerate(pages, start=1):
        page_data[i] = {
            "original": page_img,
            "analysis": None,  # Will be filled by pipeline
        }

    _docs[doc_id] = {
        "page_count": len(pages),
        "pages": page_data,
        "tmp": tmp,
    }

    return {"doc_id": doc_id, "page_count": len(pages)}


@app.get("/api/doc/{doc_id}/analyze")
async def analyze_doc(doc_id: str):
    """SSE endpoint that runs analysis on all pages and streams progress."""
    doc = _docs.get(doc_id)
    if not doc:
        return JSONResponse({"error": "not found"}, status_code=404)

    async def event_generator():
        for page_num, pd in doc["pages"].items():
            page_img = pd["original"]

            def on_progress(event_type, data):
                pass  # Progress tracked via yields below

            analysis = await analyze_page(page_img)
            pd["analysis"] = analysis

            # Build redaction data for this page
            redactions_json = []
            for r in analysis.redactions:
                redactions_json.append({
                    "id": r.box.id,
                    "x": r.box.x, "y": r.box.y,
                    "w": r.box.w, "h": r.box.h,
                })

            yield json.dumps({
                "event": "page_complete",
                "page": page_num,
                "redaction_count": len(analysis.redactions),
                "redactions": redactions_json,
            })

        yield json.dumps({"event": "done"})

    return EventSourceResponse(event_generator())
```

**Step 3: Update page data endpoint to return pre-computed analysis**

```python
@app.get("/api/doc/{doc_id}/page/{page}/data")
async def get_page_data(doc_id: str, page: int):
    doc = _docs.get(doc_id)
    if not doc or page not in doc["pages"]:
        return JSONResponse({"error": "not found"}, status_code=404)

    pd = doc["pages"][page]
    analysis = pd.get("analysis")
    if analysis is None:
        return {"redactions": []}

    redactions_json = []
    for r in analysis.redactions:
        font_id = _make_font_id(r.font.font_name)
        segments = []
        if r.left_text:
            segments.append({"text": r.left_text})
        if r.right_text:
            segments.append({"text": r.right_text})

        redactions_json.append({
            "id": r.box.id,
            "x": r.box.x, "y": r.box.y,
            "w": r.box.w, "h": r.box.h,
            "analysis": {
                "segments": segments,
                "gap": {"x": r.box.x, "w": r.box.w},
                "font": {
                    "name": r.font.font_name,
                    "id": font_id,
                    "size": r.font.font_size,
                    "score": r.font.score,
                },
                "line": {
                    "x": r.line.x,
                    "y": r.line.y,
                    "w": r.line.w,
                    "h": r.line.h,
                    "text": r.line.text,
                },
                "offset_x": r.offset_x,
                "offset_y": r.offset_y,
            },
        })

    return {"redactions": redactions_json}
```

**Step 4: Remove old endpoints**

Delete the following functions from `app.py`:
- `_run_analysis()` (the old inline analysis function)
- `analyze_redaction()` (the `/api/redaction/analyze` endpoint)
- `spot_redaction_endpoint()` (the `/api/redaction/spot` endpoint)
- `AnalyzeRequest` model
- `SpotRequest` model

Also remove unused imports: `ocr_page`, `OcrLine`, `detect_font_for_line`, `detect_font_for_line_from_crop`, `align_text_to_page`, `spot_redaction`.

**Step 5: Run existing tests**

Run: `pytest tests/test_app.py -v`

Some tests will need updating (the spot/analyze endpoint tests). Update them to use the new flow or remove them.

**Step 6: Commit**

```bash
git add unredact/app.py tests/test_app.py
git commit -m "refactor: replace per-redaction analysis with pre-computed pipeline"
```

---

### Task 6: Frontend Updates

**Files:**
- Modify: `unredact/static/main.js`
- Modify: `unredact/static/types.js`

**Context:** Update the frontend to: (1) trigger analysis SSE after upload, (2) consume pre-computed redaction data from the page data endpoint (which now includes analysis), (3) remove the click-to-analyze flow.

**Step 1: Update main.js upload flow**

After upload returns `doc_id`, open an SSE connection to `/api/doc/{doc_id}/analyze`. As pages complete, update state:

```javascript
// In uploadFile(), after receiving doc_id:
async function uploadFile(file) {
    // ... existing upload POST ...

    // Start analysis SSE
    const evtSource = new EventSource(`/api/doc/${state.docId}/analyze`);
    evtSource.onmessage = (event) => {
        const data = JSON.parse(event.data);
        if (data.event === "page_complete") {
            // Load the page data (which now includes analysis)
            loadPageData(data.page);
        } else if (data.event === "done") {
            evtSource.close();
        }
    };

    // Load first page immediately (image only)
    await loadPage(1);
}
```

**Step 2: Update loadPage to consume pre-computed analysis**

```javascript
async function loadPageData(pageNum) {
    const resp = await fetch(`/api/doc/${state.docId}/page/${pageNum}/data`);
    const data = await resp.json();

    // Each redaction now comes with analysis pre-computed
    for (const r of data.redactions) {
        const id = r.id;
        if (state.redactions[id]) continue; // Already loaded

        state.redactions[id] = {
            id, x: r.x, y: r.y, w: r.w, h: r.h,
            page: pageNum,
            status: r.analysis ? "analyzed" : "unanalyzed",
            analysis: r.analysis || null,
            solution: null,
            preview: null,
        };

        if (r.analysis) {
            state.redactions[id].overrides = {
                fontId: r.analysis.font.id,
                fontSize: r.analysis.font.size,
                offsetX: r.analysis.offset_x || 0,
                offsetY: r.analysis.offset_y || 0,
                gapWidth: r.analysis.gap.w,
                leftText: r.analysis.segments[0]?.text || "",
                rightText: r.analysis.segments[1]?.text || "",
            };
        }
    }

    renderCanvas();
    renderSidebar();
}
```

**Step 3: Remove click-to-analyze and spot detection**

Remove the double-click handler that called `/api/redaction/spot` and the `analyzeRedaction()` function that called `/api/redaction/analyze`.

**Step 4: Test manually in browser**

1. Start the server: `uvicorn unredact.app:app --reload`
2. Upload a PDF
3. Verify SSE events stream in the browser console
4. Verify redactions appear pre-analyzed with correct fonts and alignment

**Step 5: Commit**

```bash
git add unredact/static/main.js unredact/static/types.js
git commit -m "feat: update frontend to consume pre-computed analysis via SSE"
```

---

### Task 7: Cleanup and Verification

**Files:**
- Remove: Old test files that test removed functionality
- Modify: Any remaining test files that import removed functions

**Step 1: Run full test suite**

Run: `pytest tests/ -v`

Fix any failures from removed imports or changed APIs.

**Step 2: Manual end-to-end test**

1. Upload a PDF with redactions
2. Verify all redactions are auto-detected and analyzed
3. Verify font detection is correct (no more wrong font from neighbor lines)
4. Verify text alignment looks correct
5. Verify the solver still works with the new data format

**Step 3: Final commit**

```bash
git add -A
git commit -m "chore: cleanup old endpoints and fix remaining tests"
```
