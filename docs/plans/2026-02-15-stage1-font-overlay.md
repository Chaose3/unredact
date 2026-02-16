# Stage 1: Font Matching Overlay — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a web app that can perfectly overlay green text on top of visible text in a scanned PDF, proving the font detection and rendering pipeline works.

**Architecture:** FastAPI serves a single-page web UI. User uploads a PDF, backend rasterizes it, runs OCR with character bounding boxes, detects the font by comparing rendered candidates against OCR measurements, and returns the page image with a green text overlay. The overlay is composited server-side and also rendered client-side for interactive use.

**Tech Stack:** Python 3.14, FastAPI, pdf2image (poppler), pytesseract (Tesseract), Pillow, OpenCV, vanilla HTML/CSS/JS

**System dependencies (Arch/CachyOS):**
```bash
sudo pacman -S tesseract tesseract-data-eng
```

---

### Task 1: Project Scaffolding

**Files:**
- Create: `pyproject.toml`
- Create: `unredact/__init__.py`
- Create: `unredact/pipeline/__init__.py`
- Create: `tests/__init__.py`
- Create: `tests/conftest.py`

**Step 1: Create pyproject.toml**

```toml
[project]
name = "unredact"
version = "0.1.0"
description = "AI-powered redaction analysis for Epstein case files"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn>=0.34",
    "pdf2image>=1.17",
    "pytesseract>=0.3.13",
    "opencv-python>=4.10",
    "Pillow>=11.0",
    "anthropic>=0.52",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "httpx>=0.28",
]

[build-system]
requires = ["setuptools>=75"]
build-backend = "setuptools.backends._legacy:_Backend"
```

**Step 2: Create package init files**

`unredact/__init__.py` — empty file
`unredact/pipeline/__init__.py` — empty file
`tests/__init__.py` — empty file

**Step 3: Create tests/conftest.py**

```python
from pathlib import Path

import pytest

FIXTURES_DIR = Path(__file__).parent / "fixtures"


@pytest.fixture
def sample_pdf() -> Path:
    """Path to the sample Epstein PDF for testing."""
    pdf = Path("/home/alex/Documents/EFTA00554620.pdf")
    if not pdf.exists():
        pytest.skip("Sample PDF not available")
    return pdf
```

**Step 4: Install the project**

```bash
cd /home/alex/dev/unredact
pip install -e ".[dev]"
```

Verify: `python -c "import unredact; print('ok')"` → prints `ok`

**Step 5: Verify system dependencies**

```bash
tesseract --version
pdftoppm -v
```

Both should print version info.

**Step 6: Commit**

```bash
git add pyproject.toml unredact/ tests/
git commit -m "feat: project scaffolding with dependencies"
```

---

### Task 2: PDF Rasterization Module

**Files:**
- Create: `unredact/pipeline/rasterize.py`
- Create: `tests/test_rasterize.py`

**Step 1: Write the failing test**

`tests/test_rasterize.py`:
```python
from pathlib import Path

from PIL import Image

from unredact.pipeline.rasterize import rasterize_pdf


def test_rasterize_returns_list_of_images(sample_pdf: Path):
    pages = rasterize_pdf(sample_pdf)
    assert len(pages) == 2  # EFTA00554620.pdf has 2 pages
    assert all(isinstance(p, Image.Image) for p in pages)


def test_rasterize_high_dpi(sample_pdf: Path):
    pages = rasterize_pdf(sample_pdf, dpi=300)
    # At 300 DPI, a letter-size page is roughly 2550x3300
    w, h = pages[0].size
    assert w > 2000
    assert h > 3000


def test_rasterize_single_page(sample_pdf: Path):
    pages = rasterize_pdf(sample_pdf, first_page=1, last_page=1)
    assert len(pages) == 1
```

**Step 2: Run test to verify it fails**

```bash
pytest tests/test_rasterize.py -v
```

Expected: FAIL — `ModuleNotFoundError: No module named 'unredact.pipeline.rasterize'`

**Step 3: Write minimal implementation**

`unredact/pipeline/rasterize.py`:
```python
from pathlib import Path

from pdf2image import convert_from_path
from PIL import Image


def rasterize_pdf(
    pdf_path: Path,
    dpi: int = 300,
    first_page: int | None = None,
    last_page: int | None = None,
) -> list[Image.Image]:
    """Convert PDF pages to PIL images at the given DPI."""
    kwargs: dict = {"dpi": dpi}
    if first_page is not None:
        kwargs["first_page"] = first_page
    if last_page is not None:
        kwargs["last_page"] = last_page
    return convert_from_path(str(pdf_path), **kwargs)
```

**Step 4: Run test to verify it passes**

```bash
pytest tests/test_rasterize.py -v
```

Expected: 3 passed

**Step 5: Commit**

```bash
git add unredact/pipeline/rasterize.py tests/test_rasterize.py
git commit -m "feat: PDF rasterization module"
```

---

### Task 3: OCR Module — Character-Level Bounding Boxes

**Files:**
- Create: `unredact/pipeline/ocr.py`
- Create: `tests/test_ocr.py`

**Step 1: Write the failing test**

`tests/test_ocr.py`:
```python
from pathlib import Path

from PIL import Image

from unredact.pipeline.ocr import ocr_page, OcrChar, OcrLine
from unredact.pipeline.rasterize import rasterize_pdf


def test_ocr_returns_lines_with_chars(sample_pdf: Path):
    pages = rasterize_pdf(sample_pdf, first_page=1, last_page=1)
    lines = ocr_page(pages[0])
    assert len(lines) > 0
    assert all(isinstance(line, OcrLine) for line in lines)
    # Each line should have characters
    non_empty = [l for l in lines if l.chars]
    assert len(non_empty) > 0


def test_ocr_chars_have_bounding_boxes(sample_pdf: Path):
    pages = rasterize_pdf(sample_pdf, first_page=1, last_page=1)
    lines = ocr_page(pages[0])
    for line in lines:
        for char in line.chars:
            assert isinstance(char, OcrChar)
            assert char.x >= 0
            assert char.y >= 0
            assert char.w > 0
            assert char.h > 0
            assert len(char.text) == 1


def test_ocr_finds_known_text(sample_pdf: Path):
    """The first page contains 'Jeffrey Epstein' — OCR should find it."""
    pages = rasterize_pdf(sample_pdf, first_page=1, last_page=1)
    lines = ocr_page(pages[0])
    full_text = " ".join(
        "".join(c.text for c in line.chars) for line in lines
    )
    # Tesseract may not be perfect, but should get close
    assert "Jeffrey" in full_text or "jeffrey" in full_text.lower()
```

**Step 2: Run test to verify it fails**

```bash
pytest tests/test_ocr.py -v
```

Expected: FAIL — `ModuleNotFoundError`

**Step 3: Write implementation**

`unredact/pipeline/ocr.py`:
```python
from dataclasses import dataclass

import pytesseract
from PIL import Image


@dataclass
class OcrChar:
    """A single OCR'd character with its bounding box."""
    text: str
    x: int
    y: int
    w: int
    h: int
    conf: float  # 0-100 confidence


@dataclass
class OcrLine:
    """A line of OCR'd text."""
    chars: list[OcrChar]
    x: int
    y: int
    w: int
    h: int

    @property
    def text(self) -> str:
        return "".join(c.text for c in self.chars)


def ocr_page(image: Image.Image) -> list[OcrLine]:
    """Run Tesseract OCR on an image and return character-level results.

    Uses tsv output to get bounding boxes at every level:
    level 1=page, 2=block, 3=paragraph, 4=line, 5=word, 6=character
    (Note: character-level requires Tesseract config adjustment.
     We use word-level boxes and estimate character positions from them.)
    """
    # Get word-level data (Tesseract's character-level is unreliable)
    data = pytesseract.image_to_data(
        image, output_type=pytesseract.Output.DICT, config="--psm 6"
    )

    # Group words into lines, then estimate character positions
    lines: dict[tuple[int, int, int, int], list[dict]] = {}
    n = len(data["text"])

    for i in range(n):
        text = data["text"][i].strip()
        if not text:
            continue
        conf = float(data["conf"][i])
        if conf < 0:
            continue

        line_key = (
            data["block_num"][i],
            data["par_num"][i],
            data["line_num"][i],
            0,
        )
        word_info = {
            "text": text,
            "x": data["left"][i],
            "y": data["top"][i],
            "w": data["width"][i],
            "h": data["height"][i],
            "conf": conf,
        }
        lines.setdefault(line_key, []).append(word_info)

    result: list[OcrLine] = []
    for _key, words in sorted(lines.items()):
        chars: list[OcrChar] = []
        for wi, word in enumerate(words):
            # Estimate per-character positions by dividing word box evenly
            word_text = word["text"]
            if not word_text:
                continue
            char_w = word["w"] / len(word_text)
            for ci, ch in enumerate(word_text):
                chars.append(OcrChar(
                    text=ch,
                    x=int(word["x"] + ci * char_w),
                    y=word["y"],
                    w=max(1, int(char_w)),
                    h=word["h"],
                    conf=word["conf"],
                ))
            # Add space character between words (except after last word)
            if wi < len(words) - 1:
                next_word = words[wi + 1]
                space_x = word["x"] + word["w"]
                space_w = next_word["x"] - space_x
                if space_w > 0:
                    chars.append(OcrChar(
                        text=" ",
                        x=space_x,
                        y=word["y"],
                        w=space_w,
                        h=word["h"],
                        conf=word["conf"],
                    ))

        if chars:
            line_x = chars[0].x
            line_y = min(c.y for c in chars)
            line_w = (chars[-1].x + chars[-1].w) - line_x
            line_h = max(c.y + c.h for c in chars) - line_y
            result.append(OcrLine(
                chars=chars, x=line_x, y=line_y, w=line_w, h=line_h
            ))

    return result
```

**Step 4: Run test to verify it passes**

```bash
pytest tests/test_ocr.py -v
```

Expected: 3 passed

**Step 5: Commit**

```bash
git add unredact/pipeline/ocr.py tests/test_ocr.py
git commit -m "feat: OCR module with character-level bounding boxes"
```

---

### Task 4: Font Detection Module

**Files:**
- Create: `unredact/pipeline/font_detect.py`
- Create: `tests/test_font_detect.py`

**Step 1: Write the failing test**

`tests/test_font_detect.py`:
```python
from pathlib import Path

from PIL import Image

from unredact.pipeline.font_detect import detect_font, FontMatch
from unredact.pipeline.ocr import ocr_page
from unredact.pipeline.rasterize import rasterize_pdf


def test_detect_font_returns_match(sample_pdf: Path):
    pages = rasterize_pdf(sample_pdf, first_page=1, last_page=1)
    lines = ocr_page(pages[0])
    match = detect_font(lines, pages[0])
    assert isinstance(match, FontMatch)
    assert match.font_path is not None
    assert match.font_size > 0
    assert match.score > 0


def test_detect_font_reasonable_size(sample_pdf: Path):
    """At 300 DPI, body text in a letter-size doc is typically 30-60px."""
    pages = rasterize_pdf(sample_pdf, first_page=1, last_page=1)
    lines = ocr_page(pages[0])
    match = detect_font(lines, pages[0])
    # At 300 DPI, 11pt text ≈ 46px. Allow wide range.
    assert 20 < match.font_size < 80


def test_font_match_can_render(sample_pdf: Path):
    """The detected font should be usable for rendering."""
    pages = rasterize_pdf(sample_pdf, first_page=1, last_page=1)
    lines = ocr_page(pages[0])
    match = detect_font(lines, pages[0])
    font = match.to_pil_font()
    # Should be able to measure text with it
    bbox = font.getbbox("Hello")
    assert bbox is not None
    assert bbox[2] > 0  # width > 0
```

**Step 2: Run test to verify it fails**

```bash
pytest tests/test_font_detect.py -v
```

Expected: FAIL — `ModuleNotFoundError`

**Step 3: Write implementation**

`unredact/pipeline/font_detect.py`:
```python
from dataclasses import dataclass
from pathlib import Path

from PIL import Image, ImageDraw, ImageFont

from unredact.pipeline.ocr import OcrLine

# Candidate fonts to test (common document fonts available on the system)
CANDIDATE_FONTS: list[str] = [
    "Times New Roman",
    "Arial",
    "Courier New",
    "Georgia",
    "Liberation Serif",
    "Liberation Sans",
    "DejaVu Serif",
    "DejaVu Sans",
]

# Size range to search (in pixels at rendering DPI)
SIZE_RANGE = range(20, 80, 2)


def _find_font_path(font_name: str) -> Path | None:
    """Find the .ttf file for a font name using fc-match."""
    import subprocess

    result = subprocess.run(
        ["fc-match", "--format=%{file}", font_name],
        capture_output=True, text=True,
    )
    if result.returncode == 0 and result.stdout:
        p = Path(result.stdout.strip())
        if p.exists():
            return p
    return None


@dataclass
class FontMatch:
    font_name: str
    font_path: Path
    font_size: int  # in pixels
    score: float  # lower is better (mean absolute error in px)

    def to_pil_font(self) -> ImageFont.FreeTypeFont:
        return ImageFont.truetype(str(self.font_path), self.font_size)


def _score_font(
    font: ImageFont.FreeTypeFont,
    lines: list[OcrLine],
) -> float:
    """Score how well a font matches the OCR'd character widths.

    Returns mean absolute error in pixels between rendered and OCR'd word widths.
    Lower is better.
    """
    errors: list[float] = []

    for line in lines:
        # Reconstruct words from chars
        word = ""
        word_start_x = -1
        word_end_x = -1

        for char in line.chars:
            if char.text == " ":
                if word and word_start_x >= 0:
                    # Measure this word
                    rendered_bbox = font.getbbox(word)
                    rendered_w = rendered_bbox[2] - rendered_bbox[0]
                    ocr_w = word_end_x - word_start_x
                    if ocr_w > 0:
                        errors.append(abs(rendered_w - ocr_w))
                word = ""
                word_start_x = -1
            else:
                if word_start_x < 0:
                    word_start_x = char.x
                word += char.text
                word_end_x = char.x + char.w

        # Last word in line
        if word and word_start_x >= 0:
            rendered_bbox = font.getbbox(word)
            rendered_w = rendered_bbox[2] - rendered_bbox[0]
            ocr_w = word_end_x - word_start_x
            if ocr_w > 0:
                errors.append(abs(rendered_w - ocr_w))

    if not errors:
        return float("inf")
    return sum(errors) / len(errors)


def detect_font(
    lines: list[OcrLine],
    page_image: Image.Image,
) -> FontMatch:
    """Detect the best matching font and size for OCR'd text.

    Tries all candidate fonts at all sizes in SIZE_RANGE and returns
    the combination with the lowest mean word-width error.
    """
    best: FontMatch | None = None

    for font_name in CANDIDATE_FONTS:
        font_path = _find_font_path(font_name)
        if font_path is None:
            continue

        for size in SIZE_RANGE:
            try:
                font = ImageFont.truetype(str(font_path), size)
            except Exception:
                continue

            score = _score_font(font, lines)

            if best is None or score < best.score:
                best = FontMatch(
                    font_name=font_name,
                    font_path=font_path,
                    font_size=size,
                    score=score,
                )

    if best is None:
        raise RuntimeError("No matching font found. Check system fonts.")

    return best
```

**Step 4: Run test to verify it passes**

```bash
pytest tests/test_font_detect.py -v
```

Expected: 3 passed (may take a few seconds — many font/size combos to test)

**Step 5: Commit**

```bash
git add unredact/pipeline/font_detect.py tests/test_font_detect.py
git commit -m "feat: font detection by comparing rendered vs OCR'd word widths"
```

---

### Task 5: Overlay Renderer

**Files:**
- Create: `unredact/pipeline/overlay.py`
- Create: `tests/test_overlay.py`

**Step 1: Write the failing test**

`tests/test_overlay.py`:
```python
from pathlib import Path

import numpy as np
from PIL import Image

from unredact.pipeline.overlay import render_overlay
from unredact.pipeline.ocr import ocr_page
from unredact.pipeline.font_detect import detect_font
from unredact.pipeline.rasterize import rasterize_pdf


def test_overlay_same_size_as_original(sample_pdf: Path):
    pages = rasterize_pdf(sample_pdf, first_page=1, last_page=1)
    lines = ocr_page(pages[0])
    font_match = detect_font(lines, pages[0])
    result = render_overlay(pages[0], lines, font_match)
    assert result.size == pages[0].size


def test_overlay_has_green_pixels(sample_pdf: Path):
    pages = rasterize_pdf(sample_pdf, first_page=1, last_page=1)
    lines = ocr_page(pages[0])
    font_match = detect_font(lines, pages[0])
    result = render_overlay(pages[0], lines, font_match)
    # Convert to numpy and check for green-ish pixels
    arr = np.array(result)
    # Green channel significantly higher than red and blue
    green_mask = (arr[:, :, 1] > 100) & (arr[:, :, 0] < 100) & (arr[:, :, 2] < 100)
    green_pixel_count = green_mask.sum()
    assert green_pixel_count > 100, "Expected green text pixels in the overlay"


def test_overlay_preserves_original(sample_pdf: Path):
    """Overlay should be composited — original should still be visible."""
    pages = rasterize_pdf(sample_pdf, first_page=1, last_page=1)
    original = pages[0].copy()
    lines = ocr_page(pages[0])
    font_match = detect_font(lines, pages[0])
    result = render_overlay(pages[0], lines, font_match)
    # The result shouldn't be identical to the original (overlay adds green)
    assert list(result.getdata()) != list(original.getdata())
```

**Step 2: Run test to verify it fails**

```bash
pytest tests/test_overlay.py -v
```

Expected: FAIL — `ModuleNotFoundError`

**Step 3: Write implementation**

`unredact/pipeline/overlay.py`:
```python
from PIL import Image, ImageDraw, ImageFont

from unredact.pipeline.font_detect import FontMatch
from unredact.pipeline.ocr import OcrLine


def render_overlay(
    page_image: Image.Image,
    lines: list[OcrLine],
    font_match: FontMatch,
    color: tuple[int, int, int, int] = (0, 200, 0, 160),
) -> Image.Image:
    """Render green text overlay on top of the document image.

    Draws each OCR'd line's text at its detected position using
    the matched font. The overlay is semi-transparent so the
    original document is still visible underneath.

    Args:
        page_image: The original rasterized page.
        lines: OCR'd lines with character positions.
        font_match: Detected font to use for rendering.
        color: RGBA color for the overlay text.

    Returns:
        A new RGBA image with the overlay composited on the original.
    """
    base = page_image.convert("RGBA")
    overlay = Image.new("RGBA", base.size, (0, 0, 0, 0))
    draw = ImageDraw.Draw(overlay)
    font = font_match.to_pil_font()

    for line in lines:
        if not line.chars:
            continue

        text = line.text
        # Position: use the first character's x and the line's y
        # Adjust y by subtracting the font ascent offset so baseline aligns
        x = line.x
        y = line.y

        draw.text((x, y), text, font=font, fill=color)

    return Image.alpha_composite(base, overlay)
```

**Step 4: Run test to verify it passes**

```bash
pytest tests/test_overlay.py -v
```

Expected: 3 passed

**Step 5: Commit**

```bash
git add unredact/pipeline/overlay.py tests/test_overlay.py
git commit -m "feat: green text overlay renderer"
```

---

### Task 6: FastAPI Backend

**Files:**
- Create: `unredact/app.py`
- Create: `unredact/static/` (directory)
- Create: `tests/test_app.py`

**Step 1: Write the failing test**

`tests/test_app.py`:
```python
import io
from pathlib import Path

import pytest
from httpx import ASGITransport, AsyncClient

from unredact.app import app


@pytest.fixture
def pdf_bytes(sample_pdf: Path) -> bytes:
    return sample_pdf.read_bytes()


@pytest.mark.anyio
async def test_upload_pdf(pdf_bytes: bytes):
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.post(
            "/api/upload",
            files={"file": ("test.pdf", pdf_bytes, "application/pdf")},
        )
        assert resp.status_code == 200
        data = resp.json()
        assert "doc_id" in data
        assert data["page_count"] > 0


@pytest.mark.anyio
async def test_get_page_overlay(pdf_bytes: bytes):
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        # Upload first
        resp = await client.post(
            "/api/upload",
            files={"file": ("test.pdf", pdf_bytes, "application/pdf")},
        )
        doc_id = resp.json()["doc_id"]

        # Get page 1 overlay
        resp = await client.get(f"/api/doc/{doc_id}/page/1/overlay")
        assert resp.status_code == 200
        assert resp.headers["content-type"] == "image/png"
        assert len(resp.content) > 1000  # Should be a real image


@pytest.mark.anyio
async def test_get_page_original(pdf_bytes: bytes):
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.post(
            "/api/upload",
            files={"file": ("test.pdf", pdf_bytes, "application/pdf")},
        )
        doc_id = resp.json()["doc_id"]

        resp = await client.get(f"/api/doc/{doc_id}/page/1/original")
        assert resp.status_code == 200
        assert resp.headers["content-type"] == "image/png"


@pytest.mark.anyio
async def test_get_page_data(pdf_bytes: bytes):
    """Should return OCR data and font info as JSON."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.post(
            "/api/upload",
            files={"file": ("test.pdf", pdf_bytes, "application/pdf")},
        )
        doc_id = resp.json()["doc_id"]

        resp = await client.get(f"/api/doc/{doc_id}/page/1/data")
        assert resp.status_code == 200
        data = resp.json()
        assert "font" in data
        assert "lines" in data
        assert data["font"]["name"]
        assert data["font"]["size"] > 0
```

**Step 2: Run test to verify it fails**

```bash
pip install anyio pytest-anyio httpx
pytest tests/test_app.py -v
```

Expected: FAIL — `ModuleNotFoundError`

**Step 3: Write implementation**

`unredact/app.py`:
```python
import io
import uuid
from pathlib import Path
from tempfile import TemporaryDirectory

from fastapi import FastAPI, UploadFile
from fastapi.responses import Response, JSONResponse
from fastapi.staticfiles import StaticFiles
from PIL import Image

from unredact.pipeline.rasterize import rasterize_pdf
from unredact.pipeline.ocr import ocr_page
from unredact.pipeline.font_detect import detect_font
from unredact.pipeline.overlay import render_overlay

app = FastAPI(title="Unredact")

# In-memory store for uploaded docs (local-only tool, no persistence needed)
_docs: dict[str, dict] = {}

STATIC_DIR = Path(__file__).parent / "static"
if STATIC_DIR.exists():
    app.mount("/static", StaticFiles(directory=str(STATIC_DIR), html=True), name="static")


def _image_to_png_bytes(img: Image.Image) -> bytes:
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    return buf.getvalue()


@app.post("/api/upload")
async def upload_pdf(file: UploadFile):
    content = await file.read()
    doc_id = uuid.uuid4().hex[:12]

    # Write to temp file for pdf2image
    tmp = TemporaryDirectory()
    pdf_path = Path(tmp.name) / "doc.pdf"
    pdf_path.write_bytes(content)

    pages = rasterize_pdf(pdf_path)

    # Process each page
    page_data = {}
    for i, page_img in enumerate(pages, start=1):
        lines = ocr_page(page_img)
        font_match = detect_font(lines, page_img)
        overlay_img = render_overlay(page_img, lines, font_match)
        page_data[i] = {
            "original": page_img,
            "overlay": overlay_img,
            "lines": lines,
            "font_match": font_match,
        }

    _docs[doc_id] = {
        "page_count": len(pages),
        "pages": page_data,
        "tmp": tmp,  # prevent cleanup
    }

    return {"doc_id": doc_id, "page_count": len(pages)}


@app.get("/api/doc/{doc_id}/page/{page}/original")
async def get_page_original(doc_id: str, page: int):
    doc = _docs.get(doc_id)
    if not doc or page not in doc["pages"]:
        return JSONResponse({"error": "not found"}, status_code=404)
    png = _image_to_png_bytes(doc["pages"][page]["original"])
    return Response(content=png, media_type="image/png")


@app.get("/api/doc/{doc_id}/page/{page}/overlay")
async def get_page_overlay(doc_id: str, page: int):
    doc = _docs.get(doc_id)
    if not doc or page not in doc["pages"]:
        return JSONResponse({"error": "not found"}, status_code=404)
    overlay = doc["pages"][page]["overlay"]
    # Convert RGBA to RGB for PNG output
    rgb = overlay.convert("RGB")
    png = _image_to_png_bytes(rgb)
    return Response(content=png, media_type="image/png")


@app.get("/api/doc/{doc_id}/page/{page}/data")
async def get_page_data(doc_id: str, page: int):
    doc = _docs.get(doc_id)
    if not doc or page not in doc["pages"]:
        return JSONResponse({"error": "not found"}, status_code=404)

    pd = doc["pages"][page]
    fm = pd["font_match"]
    lines_json = []
    for line in pd["lines"]:
        chars_json = [
            {"text": c.text, "x": c.x, "y": c.y, "w": c.w, "h": c.h, "conf": c.conf}
            for c in line.chars
        ]
        lines_json.append({
            "text": line.text,
            "x": line.x, "y": line.y, "w": line.w, "h": line.h,
            "chars": chars_json,
        })

    return {
        "font": {
            "name": fm.font_name,
            "size": fm.font_size,
            "score": fm.score,
            "path": str(fm.font_path),
        },
        "lines": lines_json,
    }
```

**Step 4: Run tests to verify they pass**

```bash
pytest tests/test_app.py -v
```

Expected: 4 passed

**Step 5: Commit**

```bash
git add unredact/app.py tests/test_app.py
git commit -m "feat: FastAPI backend with upload, overlay, and data endpoints"
```

---

### Task 7: Frontend — Document Viewer with Overlay Toggle

**Files:**
- Create: `unredact/static/index.html`
- Create: `unredact/static/style.css`
- Create: `unredact/static/app.js`

**Step 1: Create index.html**

`unredact/static/index.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Unredact</title>
  <link rel="stylesheet" href="/static/style.css">
</head>
<body>
  <header>
    <h1>UNREDACT</h1>
    <p class="subtitle">Epstein Files Redaction Analysis</p>
  </header>

  <main>
    <section id="upload-section">
      <div id="drop-zone">
        <p>Drop a PDF here or click to upload</p>
        <input type="file" id="file-input" accept=".pdf" hidden>
      </div>
    </section>

    <section id="viewer-section" hidden>
      <div id="controls">
        <button id="prev-page" disabled>&lt; Prev</button>
        <span id="page-info">Page 1 / 1</span>
        <button id="next-page" disabled>Next &gt;</button>
        <label id="overlay-toggle">
          <input type="checkbox" id="show-overlay" checked>
          Show overlay
        </label>
        <span id="font-info"></span>
      </div>
      <div id="doc-container">
        <img id="doc-image" alt="Document page">
      </div>
    </section>
  </main>

  <script src="/static/app.js"></script>
</body>
</html>
```

**Step 2: Create style.css**

`unredact/static/style.css`:
```css
* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: system-ui, -apple-system, sans-serif;
  background: #1a1a2e;
  color: #e0e0e0;
  min-height: 100vh;
}

header {
  padding: 1.5rem 2rem;
  background: #16213e;
  border-bottom: 2px solid #0f3460;
}

header h1 {
  font-size: 1.5rem;
  font-weight: 700;
  letter-spacing: 0.2em;
  color: #00d474;
}

.subtitle {
  font-size: 0.85rem;
  color: #888;
  margin-top: 0.25rem;
}

main { padding: 2rem; }

#drop-zone {
  border: 2px dashed #0f3460;
  border-radius: 12px;
  padding: 4rem;
  text-align: center;
  cursor: pointer;
  transition: border-color 0.2s, background 0.2s;
  max-width: 600px;
  margin: 0 auto;
}

#drop-zone:hover, #drop-zone.dragover {
  border-color: #00d474;
  background: rgba(0, 212, 116, 0.05);
}

#drop-zone p { font-size: 1.1rem; color: #aaa; }

#controls {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 1rem;
  padding: 0.75rem 1rem;
  background: #16213e;
  border-radius: 8px;
}

#controls button {
  background: #0f3460;
  color: #e0e0e0;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}

#controls button:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

#overlay-toggle {
  margin-left: auto;
  cursor: pointer;
  user-select: none;
}

#font-info {
  font-size: 0.8rem;
  color: #00d474;
}

#doc-container {
  background: #111;
  border-radius: 8px;
  overflow: auto;
  max-height: 80vh;
  text-align: center;
}

#doc-image {
  max-width: 100%;
  height: auto;
}

.loading {
  color: #00d474;
  text-align: center;
  padding: 2rem;
  font-size: 1.2rem;
}
```

**Step 3: Create app.js**

`unredact/static/app.js`:
```javascript
const dropZone = document.getElementById("drop-zone");
const fileInput = document.getElementById("file-input");
const uploadSection = document.getElementById("upload-section");
const viewerSection = document.getElementById("viewer-section");
const docImage = document.getElementById("doc-image");
const pageInfo = document.getElementById("page-info");
const prevBtn = document.getElementById("prev-page");
const nextBtn = document.getElementById("next-page");
const overlayToggle = document.getElementById("show-overlay");
const fontInfo = document.getElementById("font-info");

let state = {
  docId: null,
  pageCount: 0,
  currentPage: 1,
  pageData: {},
};

// Drag and drop
dropZone.addEventListener("click", () => fileInput.click());
dropZone.addEventListener("dragover", (e) => {
  e.preventDefault();
  dropZone.classList.add("dragover");
});
dropZone.addEventListener("dragleave", () => dropZone.classList.remove("dragover"));
dropZone.addEventListener("drop", (e) => {
  e.preventDefault();
  dropZone.classList.remove("dragover");
  if (e.dataTransfer.files.length) uploadFile(e.dataTransfer.files[0]);
});
fileInput.addEventListener("change", () => {
  if (fileInput.files.length) uploadFile(fileInput.files[0]);
});

async function uploadFile(file) {
  uploadSection.innerHTML = '<p class="loading">Analyzing document...</p>';

  const form = new FormData();
  form.append("file", file);

  const resp = await fetch("/api/upload", { method: "POST", body: form });
  const data = await resp.json();

  state.docId = data.doc_id;
  state.pageCount = data.page_count;
  state.currentPage = 1;

  uploadSection.hidden = true;
  viewerSection.hidden = false;

  await loadPage(1);
}

async function loadPage(page) {
  state.currentPage = page;
  updateControls();

  const showOverlay = overlayToggle.checked;
  const endpoint = showOverlay ? "overlay" : "original";
  docImage.src = `/api/doc/${state.docId}/page/${page}/${endpoint}`;

  // Load page data (font info, lines)
  if (!state.pageData[page]) {
    const resp = await fetch(`/api/doc/${state.docId}/page/${page}/data`);
    state.pageData[page] = await resp.json();
  }

  const pd = state.pageData[page];
  fontInfo.textContent = `${pd.font.name} ~${pd.font.size}px (score: ${pd.font.score.toFixed(1)})`;
}

function updateControls() {
  pageInfo.textContent = `Page ${state.currentPage} / ${state.pageCount}`;
  prevBtn.disabled = state.currentPage <= 1;
  nextBtn.disabled = state.currentPage >= state.pageCount;
}

prevBtn.addEventListener("click", () => {
  if (state.currentPage > 1) loadPage(state.currentPage - 1);
});
nextBtn.addEventListener("click", () => {
  if (state.currentPage < state.pageCount) loadPage(state.currentPage + 1);
});
overlayToggle.addEventListener("change", () => loadPage(state.currentPage));
```

**Step 4: Manual test — run the server**

```bash
cd /home/alex/dev/unredact
uvicorn unredact.app:app --reload --port 8000
```

Open `http://localhost:8000/static/index.html` in a browser. Upload the sample PDF. Verify:
- Page renders
- Green overlay appears over the text
- Toggle switches between original and overlay
- Font info shows detected font name and size
- Page navigation works

**Step 5: Commit**

```bash
git add unredact/static/
git commit -m "feat: web UI with document viewer and overlay toggle"
```

---

### Task 8: Overlay Calibration — Tune Alignment

This is the critical quality step. After the initial pipeline runs, we iterate on the overlay alignment until the green text sits precisely on top of the real text.

**Files:**
- Modify: `unredact/pipeline/overlay.py`
- Modify: `unredact/pipeline/font_detect.py`
- Create: `tests/test_alignment.py`

**Step 1: Write alignment quality test**

`tests/test_alignment.py`:
```python
from pathlib import Path

import numpy as np
from PIL import Image

from unredact.pipeline.rasterize import rasterize_pdf
from unredact.pipeline.ocr import ocr_page
from unredact.pipeline.font_detect import detect_font
from unredact.pipeline.overlay import render_overlay


def test_overlay_alignment_score(sample_pdf: Path):
    """Measure how well the overlay aligns with the original text.

    Strategy: compare the overlay-only image against the original.
    Where the original has dark text pixels and the overlay also has
    green pixels at the same positions, that's a hit. We want a high
    hit rate.
    """
    pages = rasterize_pdf(sample_pdf, first_page=1, last_page=1)
    page = pages[0]
    lines = ocr_page(page)
    font_match = detect_font(lines, page)
    overlay = render_overlay(page, lines, font_match)

    orig_arr = np.array(page.convert("L"))  # grayscale
    overlay_arr = np.array(overlay)

    # Original dark text pixels (< 128 on grayscale)
    text_mask = orig_arr < 128

    # Overlay green pixels (green channel > 100)
    green_mask = overlay_arr[:, :, 1] > 100

    # How many text pixels have green overlay nearby (within 3px tolerance)?
    from scipy.ndimage import binary_dilation
    green_dilated = binary_dilation(green_mask, iterations=3)

    overlap = text_mask & green_dilated
    if text_mask.sum() == 0:
        return

    hit_rate = overlap.sum() / text_mask.sum()
    # We want at least 30% of text pixels to have overlay nearby
    # (not 100% because redaction boxes are dark too, and headers/footers
    # may use different fonts)
    assert hit_rate > 0.3, f"Overlay alignment too low: {hit_rate:.1%}"
```

**Step 2: Run the test, see what score we get**

```bash
pip install scipy
pytest tests/test_alignment.py -v -s
```

Use the hit_rate to guide calibration. If it's low, the font size or y-offset is wrong.

**Step 3: Add y-offset calibration to overlay.py**

The most common misalignment is vertical — Pillow's `draw.text()` positions from the top of the glyph bounding box, but OCR y-coordinates may reference the baseline or top of the line differently.

Update `render_overlay` in `unredact/pipeline/overlay.py` to compute a y-offset correction:

```python
def _compute_y_offset(
    font: ImageFont.FreeTypeFont,
    lines: list[OcrLine],
) -> int:
    """Compute vertical offset correction.

    Compare where Pillow would render text vs where OCR says it is.
    Returns the number of pixels to shift the overlay down (positive)
    or up (negative).
    """
    offsets: list[int] = []
    ascent, descent = font.getmetrics()

    for line in lines:
        if not line.chars:
            continue
        # OCR reports top of the line bounding box
        # Pillow draws from top of ascent
        # The difference is our correction
        ocr_top = line.y
        # Pillow's text y should equal ocr_top
        # But the actual glyph might be offset by ascent
        bbox = font.getbbox(line.text)
        glyph_top_offset = bbox[1]  # usually negative or 0
        offsets.append(-glyph_top_offset)

    if not offsets:
        return 0
    return int(np.median(offsets))
```

Update the render function to use this offset.

**Step 4: Run alignment test again**

```bash
pytest tests/test_alignment.py -v -s
```

Iterate until hit_rate > 0.3. May need to also adjust x positioning or font size fine-tuning.

**Step 5: Commit**

```bash
git add unredact/pipeline/overlay.py unredact/pipeline/font_detect.py tests/test_alignment.py
git commit -m "feat: overlay alignment calibration with y-offset correction"
```

---

### Task 9: End-to-End Smoke Test & Visual Inspection

**Files:**
- Create: `tests/test_e2e.py`

**Step 1: Write end-to-end test**

`tests/test_e2e.py`:
```python
from pathlib import Path

from PIL import Image

from unredact.pipeline.rasterize import rasterize_pdf
from unredact.pipeline.ocr import ocr_page
from unredact.pipeline.font_detect import detect_font
from unredact.pipeline.overlay import render_overlay


def test_full_pipeline_produces_overlay(sample_pdf: Path):
    """Run the full pipeline and save output for visual inspection."""
    pages = rasterize_pdf(sample_pdf)
    output_dir = Path("/tmp/unredact_test_output")
    output_dir.mkdir(exist_ok=True)

    for i, page in enumerate(pages, start=1):
        lines = ocr_page(page)
        font_match = detect_font(lines, page)
        overlay = render_overlay(page, lines, font_match)

        # Save for visual inspection
        overlay.convert("RGB").save(output_dir / f"page_{i}_overlay.png")
        page.save(output_dir / f"page_{i}_original.png")

        print(f"Page {i}: font={font_match.font_name} "
              f"size={font_match.font_size}px "
              f"score={font_match.score:.1f} "
              f"lines={len(lines)}")

    print(f"\nSaved to {output_dir}/ — open the overlay PNGs to visually check alignment.")
```

**Step 2: Run it and inspect the output**

```bash
pytest tests/test_e2e.py -v -s
```

Open `/tmp/unredact_test_output/page_1_overlay.png` and visually check that green text sits on top of the real text.

**Step 3: Commit**

```bash
git add tests/test_e2e.py
git commit -m "test: end-to-end smoke test with visual output"
```

---

## Summary

| Task | What it builds | Key files |
|------|---------------|-----------|
| 1 | Project scaffolding | `pyproject.toml`, package dirs |
| 2 | PDF → images | `pipeline/rasterize.py` |
| 3 | OCR with char boxes | `pipeline/ocr.py` |
| 4 | Font detection | `pipeline/font_detect.py` |
| 5 | Green overlay | `pipeline/overlay.py` |
| 6 | FastAPI backend | `app.py` |
| 7 | Web UI | `static/index.html`, `style.css`, `app.js` |
| 8 | Alignment calibration | Modify overlay + font_detect |
| 9 | E2E smoke test | Visual inspection |

After Task 9, you should have a working web app where you can upload the Epstein PDF and see green text overlaid on the document. The quality of the overlay tells you whether the foundation is solid enough to build Stages 2-5 on top of.
