# Interactive Redaction UI Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace the upfront OCR + line-based UI with a redaction-centric workflow: OpenCV detects redaction boxes on upload, renders them as clickable canvas overlays, and does on-demand OCR/font detection when a user clicks a specific redaction.

**Architecture:** FastAPI backend gains a new `detect_redactions.py` module (OpenCV) and a `/api/redaction/analyze` endpoint. The upload flow drops OCR/font detection. The vanilla JS frontend replaces the line list with a redaction list, renders clickable redaction indicators on canvas, and shows a per-redaction popover with solver controls.

**Tech Stack:** Python 3.12+, FastAPI, OpenCV (cv2), Tesseract (pytesseract), Pillow, vanilla JS (no framework)

**Design doc:** `docs/plans/2026-02-16-interactive-redaction-ui-design.md`

---

### Task 1: Create redaction detection module

**Files:**
- Create: `unredact/pipeline/detect_redactions.py`
- Create: `tests/test_detect_redactions.py`

**Step 1: Write the failing test**

```python
# tests/test_detect_redactions.py
import numpy as np
from PIL import Image

from unredact.pipeline.detect_redactions import detect_redactions, Redaction


def test_detect_single_black_rectangle():
    """A white image with one black rectangle should yield one redaction."""
    img = Image.new("RGB", (800, 600), "white")
    pixels = np.array(img)
    # Draw a black rectangle (simulating a redaction)
    pixels[200:230, 100:300] = [0, 0, 0]
    img = Image.fromarray(pixels)

    redactions = detect_redactions(img)
    assert len(redactions) == 1
    r = redactions[0]
    assert abs(r.x - 100) < 5
    assert abs(r.y - 200) < 5
    assert abs(r.w - 200) < 10
    assert abs(r.h - 30) < 10


def test_detect_no_redactions_on_clean_page():
    """A white page with no black rectangles should return empty list."""
    img = Image.new("RGB", (800, 600), "white")
    redactions = detect_redactions(img)
    assert redactions == []


def test_detect_ignores_small_noise():
    """Small black spots (< min area) should be ignored."""
    img = Image.new("RGB", (800, 600), "white")
    pixels = np.array(img)
    pixels[100:105, 100:105] = [0, 0, 0]  # 5x5 spot — too small
    img = Image.fromarray(pixels)

    redactions = detect_redactions(img)
    assert redactions == []


def test_detect_multiple_redactions():
    """Multiple black rectangles should all be detected."""
    img = Image.new("RGB", (800, 600), "white")
    pixels = np.array(img)
    pixels[100:125, 50:250] = [0, 0, 0]
    pixels[300:325, 100:400] = [0, 0, 0]
    pixels[450:475, 200:500] = [0, 0, 0]
    img = Image.fromarray(pixels)

    redactions = detect_redactions(img)
    assert len(redactions) == 3
```

**Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_detect_redactions.py -v`
Expected: FAIL with "ModuleNotFoundError: No module named 'unredact.pipeline.detect_redactions'"

**Step 3: Write minimal implementation**

```python
# unredact/pipeline/detect_redactions.py
from dataclasses import dataclass
import uuid

import cv2
import numpy as np
from PIL import Image


@dataclass
class Redaction:
    """A detected redaction bounding box."""
    id: str
    x: int
    y: int
    w: int
    h: int


# Minimum area in pixels to consider (filters noise)
MIN_AREA = 500

# Minimum aspect ratio (width/height) — redactions are wider than tall
MIN_ASPECT = 1.5


def detect_redactions(image: Image.Image) -> list[Redaction]:
    """Detect black-filled rectangles in a page image.

    Converts to grayscale, thresholds for near-black pixels, finds contours,
    and filters for rectangular shapes that look like redaction bars.

    Args:
        image: PIL Image of a document page.

    Returns:
        List of Redaction objects sorted top-to-bottom, left-to-right.
    """
    arr = np.array(image.convert("RGB"))
    gray = cv2.cvtColor(arr, cv2.COLOR_RGB2GRAY)

    # Threshold: pixels darker than 40 are "black"
    _, binary = cv2.threshold(gray, 40, 255, cv2.THRESH_BINARY_INV)

    # Morphological close to merge adjacent redaction fragments
    kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (15, 3))
    binary = cv2.morphologyEx(binary, cv2.MORPH_CLOSE, kernel)

    contours, _ = cv2.findContours(binary, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)

    redactions: list[Redaction] = []
    for contour in contours:
        x, y, w, h = cv2.boundingRect(contour)
        area = w * h

        if area < MIN_AREA:
            continue

        # Aspect ratio filter: redactions are wider than tall
        if h > 0 and w / h < MIN_ASPECT:
            continue

        # Fill ratio: the contour should fill most of the bounding rect
        contour_area = cv2.contourArea(contour)
        if contour_area / area < 0.7:
            continue

        redactions.append(Redaction(
            id=uuid.uuid4().hex[:8],
            x=x, y=y, w=w, h=h,
        ))

    # Sort top-to-bottom, then left-to-right
    redactions.sort(key=lambda r: (r.y, r.x))
    return redactions
```

**Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_detect_redactions.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/detect_redactions.py tests/test_detect_redactions.py
git commit -m "feat: add OpenCV redaction detection module"
```

---

### Task 2: Simplify upload endpoint (remove upfront OCR)

**Files:**
- Modify: `unredact/app.py` (upload handler, imports, data endpoint, remove overlay endpoint)
- Modify: `tests/test_app.py` (update tests for new behavior)

**Step 1: Write the failing test**

Update `tests/test_app.py` — replace old tests that assume OCR data with new tests that expect redaction data:

```python
# Replace test_get_page_data with:
@pytest.mark.anyio
async def test_get_page_data_returns_redactions(pdf_bytes: bytes):
    """Page data should return redaction bboxes, not OCR lines."""
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
        assert "redactions" in data
        assert isinstance(data["redactions"], list)
        # Each redaction should have id, x, y, w, h
        if data["redactions"]:
            r = data["redactions"][0]
            assert "id" in r
            assert "x" in r
            assert "y" in r
            assert "w" in r
            assert "h" in r


# Remove test_get_page_overlay entirely (endpoint removed)
# Remove old test_get_page_data
```

**Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_app.py::test_get_page_data_returns_redactions -v`
Expected: FAIL — old endpoint returns `{"lines": [...]}`

**Step 3: Modify app.py**

In `unredact/app.py`:

1. Add import: `from unredact.pipeline.detect_redactions import detect_redactions`
2. Remove imports: `ocr_page`, `detect_fonts`, `render_overlay`
3. Simplify upload handler:

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
        redactions = detect_redactions(page_img)
        page_data[i] = {
            "original": page_img,
            "redactions": redactions,
        }

    _docs[doc_id] = {
        "page_count": len(pages),
        "pages": page_data,
        "tmp": tmp,
    }

    return {"doc_id": doc_id, "page_count": len(pages)}
```

4. Update data endpoint:

```python
@app.get("/api/doc/{doc_id}/page/{page}/data")
async def get_page_data(doc_id: str, page: int):
    doc = _docs.get(doc_id)
    if not doc or page not in doc["pages"]:
        return JSONResponse({"error": "not found"}, status_code=404)

    pd = doc["pages"][page]
    redactions_json = [
        {"id": r.id, "x": r.x, "y": r.y, "w": r.w, "h": r.h}
        for r in pd["redactions"]
    ]
    return {"redactions": redactions_json}
```

5. Remove the overlay endpoint (`get_page_overlay` function, lines 122-131).

**Step 4: Run tests to verify they pass**

Run: `python -m pytest tests/test_app.py -v`
Expected: All tests pass (remove/update tests that reference old endpoints)

**Step 5: Commit**

```bash
git add unredact/app.py tests/test_app.py
git commit -m "refactor: simplify upload to rasterize + redaction detect only"
```

---

### Task 3: Add /api/redaction/analyze endpoint

**Files:**
- Modify: `unredact/app.py` (add new endpoint)
- Modify: `tests/test_app.py` (add test)

**Step 1: Write the failing test**

```python
# Add to tests/test_app.py:
@pytest.mark.anyio
async def test_redaction_analyze(pdf_bytes: bytes):
    """POST /api/redaction/analyze should OCR the line around a redaction."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        # Upload
        resp = await client.post(
            "/api/upload",
            files={"file": ("test.pdf", pdf_bytes, "application/pdf")},
        )
        doc_id = resp.json()["doc_id"]

        # Get redactions
        resp = await client.get(f"/api/doc/{doc_id}/page/1/data")
        redactions = resp.json()["redactions"]
        if not redactions:
            pytest.skip("No redactions detected in test PDF")

        r = redactions[0]

        # Analyze
        resp = await client.post("/api/redaction/analyze", json={
            "doc_id": doc_id,
            "page": 1,
            "redaction": {"x": r["x"], "y": r["y"], "w": r["w"], "h": r["h"]},
        })
        assert resp.status_code == 200
        data = resp.json()
        assert "segments" in data
        assert "gap" in data
        assert "font" in data
        assert data["gap"]["w"] > 0
        assert data["font"]["name"]
        assert data["font"]["size"] > 0
```

**Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_app.py::test_redaction_analyze -v`
Expected: FAIL — 404 or endpoint not found

**Step 3: Write the endpoint**

Add to `unredact/app.py`:

```python
from unredact.pipeline.ocr import ocr_page
from unredact.pipeline.font_detect import detect_font_for_line


class AnalyzeRequest(BaseModel):
    doc_id: str
    page: int
    redaction: dict  # {x, y, w, h}


@app.post("/api/redaction/analyze")
async def analyze_redaction(req: AnalyzeRequest):
    doc = _docs.get(req.doc_id)
    if not doc or req.page not in doc["pages"]:
        return JSONResponse({"error": "not found"}, status_code=404)

    page_img = doc["pages"][req.page]["original"]
    rx, ry, rw, rh = req.redaction["x"], req.redaction["y"], req.redaction["w"], req.redaction["h"]

    # Expand vertically to capture the full line, horizontally to page edges
    # Use redaction's y-band ± some padding for the line crop
    pad_y = rh  # one redaction-height of vertical padding
    crop_y1 = max(0, ry - pad_y)
    crop_y2 = min(page_img.height, ry + rh + pad_y)
    line_crop = page_img.crop((0, crop_y1, page_img.width, crop_y2))

    # OCR just this line crop
    lines = ocr_page(line_crop)
    if not lines:
        return JSONResponse({"error": "no text detected near redaction"}, status_code=422)

    # Find the line closest to the redaction's y-center
    redaction_cy = (ry - crop_y1) + rh / 2
    best_line = min(lines, key=lambda l: abs((l.y + l.h / 2) - redaction_cy))

    # Detect font for this line
    font_match = detect_font_for_line(best_line)

    # Build segments: text before redaction, gap, text after redaction
    # Convert redaction x to crop-relative coordinates (x is already page-relative,
    # crop starts at x=0 so no adjustment needed for x)
    segments = []
    gap = {"x": rx, "w": rw}

    left_chars = [c for c in best_line.chars if c.x + c.w <= rx]
    right_chars = [c for c in best_line.chars if c.x >= rx + rw]

    left_text = "".join(c.text for c in left_chars).rstrip()
    right_text = "".join(c.text for c in right_chars).lstrip()

    if left_text:
        lx = left_chars[0].x
        lw = (left_chars[-1].x + left_chars[-1].w) - lx
        segments.append({"text": left_text, "x": lx, "w": lw})
    if right_text:
        rx2 = right_chars[0].x
        rw2 = (right_chars[-1].x + right_chars[-1].w) - rx2
        segments.append({"text": right_text, "x": rx2, "w": rw2})

    # Adjust segment coordinates back to page-relative y
    chars_json = [
        {"text": c.text, "x": c.x, "y": c.y + crop_y1, "w": c.w, "h": c.h, "conf": c.conf}
        for c in best_line.chars
    ]

    return {
        "segments": segments,
        "gap": gap,
        "font": {
            "name": font_match.font_name,
            "id": _make_font_id(font_match.font_name),
            "size": font_match.font_size,
            "score": font_match.score,
        },
        "line": {
            "x": best_line.x,
            "y": best_line.y + crop_y1,
            "w": best_line.w,
            "h": best_line.h,
            "text": best_line.text,
        },
        "chars": chars_json,
    }
```

Note: We restore the `ocr_page` and `detect_font_for_line` imports that Task 2 removed. They're now used on-demand only, not at upload.

**Step 4: Run tests to verify they pass**

Run: `python -m pytest tests/test_app.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/app.py tests/test_app.py
git commit -m "feat: add /api/redaction/analyze endpoint for on-demand OCR"
```

---

### Task 4: Rewrite frontend — state model and upload flow

**Files:**
- Modify: `unredact/static/app.js` (state, upload, page loading)

This task replaces the state model and upload/page-loading flow. No tests (vanilla JS, tested manually).

**Step 1: Replace state model (app.js lines 50-66)**

Replace the old state block with:

```javascript
const state = {
  docId: null,
  pageCount: 0,
  currentPage: 1,
  redactions: {},      // id -> {id, x, y, w, h, page, status, analysis, solution}
  activeRedaction: null, // id of the currently active redaction
  fonts: [],
  fontsReady: false,
  zoom: 1,
  panX: 0,
  panY: 0,
  associates: null,
};
```

**Step 2: Update loadPage (app.js lines 159-177)**

Replace with:

```javascript
async function loadPage(page) {
  state.currentPage = page;
  state.activeRedaction = null;
  closePopover();
  updatePageControls();

  docImage.src = `/api/doc/${state.docId}/page/${page}/original`;

  // Load redactions for this page if not already loaded
  const pageRedactions = Object.values(state.redactions).filter(r => r.page === page);
  if (pageRedactions.length === 0) {
    const resp = await fetch(`/api/doc/${state.docId}/page/${page}/data`);
    const data = await resp.json();
    for (const r of data.redactions) {
      state.redactions[r.id] = {
        ...r,
        page: page,
        status: "unanalyzed",
        analysis: null,
        solution: null,
      };
    }
  }

  renderRedactionList();
  renderCanvas();
}
```

**Step 3: Update uploadFile to not wait for OCR**

The upload handler no longer does OCR so it should be fast. Update to match new flow.

**Step 4: Commit**

```bash
git add unredact/static/app.js
git commit -m "refactor: replace line-based state with redaction-centric state model"
```

---

### Task 5: Rewrite frontend — left panel redaction list

**Files:**
- Modify: `unredact/static/app.js` (replace `renderLineList` and `selectLine`)
- Modify: `unredact/static/index.html` (rename `line-list` to `redaction-list`)
- Modify: `unredact/static/style.css` (update left panel styles)

**Step 1: Update HTML**

In `index.html`, replace the left panel contents (line 37):

```html
<div id="left-panel">
  <div id="redaction-list"></div>
</div>
```

**Step 2: Write renderRedactionList in app.js**

Replace `renderLineList` and `selectLine` with:

```javascript
const redactionList = document.getElementById("redaction-list");

function renderRedactionList() {
  const pageRedactions = Object.values(state.redactions)
    .filter(r => r.page === state.currentPage)
    .sort((a, b) => a.y - b.y || a.x - b.x);

  redactionList.innerHTML = "";

  pageRedactions.forEach((r, idx) => {
    const div = document.createElement("div");
    div.className = "redaction-item";
    if (state.activeRedaction === r.id) div.classList.add("selected");
    div.dataset.id = r.id;

    const numEl = document.createElement("div");
    numEl.className = "redaction-num";
    numEl.textContent = `#${idx + 1}`;

    const statusEl = document.createElement("div");
    statusEl.className = `redaction-status status-${r.status}`;
    statusEl.textContent = r.status;

    const infoEl = document.createElement("div");
    infoEl.className = "redaction-info";

    if (r.status === "solved" && r.solution) {
      infoEl.textContent = r.solution.text;
      infoEl.classList.add("solved-text");
    } else if (r.status === "analyzed" && r.analysis) {
      const segs = r.analysis.segments.map(s => s.text).join(" ▮ ");
      infoEl.textContent = segs || `${r.w}×${r.h}px`;
    } else {
      infoEl.textContent = `${r.w}×${r.h}px`;
    }

    div.appendChild(numEl);
    div.appendChild(statusEl);
    div.appendChild(infoEl);

    div.addEventListener("click", () => activateRedaction(r.id));
    redactionList.appendChild(div);
  });
}

function activateRedaction(id) {
  state.activeRedaction = id;
  const r = state.redactions[id];
  if (!r) return;

  renderRedactionList();

  // Pan to the redaction
  state.panX = r.x + r.w / 2;
  state.panY = r.y + r.h / 2;
  applyTransform(true);

  // If unanalyzed, trigger analysis
  if (r.status === "unanalyzed") {
    analyzeRedaction(id);
  } else {
    openPopover(id);
  }

  renderCanvas();
}
```

**Step 3: Update CSS**

Add styles for `.redaction-item`, `.redaction-num`, `.redaction-status`, `.redaction-info` in `style.css`. Replace `.line-item` styles with similar redaction-item styles. Add status color classes: `.status-unanalyzed` (gray), `.status-analyzed` (yellow), `.status-solved` (green).

**Step 4: Commit**

```bash
git add unredact/static/app.js unredact/static/index.html unredact/static/style.css
git commit -m "feat: replace line list with redaction list in left panel"
```

---

### Task 6: Rewrite frontend — canvas rendering for redactions

**Files:**
- Modify: `unredact/static/app.js` (replace `renderOverlay` with `renderCanvas`)

**Step 1: Write renderCanvas**

Replace the old `renderOverlay` function with a new `renderCanvas` that:

```javascript
function renderCanvas() {
  if (!docImage.naturalWidth) return;

  canvas.width = docImage.naturalWidth;
  canvas.height = docImage.naturalHeight;
  canvas.style.width = docImage.naturalWidth + "px";
  canvas.style.height = docImage.naturalHeight + "px";

  ctx.clearRect(0, 0, canvas.width, canvas.height);

  const pageRedactions = Object.values(state.redactions)
    .filter(r => r.page === state.currentPage);

  for (const r of pageRedactions) {
    const isActive = state.activeRedaction === r.id;

    if (r.status === "solved" && r.solution) {
      // Draw solved text in green
      const fontStr = `${r.solution.fontSize}px "${r.solution.fontName}"`;
      ctx.font = fontStr;
      ctx.fillStyle = "rgba(0, 220, 0, 0.85)";
      ctx.textBaseline = "top";
      ctx.fillText(r.solution.text, r.x, r.y);
    } else if (r.preview) {
      // Draw preview text in yellow
      const a = r.analysis;
      if (a) {
        const fontStr = `${a.font.size}px "${a.font.name}"`;
        ctx.font = fontStr;
        ctx.fillStyle = "rgba(255, 200, 0, 0.85)";
        ctx.textBaseline = "top";
        ctx.fillText(r.preview, r.x, r.y);
      }
      // Subtle highlight
      ctx.fillStyle = isActive
        ? "rgba(255, 200, 0, 0.2)"
        : "rgba(255, 200, 0, 0.1)";
      ctx.fillRect(r.x, r.y, r.w, r.h);
    } else {
      // Draw redaction indicator
      ctx.fillStyle = isActive
        ? "rgba(80, 120, 255, 0.35)"
        : "rgba(80, 120, 255, 0.2)";
      ctx.fillRect(r.x, r.y, r.w, r.h);

      ctx.strokeStyle = isActive
        ? "rgba(80, 120, 255, 0.9)"
        : "rgba(80, 120, 255, 0.5)";
      ctx.lineWidth = isActive ? 2 : 1;
      ctx.strokeRect(r.x, r.y, r.w, r.h);
    }
  }
}
```

**Step 2: Add canvas click handler for hit-testing**

Enable pointer-events on the canvas and add click detection:

```javascript
// In style.css, change #overlay-canvas pointer-events from "none" to "auto"

canvas.addEventListener("click", (e) => {
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  const cx = (e.clientX - rect.left) * scaleX;
  const cy = (e.clientY - rect.top) * scaleY;

  // Hit-test against redactions on current page
  const hit = Object.values(state.redactions)
    .filter(r => r.page === state.currentPage)
    .find(r => cx >= r.x && cx <= r.x + r.w && cy >= r.y && cy <= r.y + r.h);

  if (hit) {
    e.stopPropagation(); // prevent pan
    activateRedaction(hit.id);
  }
});
```

**Important:** The canvas needs `pointer-events: auto` but we need to be careful not to break pan/zoom. The click handler should only consume the event when hitting a redaction; otherwise let it bubble for panning. This requires adjusting the drag logic to check if the click hit a redaction.

**Step 3: Update hover cursor**

Add mousemove handler on canvas to change cursor when hovering over a redaction:

```javascript
canvas.addEventListener("mousemove", (e) => {
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  const cx = (e.clientX - rect.left) * scaleX;
  const cy = (e.clientY - rect.top) * scaleY;

  const hover = Object.values(state.redactions)
    .filter(r => r.page === state.currentPage)
    .some(r => cx >= r.x && cx <= r.x + r.w && cy >= r.y && cy <= r.y + r.h);

  canvas.style.cursor = hover ? "pointer" : "";
});
```

**Step 4: Commit**

```bash
git add unredact/static/app.js unredact/static/style.css
git commit -m "feat: canvas-native redaction overlays with hit-testing"
```

---

### Task 7: Rewrite frontend — analyze flow and popover

**Files:**
- Modify: `unredact/static/app.js` (analyzeRedaction, popover)
- Modify: `unredact/static/index.html` (add popover container)
- Modify: `unredact/static/style.css` (popover styles)

**Step 1: Add popover HTML to index.html**

Add inside `#right-panel`, after `#doc-container`:

```html
<div id="redaction-popover" hidden>
  <div class="popover-header">
    <span class="popover-title">Analyze Redaction</span>
    <button id="popover-close" class="size-btn">X</button>
  </div>
  <div id="popover-context" class="popover-section"></div>
  <div id="popover-font" class="popover-section"></div>
  <div id="popover-gap" class="popover-section"></div>
  <div class="popover-section">
    <!-- Solver controls — reuse same IDs from existing solve panel -->
    <label>Charset <select id="solve-charset">
      <option value="lowercase">lowercase</option>
      <option value="uppercase">UPPERCASE</option>
      <option value="capitalized">Capitalized</option>
      <option value="full_name_capitalized">Full Name</option>
      <option value="full_name_caps">FULL NAME</option>
      <option value="alpha">Mixed Case</option>
      <option value="alphanumeric">Alphanumeric</option>
    </select></label>
    <label>Tolerance <input type="range" id="solve-tolerance" min="0" max="5" step="0.5" value="0">
    <span id="solve-tol-value">0</span>px</label>
    <label>Mode <select id="solve-mode">
      <option value="enumerate">Enumerate</option>
      <option value="dictionary">Dictionary</option>
      <option value="both">Both</option>
      <option value="emails">Emails</option>
    </select></label>
    <label>Filter <select id="solve-filter">
      <option value="none">None</option>
      <option value="words">English words</option>
      <option value="names">Names</option>
      <option value="both">Words + Names</option>
    </select></label>
    <label>Prefix <input type="text" id="solve-filter-prefix" placeholder="e.g. j" spellcheck="false" autocomplete="off"></label>
    <label>Suffix <input type="text" id="solve-filter-suffix" placeholder="e.g. son" spellcheck="false" autocomplete="off"></label>
  </div>
  <div class="popover-actions">
    <button id="solve-start" class="solve-btn">Solve</button>
    <button id="solve-stop" class="solve-btn" hidden>Stop</button>
    <button id="solve-accept" class="solve-btn accept" hidden>Accept</button>
    <span id="solve-status"></span>
  </div>
  <div id="solve-results"></div>
</div>
```

**Step 2: Write analyzeRedaction in app.js**

```javascript
async function analyzeRedaction(id) {
  const r = state.redactions[id];
  if (!r) return;

  r.status = "analyzing";
  renderRedactionList();

  const resp = await fetch("/api/redaction/analyze", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      doc_id: state.docId,
      page: r.page,
      redaction: { x: r.x, y: r.y, w: r.w, h: r.h },
    }),
  });

  if (!resp.ok) {
    r.status = "error";
    renderRedactionList();
    return;
  }

  const data = await resp.json();
  r.status = "analyzed";
  r.analysis = data;
  renderRedactionList();
  openPopover(id);
}
```

**Step 3: Write openPopover / closePopover**

```javascript
const popover = document.getElementById("redaction-popover");
const popoverClose = document.getElementById("popover-close");
const popoverContext = document.getElementById("popover-context");
const popoverFont = document.getElementById("popover-font");
const popoverGap = document.getElementById("popover-gap");

function openPopover(id) {
  const r = state.redactions[id];
  if (!r || !r.analysis) return;
  const a = r.analysis;

  // Populate context
  const leftText = a.segments.length > 0 ? a.segments[0].text : "";
  const rightText = a.segments.length > 1 ? a.segments[1].text : "";
  popoverContext.textContent = `...${leftText} ▮▮▮ ${rightText}...`;

  // Font info
  popoverFont.textContent = `${a.font.name} ${a.font.size}px (score: ${a.font.score.toFixed(1)})`;

  // Gap info
  popoverGap.textContent = `Gap: ${a.gap.w}px`;

  // Position popover near the redaction
  // Convert redaction center to screen coordinates
  popover.hidden = false;

  // Position: anchor to top-right of right panel (simple, always visible)
  popover.style.top = "4rem";
  popover.style.right = "0.75rem";
}

function closePopover() {
  popover.hidden = true;
  stopSolve();
}

popoverClose.addEventListener("click", closePopover);
```

**Step 4: Wire up solve in the popover**

The existing `startSolve` function needs to be updated to read from the active redaction's analysis instead of from the old line/segment model. Update it to use `state.redactions[state.activeRedaction].analysis` for font, gap width, and context characters.

```javascript
function startSolve() {
  const r = state.redactions[state.activeRedaction];
  if (!r || !r.analysis) return;
  const a = r.analysis;

  const fontId = a.font.id;
  const fontSize = a.font.size;

  const gapWidth = a.gap.w;

  // Context: last char of left segment, first char of right segment
  const leftCtx = a.segments.length > 0
    ? a.segments[0].text.slice(-1)
    : "";
  const rightCtx = a.segments.length > 1
    ? a.segments[1].text.slice(0, 1)
    : "";

  // ... rest of solve logic stays the same (build body, fetch /api/solve, stream SSE)
}
```

**Step 5: Update handleSolveEvent to set preview on redaction**

When a user clicks a solver result, set `r.preview = data.text` and re-render canvas. When they click Accept, set `r.status = "solved"`, `r.solution = {...}`, clear preview.

**Step 6: Commit**

```bash
git add unredact/static/app.js unredact/static/index.html unredact/static/style.css
git commit -m "feat: popover with on-demand analysis and solver integration"
```

---

### Task 8: Clean up — remove old code

**Files:**
- Modify: `unredact/static/app.js` (remove dead code)
- Modify: `unredact/static/index.html` (remove old UI elements)
- Modify: `unredact/static/style.css` (remove old styles)
- Modify: `unredact/app.py` (remove unused overlay import)

**Step 1: Remove from app.js**

Remove these functions and their event listeners (they're dead code now):
- `renderLineList`, `selectLine`, `scrollToLine`
- `splitSegmentAtCursor`, `renderSegmentInputs`, `updateLineListPreview`
- `ensureOverride`, `ensureSegments`, `getSegments`
- `nudge`, `updatePosDisplay`
- `saveOverrideAndRender`
- Old `renderOverlay`
- Old `updateSolveButton`
- Old `acceptSolution`

Remove old DOM references:
- `lineList`, `fontControls`, `fontSelect`, `sizeSlider`, `sizeValue`, `sizeDown`, `sizeUp`
- `posUp`, `posDown`, `posLeft`, `posRight`, `posReset`, `posDisplay`
- `textEditBar`, `segmentInputs`, `textReset`
- `solveBtn`, `solvePanel`, `solveClose` (old ones — now replaced by popover)

**Step 2: Remove from index.html**

Remove these elements:
- `#font-controls` div (lines 41-68)
- `#text-edit-bar` div (lines 69-73)
- Old `#solve-panel` div (lines 74-131) — replaced by popover

**Step 3: Remove from style.css**

Remove these style blocks:
- `#font-controls` and children
- `#text-edit-bar` and children
- `.seg-input`, `.seg-hint`, `.redaction-marker`
- `.pos-control`, `.dpad` and children
- Old `#solve-panel` (replaced by popover styles)

**Step 4: Remove overlay.py references**

In `app.py`, remove the `overlay` import if still present. The `overlay.py` module itself can stay (not hurting anything), but it's no longer imported or called.

**Step 5: Commit**

```bash
git add unredact/static/app.js unredact/static/index.html unredact/static/style.css unredact/app.py
git commit -m "refactor: remove old line-based UI code and overlay endpoint"
```

---

### Task 9: Add manual redaction marking (click-drag)

**Files:**
- Modify: `unredact/static/app.js` (add drag-to-create-redaction)

**Step 1: Add draw mode**

When user holds Shift and drags on the canvas, draw a new redaction box:

```javascript
let drawDrag = null;

canvas.addEventListener("mousedown", (e) => {
  if (!e.shiftKey) return;

  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;

  drawDrag = {
    startX: (e.clientX - rect.left) * scaleX,
    startY: (e.clientY - rect.top) * scaleY,
  };
  e.stopPropagation();
  e.preventDefault();
});

window.addEventListener("mousemove", (e) => {
  if (!drawDrag) return;
  // Draw preview rectangle on canvas (render existing + new rect)
  renderCanvas();
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  const cx = (e.clientX - rect.left) * scaleX;
  const cy = (e.clientY - rect.top) * scaleY;

  const x = Math.min(drawDrag.startX, cx);
  const y = Math.min(drawDrag.startY, cy);
  const w = Math.abs(cx - drawDrag.startX);
  const h = Math.abs(cy - drawDrag.startY);

  ctx.strokeStyle = "rgba(255, 100, 100, 0.8)";
  ctx.lineWidth = 2;
  ctx.setLineDash([6, 3]);
  ctx.strokeRect(x, y, w, h);
  ctx.setLineDash([]);
});

window.addEventListener("mouseup", (e) => {
  if (!drawDrag) return;

  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  const cx = (e.clientX - rect.left) * scaleX;
  const cy = (e.clientY - rect.top) * scaleY;

  const x = Math.round(Math.min(drawDrag.startX, cx));
  const y = Math.round(Math.min(drawDrag.startY, cy));
  const w = Math.round(Math.abs(cx - drawDrag.startX));
  const h = Math.round(Math.abs(cy - drawDrag.startY));

  drawDrag = null;

  // Only create if large enough
  if (w < 20 || h < 5) {
    renderCanvas();
    return;
  }

  // Create manual redaction
  const id = "m" + Date.now().toString(36);
  state.redactions[id] = {
    id, x, y, w, h,
    page: state.currentPage,
    status: "unanalyzed",
    analysis: null,
    solution: null,
  };

  renderRedactionList();
  renderCanvas();
  activateRedaction(id);
});
```

**Step 2: Commit**

```bash
git add unredact/static/app.js
git commit -m "feat: shift+drag to manually mark redaction boxes"
```

---

### Task 10: Update and clean up tests

**Files:**
- Modify: `tests/test_app.py` (remove/update tests for removed endpoints)
- Modify: `tests/test_overlay.py` (mark as skip or remove if only testing removed overlay usage)

**Step 1: Update test_app.py**

- Remove `test_get_page_overlay` (endpoint removed)
- Remove old `test_get_page_data` (replaced in Task 2)
- Ensure all remaining tests pass

**Step 2: Run full test suite**

Run: `python -m pytest tests/ -v --ignore=tests/test_e2e.py`
Expected: All tests pass. Some tests (like test_overlay.py) may need import updates but the module itself still exists.

**Step 3: Commit**

```bash
git add tests/
git commit -m "test: update test suite for redaction-centric architecture"
```

---

### Task 11: Popover CSS and visual polish

**Files:**
- Modify: `unredact/static/style.css`

**Step 1: Add popover styles**

```css
#redaction-popover {
  position: absolute;
  top: 4rem;
  right: 0.75rem;
  width: 320px;
  max-height: calc(100% - 5rem);
  display: flex;
  flex-direction: column;
  background: rgba(20, 20, 40, 0.95);
  border: 1px solid #333;
  border-radius: 8px;
  padding: 12px;
  z-index: 30;
  overflow: hidden;
  backdrop-filter: blur(10px);
}

/* ... plus styles for .popover-header, .popover-section, etc. */
/* ... plus styles for .redaction-item, .redaction-num, .redaction-status */
```

**Step 2: Add redaction list styles**

```css
.redaction-item {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 0.75rem;
  cursor: pointer;
  border-left: 3px solid transparent;
  transition: background 0.15s;
  font-size: 0.8rem;
}

.redaction-item:hover { background: rgba(255, 255, 255, 0.05); }
.redaction-item.selected { border-left-color: #00d474; background: rgba(0, 212, 116, 0.1); }

.redaction-num { color: #00d474; font-weight: 700; min-width: 2rem; }
.redaction-info { flex: 1; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.redaction-info.solved-text { color: #00d474; }

.status-unanalyzed { color: #666; font-size: 0.65rem; }
.status-analyzed { color: #ffc800; font-size: 0.65rem; }
.status-solved { color: #00d474; font-size: 0.65rem; }
```

**Step 3: Commit**

```bash
git add unredact/static/style.css
git commit -m "style: add popover and redaction list styles"
```

---

## Execution Order

Tasks 1-3 are backend (can be done sequentially).
Tasks 4-9 are frontend (must be sequential, each builds on the previous).
Task 10 is cleanup (after both backend and frontend are done).
Task 11 is visual polish (can be done last).

Dependencies:
- Task 2 depends on Task 1 (uses `detect_redactions`)
- Task 3 depends on Task 2 (builds on new upload flow)
- Task 4 depends on Tasks 2-3 (frontend needs new API shape)
- Tasks 5-9 depend on Task 4 (each builds on new state model)
- Task 10 depends on all above
- Task 11 is independent (CSS only)
