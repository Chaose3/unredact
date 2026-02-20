# Redaction Workflow Redesign Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Decouple auto-detection from upload, make double-click (spot) redactions fully analyzed, and move to inline-on-canvas editing.

**Architecture:** OCR runs at upload time and is cached. Both the "Detect Redactions" button and double-click spot use cached OCR data. A new `analyze_spot_redaction()` function runs font detection + context extraction on a single known bounding box. The frontend creates spot redactions as `"analyzed"` with full data, identical to auto-detected ones.

**Tech Stack:** Python/FastAPI backend, vanilla JS frontend (ES modules), Canvas2D rendering, SSE streaming.

---

### Task 1: Add OCR caching to document state

**Files:**
- Modify: `unredact/app.py:91-104` (upload endpoint, page_data structure)
- Test: `tests/test_app.py`

**Step 1: Modify upload to initialize OCR cache slot**

In `unredact/app.py`, change the page_data structure at lines 91-96 to include an `ocr_lines` field:

```python
    page_data = {}
    for i, page_img in enumerate(pages, start=1):
        page_data[i] = {
            "original": page_img,
            "analysis": None,
            "ocr_lines": None,  # Cached OCR results
        }
```

**Step 2: Run test to verify nothing breaks**

Run: `pytest tests/test_app.py -v`
Expected: All existing tests PASS (adding a field doesn't break anything)

**Step 3: Commit**

```bash
git add unredact/app.py
git commit -m "feat: add ocr_lines cache slot to page data structure"
```

---

### Task 2: Create OCR SSE endpoint

**Files:**
- Modify: `unredact/app.py` (add new endpoint after line 127)
- Modify: `unredact/pipeline/ocr.py` (export `OcrLine` and `OcrChar` — already exported)
- Test: `tests/test_app.py` (add test)

**Step 1: Write the failing test**

Add to `tests/test_app.py`:

```python
@pytest.mark.anyio
async def test_ocr_endpoint_streams_results(pdf_bytes: bytes):
    """GET /api/doc/{id}/ocr should stream OCR progress and cache results."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.post(
            "/api/upload",
            files={"file": ("test.pdf", pdf_bytes, "application/pdf")},
        )
        doc_id = resp.json()["doc_id"]

        resp = await client.get(f"/api/doc/{doc_id}/ocr")
        assert resp.status_code == 200
        assert "text/event-stream" in resp.headers.get("content-type", "")
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_app.py::test_ocr_endpoint_streams_results -v`
Expected: FAIL (404 or route not found)

**Step 3: Implement the OCR SSE endpoint**

Add to `unredact/app.py` after line 127 (after `get_font`):

```python
@app.get("/api/doc/{doc_id}/ocr")
async def ocr_doc(doc_id: str):
    """SSE endpoint that runs OCR on all pages and caches results."""
    doc = _docs.get(doc_id)
    if not doc:
        return JSONResponse({"error": "not found"}, status_code=404)

    async def event_generator():
        for page_num, pd in doc["pages"].items():
            # Skip if already cached
            if pd["ocr_lines"] is not None:
                yield json.dumps({
                    "event": "page_ocr_complete",
                    "page": page_num,
                    "num_lines": len(pd["ocr_lines"]),
                })
                continue

            page_img = pd["original"]
            try:
                lines = await asyncio.to_thread(ocr_page, page_img)
            except Exception as exc:
                yield json.dumps({
                    "event": "error",
                    "page": page_num,
                    "message": str(exc),
                })
                continue

            pd["ocr_lines"] = lines
            yield json.dumps({
                "event": "page_ocr_complete",
                "page": page_num,
                "num_lines": len(lines),
            })

        yield json.dumps({"event": "ocr_complete"})

    return EventSourceResponse(event_generator())
```

Also add the import at the top of `app.py` (around line 16):

```python
from unredact.pipeline.ocr import ocr_page, OcrLine
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_app.py::test_ocr_endpoint_streams_results -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/app.py tests/test_app.py
git commit -m "feat: add /api/doc/{id}/ocr SSE endpoint for cached OCR"
```

---

### Task 3: Refactor analyze_page to accept pre-computed OCR

**Files:**
- Modify: `unredact/pipeline/analyze_page.py:45-69`
- Test: `tests/test_analyze_page.py`

**Step 1: Write the failing test**

Add to `tests/test_analyze_page.py`:

```python
@pytest.mark.anyio
async def test_analyze_page_uses_precomputed_ocr(sample_page_image):
    """analyze_page should use pre-computed OCR lines when provided."""
    from unredact.pipeline.ocr import ocr_page
    from unredact.pipeline.analyze_page import analyze_page

    # Pre-compute OCR
    lines = ocr_page(sample_page_image)
    assert len(lines) > 0

    # Pass pre-computed lines — should not re-run OCR
    result = await analyze_page(sample_page_image, ocr_lines=lines)
    assert result.lines is lines  # Same object, not re-computed
```

Note: You'll need to check if `sample_page_image` fixture exists in conftest. If not, create one that rasterizes the sample PDF's first page. Check `test_analyze_page.py` for existing fixtures.

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_analyze_page.py::test_analyze_page_uses_precomputed_ocr -v`
Expected: FAIL (unexpected keyword argument `ocr_lines`)

**Step 3: Modify analyze_page signature**

In `unredact/pipeline/analyze_page.py`, change the function signature at line 45:

```python
async def analyze_page(
    page_image: Image.Image,
    on_progress: callable | None = None,
    ocr_lines: list[OcrLine] | None = None,
) -> PageAnalysis:
```

And change line 69 from:
```python
    lines: list[OcrLine] = await asyncio.to_thread(ocr_page, page_image)
```
to:
```python
    if ocr_lines is not None:
        lines = ocr_lines
    else:
        lines = await asyncio.to_thread(ocr_page, page_image)
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_analyze_page.py::test_analyze_page_uses_precomputed_ocr -v`
Expected: PASS

**Step 5: Run all tests to verify no regressions**

Run: `pytest tests/ -v --timeout=120`
Expected: All tests PASS

**Step 6: Commit**

```bash
git add unredact/pipeline/analyze_page.py tests/test_analyze_page.py
git commit -m "feat: allow analyze_page to accept pre-computed OCR lines"
```

---

### Task 4: Wire analyze endpoint to use cached OCR

**Files:**
- Modify: `unredact/app.py:129-175` (analyze_doc endpoint)

**Step 1: Modify analyze_doc to use cached OCR**

In the `event_generator()` inside `analyze_doc()` (line 136-173), change lines 139-140 from:

```python
            page_img = pd["original"]
            try:
                analysis = await analyze_page(page_img)
```

to:

```python
            page_img = pd["original"]
            ocr_lines = pd.get("ocr_lines")
            try:
                analysis = await analyze_page(page_img, ocr_lines=ocr_lines)
```

Also, after analysis completes (after line 149 `pd["analysis"] = analysis`), cache the OCR lines if they weren't already:

```python
            pd["analysis"] = analysis
            # Cache OCR lines if not already cached
            if pd.get("ocr_lines") is None:
                pd["ocr_lines"] = analysis.lines
```

**Step 2: Run existing tests**

Run: `pytest tests/test_app.py -v`
Expected: All PASS

**Step 3: Commit**

```bash
git add unredact/app.py
git commit -m "feat: wire analyze endpoint to use cached OCR data"
```

---

### Task 5: Create analyze_spot_redaction function

This is the core new function that runs the analysis mini-pipeline on a single known bounding box using cached OCR data.

**Files:**
- Modify: `unredact/pipeline/analyze_page.py` (add new function)
- Test: `tests/test_analyze_page.py` (add test)

**Step 1: Write the failing test**

Add to `tests/test_analyze_page.py`:

```python
def test_analyze_spot_redaction_returns_analysis(sample_page_image):
    """analyze_spot_redaction should return full analysis for a known bbox."""
    from unredact.pipeline.ocr import ocr_page
    from unredact.pipeline.detect_redactions import Redaction
    from unredact.pipeline.analyze_page import analyze_spot_redaction

    lines = ocr_page(sample_page_image)
    assert len(lines) > 0

    # Use a synthetic redaction box in the middle of the first line
    line = lines[0]
    box = Redaction(
        id="test123",
        x=line.x + line.w // 3,
        y=line.y,
        w=line.w // 3,
        h=line.h,
    )

    result = analyze_spot_redaction(sample_page_image, lines, box)
    assert result is not None
    assert result.box is box
    assert result.line is line
    assert result.font is not None
    assert result.font.font_size > 0
    assert isinstance(result.offset_x, float)
    assert isinstance(result.offset_y, float)
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_analyze_page.py::test_analyze_spot_redaction_returns_analysis -v`
Expected: FAIL (cannot import `analyze_spot_redaction`)

**Step 3: Implement analyze_spot_redaction**

Add to `unredact/pipeline/analyze_page.py` after the `analyze_page` function (after line 196):

```python
def analyze_spot_redaction(
    page_image: Image.Image,
    ocr_lines: list[OcrLine],
    box: Redaction,
) -> RedactionAnalysis | None:
    """Run analysis on a single known redaction bounding box.

    Uses cached OCR data to:
    1. Find the OCR line containing the redaction
    2. Detect the font (with redaction masking)
    3. Extract left/right context text
    4. Compute pixel alignment offsets

    Args:
        page_image: PIL Image of the document page.
        ocr_lines: Pre-computed OCR lines for the page.
        box: Known redaction bounding box.

    Returns:
        RedactionAnalysis or None if no suitable OCR line found.
    """
    # Find the OCR line that best contains this redaction box.
    # Use vertical overlap: the line whose vertical range overlaps most with the box.
    best_line = None
    best_overlap = 0
    box_top = box.y
    box_bottom = box.y + box.h

    for line in ocr_lines:
        line_top = line.y
        line_bottom = line.y + line.h
        overlap = max(0, min(box_bottom, line_bottom) - max(box_top, line_top))
        if overlap > best_overlap:
            best_overlap = overlap
            best_line = line

    if best_line is None:
        log.warning("No OCR line found for spot redaction at (%d,%d)", box.x, box.y)
        return None

    line = best_line
    redaction_boxes = [(box.x, box.y, box.w, box.h)]

    # Font detection with masking
    font = detect_font_masked(line, page_image, redaction_boxes)

    # Extract left/right text using center-point char filtering
    left_chars = [c for c in line.chars if c.x + c.w / 2 < box.x]
    right_chars = [c for c in line.chars if c.x + c.w / 2 > box.x + box.w]
    left_text = "".join(c.text for c in left_chars).strip()
    right_text = "".join(c.text for c in right_chars).strip()

    # Pixel alignment
    offset_x = 0.0
    offset_y = 0.0

    if left_text:
        pil_font = font.to_pil_font()
        text_region_x1 = max(0, line.x - 20)
        text_region_x2 = min(page_image.width, box.x + 20)
        text_region_y1 = max(0, line.y - 10)
        text_region_y2 = min(page_image.height, line.y + line.h + 10)
        text_crop = np.array(page_image.convert("L").crop(
            (text_region_x1, text_region_y1, text_region_x2, text_region_y2)
        ))
        align_dx, align_dy = align_text_to_page(
            left_text, pil_font, text_crop,
        )
        offset_x = float(text_region_x1 + align_dx - line.x)
        offset_y = float(text_region_y1 + align_dy - line.y)

    return RedactionAnalysis(
        box=box,
        line=line,
        font=font,
        left_text=left_text,
        right_text=right_text,
        offset_x=offset_x,
        offset_y=offset_y,
    )
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_analyze_page.py::test_analyze_spot_redaction_returns_analysis -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/pipeline/analyze_page.py tests/test_analyze_page.py
git commit -m "feat: add analyze_spot_redaction for single-box analysis"
```

---

### Task 6: Enrich the spot endpoint to return full analysis

**Files:**
- Modify: `unredact/app.py:459-470` (spot endpoint)
- Test: `tests/test_app.py`

**Step 1: Write the failing test**

Add to `tests/test_app.py`:

```python
@pytest.mark.anyio
async def test_spot_returns_analysis(pdf_bytes: bytes):
    """POST /spot should return full analysis data, not just bbox."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        # Upload
        resp = await client.post(
            "/api/upload",
            files={"file": ("test.pdf", pdf_bytes, "application/pdf")},
        )
        doc_id = resp.json()["doc_id"]

        # Run OCR first (required for spot analysis)
        resp = await client.get(f"/api/doc/{doc_id}/ocr")
        assert resp.status_code == 200

        # Get page dimensions to pick a spot
        resp = await client.get(f"/api/doc/{doc_id}/page/1/original")
        assert resp.status_code == 200

        # Try to spot a redaction (we pick center of page — may or may not find one)
        resp = await client.post(
            f"/api/doc/{doc_id}/page/1/spot",
            json={"x": 300, "y": 300},
        )
        if resp.status_code == 404:
            pytest.skip("No redaction found at test coordinates")

        data = resp.json()
        # Should have bbox fields
        assert "id" in data
        assert "x" in data
        assert "w" in data
        # Should ALSO have analysis data
        assert "analysis" in data
        assert data["analysis"] is not None or data.get("analysis_error") is not None
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_app.py::test_spot_returns_analysis -v`
Expected: FAIL (`"analysis" not in data`)

**Step 3: Modify spot endpoint**

Replace the spot endpoint in `unredact/app.py` (lines 459-470) with:

```python
@app.post("/api/doc/{doc_id}/page/{page}/spot")
async def spot(doc_id: str, page: int, data: dict):
    doc = _docs.get(doc_id)
    if not doc or page not in doc["pages"]:
        return JSONResponse({"error": "not found"}, status_code=404)
    click_x = int(data["x"])
    click_y = int(data["y"])
    pd = doc["pages"][page]
    page_img = pd["original"]
    result = spot_redaction(page_img, click_x, click_y)
    if result is None:
        return JSONResponse({"error": "no redaction found"}, status_code=404)

    # Run analysis if OCR data is available
    ocr_lines = pd.get("ocr_lines")
    analysis_json = None
    if ocr_lines:
        ra = await asyncio.to_thread(
            analyze_spot_redaction, page_img, ocr_lines, result,
        )
        if ra is not None:
            font_id = _make_font_id(ra.font.font_name)
            segments = []
            if ra.left_text:
                segments.append({"text": ra.left_text})
            if ra.right_text:
                segments.append({"text": ra.right_text})

            analysis_json = {
                "segments": segments,
                "gap": {"x": ra.box.x, "w": ra.box.w},
                "font": {
                    "name": ra.font.font_name,
                    "id": font_id,
                    "size": ra.font.font_size,
                    "score": ra.font.score,
                },
                "line": {
                    "x": ra.line.x,
                    "y": ra.line.y,
                    "w": ra.line.w,
                    "h": ra.line.h,
                    "text": ra.line.text,
                },
                "offset_x": ra.offset_x,
                "offset_y": ra.offset_y,
            }

    return {
        "id": result.id,
        "x": result.x, "y": result.y,
        "w": result.w, "h": result.h,
        "analysis": analysis_json,
    }
```

Also add the import at the top of `app.py`:

```python
from unredact.pipeline.analyze_page import analyze_page, analyze_spot_redaction
```

(Change the existing `from unredact.pipeline.analyze_page import analyze_page` to include `analyze_spot_redaction`.)

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_app.py::test_spot_returns_analysis -v`
Expected: PASS

**Step 5: Run all app tests**

Run: `pytest tests/test_app.py -v`
Expected: All PASS

**Step 6: Commit**

```bash
git add unredact/app.py tests/test_app.py
git commit -m "feat: spot endpoint returns full analysis data"
```

---

### Task 7: Frontend — Add OCR SSE and remove auto-analyze

**Files:**
- Modify: `unredact/static/main.js:76-105` (uploadFile function)
- Modify: `unredact/static/main.js:107-130` (remove startAnalysisSSE auto-trigger, keep function for button)

**Step 1: Add startOcrSSE function and modify uploadFile**

In `unredact/static/main.js`, add a new function after `startAnalysisSSE` (around line 131):

```javascript
function startOcrSSE() {
  const es = new EventSource(`/api/doc/${state.docId}/ocr`);
  showToast("Running OCR...", "info");

  es.onmessage = (e) => {
    const data = JSON.parse(e.data);
    if (data.event === "page_ocr_complete") {
      // OCR cached for this page on the backend
    } else if (data.event === "ocr_complete") {
      es.close();
      state.ocrReady = true;
      showToast("OCR complete — ready to detect redactions", "success");
      if (detectBtn) detectBtn.disabled = false;
    } else if (data.event === "error") {
      showToast(`OCR error on page ${data.page}: ${data.message}`, "error");
    }
  };
  es.onerror = () => {
    es.close();
    showToast("OCR connection lost", "error");
  };
}
```

**Step 2: Modify uploadFile to call OCR instead of analysis**

In `uploadFile()` (line 76), change line 100 from:

```javascript
  startAnalysisSSE();
```

to:

```javascript
  startOcrSSE();
```

**Step 3: Add `ocrReady` to state**

In `unredact/static/state.js`, add to the state object:

```javascript
  ocrReady: false,
```

**Step 4: Commit**

```bash
git add unredact/static/main.js unredact/static/state.js
git commit -m "feat: run OCR on upload instead of full analysis"
```

---

### Task 8: Frontend — Add "Detect Redactions" button

**Files:**
- Modify: `unredact/static/index.html:36-38` (left panel)
- Modify: `unredact/static/dom.js` (add detectBtn export)
- Modify: `unredact/static/main.js` (import detectBtn, wire click handler)

**Step 1: Add button to HTML**

In `unredact/static/index.html`, change lines 36-38 from:

```html
      <div id="left-panel">
        <div id="redaction-list"></div>
      </div>
```

to:

```html
      <div id="left-panel">
        <button id="detect-btn" disabled>Detect Redactions</button>
        <div id="redaction-list"></div>
      </div>
```

**Step 2: Add DOM reference**

In `unredact/static/dom.js`, add after the `redactionListEl` line (around line 14):

```javascript
export const detectBtn = document.getElementById("detect-btn");
```

**Step 3: Wire click handler in main.js**

In `unredact/static/main.js`, import `detectBtn`:

```javascript
import { dropZone, fileInput, uploadSection, viewerSection, docImage,
    canvas, pageInfo, prevBtn, nextBtn, redactionListEl, detectBtn,
    rightPanel, fontSelect, solveAccept, gapValue, showToast } from './dom.js';
```

Add a click handler (after the `startOcrSSE` function):

```javascript
if (detectBtn) {
  detectBtn.addEventListener("click", () => {
    detectBtn.disabled = true;
    detectBtn.textContent = "Detecting...";
    startAnalysisSSE();
  });
}
```

Also modify `startAnalysisSSE` to re-enable the button when done. In the `done` event handler (around line 120-122), add:

```javascript
      if (detectBtn) {
        detectBtn.disabled = false;
        detectBtn.textContent = "Detect Redactions";
      }
```

**Step 4: Add basic styling for the detect button**

In `unredact/static/index.html` or `style.css`, add styling. Check where existing styles are defined and add:

```css
#detect-btn {
  width: 100%;
  padding: 8px 12px;
  margin-bottom: 8px;
  cursor: pointer;
}
#detect-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
```

**Step 5: Commit**

```bash
git add unredact/static/index.html unredact/static/dom.js unredact/static/main.js
git commit -m "feat: add Detect Redactions button in left panel"
```

---

### Task 9: Frontend — Make spot redactions first-class

**Files:**
- Modify: `unredact/static/main.js:309-334` (dblclick handler)

**Step 1: Update the dblclick handler**

Replace lines 309-334 in `unredact/static/main.js`:

```javascript
canvas.addEventListener("dblclick", async (e) => {
  const rect = rightPanel.getBoundingClientRect();
  const sx = e.clientX - rect.left;
  const sy = e.clientY - rect.top;
  const doc = screenToDoc(sx, sy);

  const resp = await fetch(`/api/doc/${state.docId}/page/${state.currentPage}/spot`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ x: Math.round(doc.x), y: Math.round(doc.y) }),
  });
  if (!resp.ok) {
    showToast("No redaction found at that spot", "error");
    return;
  }
  const r = await resp.json();

  const redaction = {
    id: r.id, x: r.x, y: r.y, w: r.w, h: r.h,
    page: state.currentPage,
    status: r.analysis ? "analyzed" : "unanalyzed",
    analysis: r.analysis || null,
    solution: null,
    preview: null,
  };

  if (r.analysis) {
    redaction.overrides = {
      fontId: r.analysis.font.id,
      fontSize: r.analysis.font.size,
      offsetX: r.analysis.offset_x || 0,
      offsetY: r.analysis.offset_y || 0,
      gapWidth: r.analysis.gap.w,
      leftText: r.analysis.segments[0]?.text || "",
      rightText: r.analysis.segments[1]?.text || "",
    };
  }

  state.redactions[r.id] = redaction;
  renderRedactionList();
  renderCanvas();
  activateRedaction(r.id);
});
```

This is the key change: the frontend now processes the full analysis data from the spot response, creating the redaction with `status: "analyzed"` and a populated `overrides` object — identical to how `loadPageData` creates auto-detected redactions.

**Step 2: Manual verification**

Run the app (`make run` or equivalent), upload a PDF, wait for OCR to complete, then double-click a black bar. Verify:
- The redaction appears as green/analyzed (not blue/unanalyzed)
- The popover opens with font and text data
- The text edit bar populates with left/right context
- Ctrl+drag adjusts offset
- Shift+drag adjusts gap width
- The solver works

**Step 3: Commit**

```bash
git add unredact/static/main.js
git commit -m "feat: spot redactions now return fully analyzed with overrides"
```

---

### Task 10: Frontend — Inline text editing on canvas

This adds HTML `<input>` overlays positioned on the canvas at the left/right text locations, so the user feels like they're typing on the document.

**Files:**
- Modify: `unredact/static/index.html:130-133` (add overlay container inside doc-container)
- Modify: `unredact/static/dom.js` (add new element refs)
- Create: `unredact/static/inline-edit.js` (new module for inline editing logic)
- Modify: `unredact/static/main.js` (integrate inline edit)
- Modify: `unredact/static/popover.js` (sync with inline edits)

**Step 1: Add overlay container in HTML**

In `unredact/static/index.html`, after the canvas element (line 132), add:

```html
        <div id="inline-edit-layer" style="position:absolute;top:0;left:0;pointer-events:none;transform-origin:0 0;"></div>
```

The layer sits on top of the canvas inside `doc-container`, transforms with the same zoom/pan.

**Step 2: Create inline-edit.js module**

Create `unredact/static/inline-edit.js`:

```javascript
/**
 * Inline text editing — HTML inputs positioned on the canvas at
 * the left/right text locations so users edit directly on the document.
 */
import { state } from './state.js';
import { renderCanvas } from './canvas.js';

const layer = document.getElementById("inline-edit-layer");

let leftInput = null;
let rightInput = null;

/**
 * Show inline text inputs for the active redaction.
 * @param {string} id - Redaction ID
 */
export function showInlineEdit(id) {
  hideInlineEdit();

  const r = state.redactions[id];
  if (!r?.analysis || !r?.overrides) return;

  const a = r.analysis;
  const o = r.overrides;
  const fontSize = o.fontSize || a.font.size;
  const offsetX = o.offsetX || 0;
  const offsetY = o.offsetY || 0;
  const lineX = a.line.x + offsetX;
  const lineY = a.line.y + offsetY;
  const gapWidth = o.gapWidth || a.gap.w;

  // Measure approximate left text width (rough: fontSize * 0.6 * charCount)
  const leftText = o.leftText || "";
  const rightText = o.rightText || "";

  // Left input: positioned at the start of the line, width up to the gap
  leftInput = _createInput(leftText, lineX, lineY, r.x - lineX, a.line.h, fontSize);
  leftInput.style.textAlign = "right";
  leftInput.addEventListener("input", () => {
    r.overrides.leftText = leftInput.value;
    renderCanvas();
  });

  // Right input: positioned after the gap
  const rightX = r.x + gapWidth;
  const rightW = (a.line.x + a.line.w) - rightX + 20;
  rightInput = _createInput(rightText, rightX, lineY, Math.max(50, rightW), a.line.h, fontSize);
  rightInput.addEventListener("input", () => {
    r.overrides.rightText = rightInput.value;
    renderCanvas();
  });

  layer.appendChild(leftInput);
  layer.appendChild(rightInput);
}

/** Remove all inline edit inputs. */
export function hideInlineEdit() {
  if (leftInput) { leftInput.remove(); leftInput = null; }
  if (rightInput) { rightInput.remove(); rightInput = null; }
}

/** Update input values from state (e.g., after text reset). */
export function syncInlineEdit(id) {
  const r = state.redactions[id];
  if (!r?.overrides) return;
  if (leftInput) leftInput.value = r.overrides.leftText;
  if (rightInput) rightInput.value = r.overrides.rightText;
}

function _createInput(value, x, y, w, h, fontSize) {
  const input = document.createElement("input");
  input.type = "text";
  input.value = value;
  input.style.cssText = `
    position: absolute;
    left: ${x}px;
    top: ${y}px;
    width: ${Math.max(30, w)}px;
    height: ${h}px;
    font-size: ${fontSize}px;
    background: rgba(255,255,255,0.7);
    border: 1px solid rgba(0,180,0,0.5);
    padding: 0 2px;
    pointer-events: auto;
    box-sizing: border-box;
    outline: none;
    font-family: sans-serif;
  `;
  return input;
}
```

**Step 3: Integrate into main.js**

In `main.js`, import the inline edit functions:

```javascript
import { showInlineEdit, hideInlineEdit, syncInlineEdit } from './inline-edit.js';
```

In `activateRedaction()` (around line 287-289), after `openPopover(id)`, add:

```javascript
    showInlineEdit(id);
```

In `closePopover` calls throughout main.js, also call `hideInlineEdit()`. Or better: in the `setOnPopoverClose` callback setup (line 462), add `hideInlineEdit`:

```javascript
setOnPopoverClose(() => { stopSolve(); hideInlineEdit(); });
```

**Step 4: Sync inline edits with text edit bar**

In `popover.js`, the `leftTextInput` and `rightTextInput` `input` event handlers (lines 138-150) should also update inline inputs. Import `syncInlineEdit` and call it:

```javascript
import { syncInlineEdit } from './inline-edit.js';
```

In the leftTextInput handler (line 138-143), after setting `r.overrides.leftText`, add:
```javascript
    syncInlineEdit(state.activeRedaction);
```

Same for rightTextInput handler (line 145-150).

Same for textReset handler (line 152-161).

**Step 5: Commit**

```bash
git add unredact/static/inline-edit.js unredact/static/index.html unredact/static/dom.js unredact/static/main.js unredact/static/popover.js
git commit -m "feat: inline text editing on canvas for redaction context"
```

---

### Task 11: Frontend — Drag handles for bbox resize

**Files:**
- Modify: `unredact/static/main.js` (add drag handle logic near existing modDrag)
- Modify: `unredact/static/canvas.js` (draw resize handles when active)

**Step 1: Draw resize handles in canvas.js**

Add a new function to `canvas.js`:

```javascript
function drawResizeHandles(r) {
  const sz = 6;
  ctx.fillStyle = "rgba(0, 120, 255, 0.8)";
  // Left edge
  ctx.fillRect(r.x - sz/2, r.y + r.h/2 - sz/2, sz, sz);
  // Right edge
  ctx.fillRect(r.x + r.w - sz/2, r.y + r.h/2 - sz/2, sz, sz);
  // Top edge
  ctx.fillRect(r.x + r.w/2 - sz/2, r.y - sz/2, sz, sz);
  // Bottom edge
  ctx.fillRect(r.x + r.w/2 - sz/2, r.y + r.h - sz/2, sz, sz);
}
```

Call `drawResizeHandles(r)` in `renderCanvas()` after drawing the active redaction (inside the loop, when `r.id === state.activeRedaction`).

**Step 2: Add resize drag handling in main.js**

Add a new drag handler (alongside existing `modDrag`) for bbox resize. When the user clicks near a handle (within ~8px in doc coords), start a resize drag:

```javascript
let resizeDrag = null;

canvas.addEventListener("mousedown", (e) => {
  if (e.button !== 0 || e.ctrlKey || e.shiftKey) return;
  const r = state.redactions[state.activeRedaction];
  if (!r) return;

  const rect = rightPanel.getBoundingClientRect();
  const doc = screenToDoc(e.clientX - rect.left, e.clientY - rect.top);
  const threshold = 8 / state.zoom;

  // Check if near an edge handle
  let edge = null;
  if (Math.abs(doc.x - r.x) < threshold && Math.abs(doc.y - (r.y + r.h/2)) < threshold) edge = "left";
  else if (Math.abs(doc.x - (r.x + r.w)) < threshold && Math.abs(doc.y - (r.y + r.h/2)) < threshold) edge = "right";
  else if (Math.abs(doc.y - r.y) < threshold && Math.abs(doc.x - (r.x + r.w/2)) < threshold) edge = "top";
  else if (Math.abs(doc.y - (r.y + r.h)) < threshold && Math.abs(doc.x - (r.x + r.w/2)) < threshold) edge = "bottom";

  if (!edge) return;

  resizeDrag = { edge, startX: e.clientX, startY: e.clientY, origX: r.x, origY: r.y, origW: r.w, origH: r.h };
  e.stopPropagation();
  e.preventDefault();
}, { capture: true });

window.addEventListener("mousemove", (e) => {
  if (!resizeDrag) return;
  const r = state.redactions[state.activeRedaction];
  if (!r) return;

  const dx = (e.clientX - resizeDrag.startX) / state.zoom;
  const dy = (e.clientY - resizeDrag.startY) / state.zoom;

  if (resizeDrag.edge === "left") {
    r.x = Math.round(resizeDrag.origX + dx);
    r.w = Math.max(10, Math.round(resizeDrag.origW - dx));
  } else if (resizeDrag.edge === "right") {
    r.w = Math.max(10, Math.round(resizeDrag.origW + dx));
  } else if (resizeDrag.edge === "top") {
    r.y = Math.round(resizeDrag.origY + dy);
    r.h = Math.max(10, Math.round(resizeDrag.origH - dy));
  } else if (resizeDrag.edge === "bottom") {
    r.h = Math.max(10, Math.round(resizeDrag.origH + dy));
  }

  // Update gap width in overrides to match
  if (r.overrides && (resizeDrag.edge === "left" || resizeDrag.edge === "right")) {
    r.overrides.gapWidth = r.w;
    gapValue.textContent = String(Math.round(r.w));
  }

  renderCanvas();
});

window.addEventListener("mouseup", () => {
  if (resizeDrag) resizeDrag = null;
});
```

**Step 3: Manual verification**

Run the app, create a redaction, verify:
- Small blue squares appear at the edges of the active redaction
- Dragging left/right edges changes width
- Dragging top/bottom edges changes height
- Gap width display updates when width changes

**Step 4: Commit**

```bash
git add unredact/static/main.js unredact/static/canvas.js
git commit -m "feat: drag handles for resizing redaction bounding boxes"
```

---

### Task 12: Style the Detect Redactions button and polish

**Files:**
- Modify: `unredact/static/index.html` or CSS file (style the button)
- Modify: `unredact/static/main.js` (clean up any loose ends)

**Step 1: Style the button**

Check how existing styles are defined (inline in HTML or in a CSS file). Match the existing visual style. The button should be prominent but not overwhelming.

**Step 2: Clean up OCR status indicator**

Add a small status line near the detect button showing OCR progress:

```html
<div id="ocr-status" style="font-size:12px;color:#888;margin-bottom:4px;"></div>
```

Update `startOcrSSE` to show page-by-page progress in this element.

**Step 3: Handle edge case: double-click before OCR completes**

In the dblclick handler, if `!state.ocrReady`, still allow the spot (it will return analysis: null), but show a toast: "OCR still processing — redaction created without analysis. Try again after OCR completes."

**Step 4: Commit**

```bash
git add unredact/static/index.html unredact/static/main.js
git commit -m "style: polish detect button, OCR status, and edge cases"
```

---

### Task 13: Run full test suite and verify

**Files:** None (verification only)

**Step 1: Run all Python tests**

Run: `pytest tests/ -v --timeout=120`
Expected: All PASS

**Step 2: Run the app end-to-end**

Run: `make run` (or equivalent)

Verify the complete workflow:
1. Upload a PDF
2. OCR runs automatically, "Detect Redactions" button enables
3. Double-click a black bar → redaction appears analyzed with popover
4. Text edit bar shows left/right context
5. Inline text inputs appear on the canvas
6. Ctrl+drag adjusts offset, Shift+drag adjusts gap
7. Drag handles resize the box
8. Solver runs and returns results
9. Click "Detect Redactions" → full LLM pipeline runs, all redactions appear
10. Delete key removes a redaction
11. Page navigation works
12. Export (Ctrl+E) includes all data

**Step 3: Commit any fixes**

```bash
git add -A && git commit -m "fix: address issues found in end-to-end verification"
```
