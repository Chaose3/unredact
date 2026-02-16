# Spot Redaction Detection Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace auto-detection with user-guided spot detection via double-click, using connected-component flood fill.

**Architecture:** New `spot_redaction()` function uses OpenCV connected components to find the dark region at a click point. New `/api/redaction/spot` endpoint chains this with existing analysis logic. Frontend replaces double-click-to-zoom with spot detection, remaps shift+drag from manual marking to gap width adjustment.

**Tech Stack:** Python/FastAPI (backend), OpenCV (connected components), vanilla JS (frontend)

---

### Task 1: Backend — Add `spot_redaction()` function

**Files:**
- Modify: `unredact/pipeline/detect_redactions.py`

**Step 1: Add spot_redaction function after the existing detect_redactions function**

Add at the end of `unredact/pipeline/detect_redactions.py` (after line 74):

```python
def spot_redaction(image: Image.Image, click_x: int, click_y: int) -> Redaction | None:
    """Find a redaction box at a specific click point using connected components.

    Args:
        image: PIL Image of a document page.
        click_x: X coordinate of the click in page-image pixels.
        click_y: Y coordinate of the click in page-image pixels.

    Returns:
        Redaction object if a dark region is found, None otherwise.
    """
    arr = np.array(image.convert("RGB"))
    gray = cv2.cvtColor(arr, cv2.COLOR_RGB2GRAY)
    _, binary = cv2.threshold(gray, 40, 255, cv2.THRESH_BINARY_INV)

    num_labels, labels = cv2.connectedComponents(binary)

    if click_y < 0 or click_y >= labels.shape[0] or click_x < 0 or click_x >= labels.shape[1]:
        return None

    label = int(labels[click_y, click_x])
    if label == 0:
        return None

    coords = cv2.findNonZero((labels == label).astype(np.uint8))
    x, y, w, h = cv2.boundingRect(coords)

    if w * h < 100:
        return None

    return Redaction(id=uuid.uuid4().hex[:8], x=int(x), y=int(y), w=int(w), h=int(h))
```

**Step 2: Commit**

```bash
git add unredact/pipeline/detect_redactions.py
git commit -m "feat: add spot_redaction() using connected components"
```

---

### Task 2: Backend — Extract analysis helper and add `/api/redaction/spot` endpoint

**Files:**
- Modify: `unredact/app.py:19-21` (imports)
- Modify: `unredact/app.py:150-230` (extract helper from analyze endpoint)
- Add new endpoint after the analyze endpoint

**Step 1: Update imports in `app.py` (line 20)**

Change:
```python
from unredact.pipeline.detect_redactions import detect_redactions
```
To:
```python
from unredact.pipeline.detect_redactions import detect_redactions, spot_redaction
```

**Step 2: Extract analysis logic into helper function**

Insert before the `@app.post("/api/redaction/analyze")` decorator (before line 150):

```python
def _run_analysis(page_img: Image.Image, rx: int, ry: int, rw: int, rh: int) -> dict | None:
    """Run OCR + font detection for a redaction box. Returns analysis dict or None."""
    pad_y = rh
    crop_y1 = max(0, ry - pad_y)
    crop_y2 = min(page_img.height, ry + rh + pad_y)
    line_crop = page_img.crop((0, crop_y1, page_img.width, crop_y2))

    lines = ocr_page(line_crop)
    if not lines:
        return None

    redaction_cy = (ry - crop_y1) + rh / 2
    best_line = min(lines, key=lambda l: abs((l.y + l.h / 2) - redaction_cy))

    font_match = detect_font_for_line(best_line)

    segments = []
    gap = {"x": rx, "w": rw}

    left_chars = [c for c in best_line.chars if c.x + c.w <= rx]
    right_chars = [c for c in best_line.chars if c.x >= rx + rw]

    left_text = "".join(c.text for c in left_chars).rstrip()
    right_text = "".join(c.text for c in right_chars).lstrip()

    pil_font = font_match.to_pil_font()
    if left_text:
        left_rendered_width = pil_font.getlength(left_text)
        offset_x = float(rx - left_rendered_width - best_line.x)
    else:
        offset_x = 0.0
    offset_y = 0.0

    if left_text:
        lx = left_chars[0].x
        lw = (left_chars[-1].x + left_chars[-1].w) - lx
        segments.append({"text": left_text, "x": lx, "w": lw})
    if right_text:
        rx2 = right_chars[0].x
        rw2 = (right_chars[-1].x + right_chars[-1].w) - rx2
        segments.append({"text": right_text, "x": rx2, "w": rw2})

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
        "offset_x": round(offset_x, 1),
        "offset_y": round(offset_y, 1),
    }
```

**Step 3: Simplify existing analyze endpoint to use helper**

Replace the body of the `analyze_redaction` function (lines 152-230) with:

```python
@app.post("/api/redaction/analyze")
async def analyze_redaction(req: AnalyzeRequest):
    doc = _docs.get(req.doc_id)
    if not doc or req.page not in doc["pages"]:
        return JSONResponse({"error": "not found"}, status_code=404)

    page_img = doc["pages"][req.page]["original"]
    rx, ry, rw, rh = req.redaction["x"], req.redaction["y"], req.redaction["w"], req.redaction["h"]

    result = _run_analysis(page_img, rx, ry, rw, rh)
    if result is None:
        return JSONResponse({"error": "no text detected near redaction"}, status_code=422)
    return result
```

**Step 4: Add SpotRequest model and endpoint**

Add after the analyze endpoint:

```python
class SpotRequest(BaseModel):
    doc_id: str
    page: int
    click_x: int
    click_y: int


@app.post("/api/redaction/spot")
async def spot_redaction_endpoint(req: SpotRequest):
    doc = _docs.get(req.doc_id)
    if not doc or req.page not in doc["pages"]:
        return JSONResponse({"error": "not found"}, status_code=404)

    page_img = doc["pages"][req.page]["original"]
    redaction = spot_redaction(page_img, req.click_x, req.click_y)
    if redaction is None:
        return JSONResponse({"error": "no_redaction_found"}, status_code=422)

    result = _run_analysis(page_img, redaction.x, redaction.y, redaction.w, redaction.h)
    response = {"box": {"x": redaction.x, "y": redaction.y, "w": redaction.w, "h": redaction.h}}
    if result:
        response.update(result)
    return response
```

**Step 5: Commit**

```bash
git add unredact/app.py
git commit -m "feat: add /api/redaction/spot endpoint with analysis helper"
```

---

### Task 3: Backend — Remove auto-detection from upload

**Files:**
- Modify: `unredact/app.py:79-105` (upload_pdf function)

**Step 1: Remove detect_redactions call from upload_pdf**

Replace lines 91-97 of `upload_pdf`:

```python
    page_data = {}
    for i, page_img in enumerate(pages, start=1):
        redactions = detect_redactions(page_img)
        page_data[i] = {
            "original": page_img,
            "redactions": redactions,
        }
```

With:

```python
    page_data = {}
    for i, page_img in enumerate(pages, start=1):
        page_data[i] = {
            "original": page_img,
            "redactions": [],
        }
```

**Step 2: Commit**

```bash
git add unredact/app.py
git commit -m "feat: remove auto-detection from upload pipeline"
```

---

### Task 4: Frontend — Add toast notification system

**Files:**
- Modify: `unredact/static/index.html` (add toast container)
- Modify: `unredact/static/style.css` (add toast styles)
- Modify: `unredact/static/app.js` (add showToast function)

**Step 1: Add toast container to HTML**

In `index.html`, add before the closing `</body>` tag (before line 150):

```html
  <div id="toast-container"></div>
```

**Step 2: Add toast CSS**

Append to `style.css`:

```css
/* Toast notifications */
#toast-container {
  position: fixed;
  bottom: 2rem;
  left: 50%;
  transform: translateX(-50%);
  z-index: 100;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  pointer-events: none;
}

.toast {
  background: rgba(30, 30, 50, 0.95);
  color: #e0e0e0;
  padding: 0.6rem 1.2rem;
  border-radius: 6px;
  border: 1px solid #333;
  font-size: 0.85rem;
  backdrop-filter: blur(6px);
  animation: toast-in 0.2s ease-out, toast-out 0.3s ease-in 2.5s forwards;
}

.toast.error {
  border-color: #d32f2f;
  color: #ef9a9a;
}

@keyframes toast-in {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes toast-out {
  from { opacity: 1; }
  to { opacity: 0; }
}
```

**Step 3: Add showToast function to app.js**

Add near the top of `app.js`, after the DOM element declarations (after line 53):

```javascript
const toastContainer = document.getElementById("toast-container");

function showToast(message, type = "info") {
  const el = document.createElement("div");
  el.className = `toast ${type}`;
  el.textContent = message;
  toastContainer.appendChild(el);
  setTimeout(() => el.remove(), 3000);
}
```

**Step 4: Commit**

```bash
git add unredact/static/index.html unredact/static/style.css unredact/static/app.js
git commit -m "feat: add toast notification system"
```

---

### Task 5: Frontend — Replace double-click-to-zoom with spot redaction

**Files:**
- Modify: `unredact/static/app.js:870-877` (dblclick handler)

**Step 1: Replace the double-click handler**

Replace the existing dblclick handler (lines 870-877):

```javascript
// Double-click to zoom in
rightPanel.addEventListener("dblclick", (e) => {
  if (popover.contains(e.target) || fontToolbar.contains(e.target) || textEditBar.contains(e.target)) return;
  const rect = rightPanel.getBoundingClientRect();
  const sx = e.clientX - rect.left;
  const sy = e.clientY - rect.top;
  zoomTo(state.zoom * 2, sx, sy, true);
});
```

With:

```javascript
// Double-click to spot-detect a redaction
rightPanel.addEventListener("dblclick", async (e) => {
  if (popover.contains(e.target) || fontToolbar.contains(e.target) || textEditBar.contains(e.target)) return;
  if (!state.docId) return;

  const rect = rightPanel.getBoundingClientRect();
  const sx = e.clientX - rect.left;
  const sy = e.clientY - rect.top;
  const doc = screenToDoc(sx, sy);

  // If clicking on an existing redaction, just activate it (handled by canvas mousedown)
  const hit = hitTestRedaction(doc.x, doc.y);
  if (hit) return;

  const clickX = Math.round(doc.x);
  const clickY = Math.round(doc.y);

  showToast("Detecting redaction...");

  try {
    const resp = await fetch("/api/redaction/spot", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        doc_id: state.docId,
        page: state.currentPage,
        click_x: clickX,
        click_y: clickY,
      }),
    });

    if (!resp.ok) {
      const err = await resp.json();
      showToast(err.error === "no_redaction_found" ? "No redaction found at click point" : "Detection failed", "error");
      return;
    }

    const data = await resp.json();
    const box = data.box;
    const id = "m" + Date.now().toString(36);

    state.redactions[id] = {
      id,
      x: box.x,
      y: box.y,
      w: box.w,
      h: box.h,
      page: state.currentPage,
      status: data.segments ? "analyzed" : "unanalyzed",
      analysis: data.segments ? data : null,
      solution: null,
      preview: null,
    };

    if (data.segments) {
      state.redactions[id].overrides = {
        fontId: data.font.id,
        fontSize: data.font.size,
        offsetX: data.offset_x || 0,
        offsetY: data.offset_y || 0,
        gapWidth: data.gap.w,
        leftText: data.segments.length > 0 ? data.segments[0].text : "",
        rightText: data.segments.length > 1 ? data.segments[1].text : "",
      };
    }

    renderRedactionList();
    renderCanvas();
    activateRedaction(id);
  } catch (e) {
    console.error("Spot detection error:", e);
    showToast("Detection failed: " + e.message, "error");
  }
});
```

**Step 2: Commit**

```bash
git add unredact/static/app.js
git commit -m "feat: replace double-click zoom with spot redaction detection"
```

---

### Task 6: Frontend — Remove shift+drag manual marking, remap shift+drag to gap width

**Files:**
- Modify: `unredact/static/app.js:1299-1372` (remove shift+drag marking)
- Modify: `unredact/static/app.js:1374-1416` (remap ctrl+shift to shift for gap width)

**Step 1: Remove the entire shift+drag manual marking block**

Delete lines 1299-1372 (from `// ── Manual redaction marking (Shift+drag) ──` through the final `mouseup` handler for `drawDrag`). This includes the `drawDrag` variable and all three event listeners.

**Step 2: Remap the modifier drag handlers**

Replace the existing ctrl+drag block (lines 1374-1416, now renumbered after deletion):

```javascript
// ── Ctrl+drag offset / Ctrl+Shift+drag gap width ──

let ctrlDrag = null;

canvas.addEventListener("mousedown", (e) => {
  if (!e.ctrlKey || e.button !== 0) return;
  const r = state.redactions[state.activeRedaction];
  if (!r?.overrides) return;

  ctrlDrag = {
    startX: e.clientX,
    startY: e.clientY,
    startOffsetX: r.overrides.offsetX,
    startOffsetY: r.overrides.offsetY,
    startGapWidth: r.overrides.gapWidth,
    widthMode: e.shiftKey,
  };
  e.stopPropagation();
  e.preventDefault();
}, { capture: true });
```

With:

```javascript
// ── Ctrl+drag offset / Shift+drag gap width ──

let modDrag = null;

canvas.addEventListener("mousedown", (e) => {
  if ((!e.ctrlKey && !e.shiftKey) || e.button !== 0) return;
  const r = state.redactions[state.activeRedaction];
  if (!r?.overrides) return;

  modDrag = {
    startX: e.clientX,
    startY: e.clientY,
    startOffsetX: r.overrides.offsetX,
    startOffsetY: r.overrides.offsetY,
    startGapWidth: r.overrides.gapWidth,
    widthMode: e.shiftKey && !e.ctrlKey,
  };
  e.stopPropagation();
  e.preventDefault();
}, { capture: true });
```

And update the mousemove and mouseup handlers to use `modDrag` instead of `ctrlDrag`:

```javascript
window.addEventListener("mousemove", (e) => {
  if (!modDrag) return;
  const r = state.redactions[state.activeRedaction];
  if (!r?.overrides) return;

  const dx = (e.clientX - modDrag.startX) / state.zoom;
  const dy = (e.clientY - modDrag.startY) / state.zoom;

  if (modDrag.widthMode) {
    r.overrides.gapWidth = Math.max(1, modDrag.startGapWidth + dx);
    gapValue.textContent = Math.round(r.overrides.gapWidth);
  } else {
    r.overrides.offsetX = modDrag.startOffsetX + dx;
    r.overrides.offsetY = modDrag.startOffsetY + dy;
    updatePosDisplay();
  }
  renderCanvas();
});

window.addEventListener("mouseup", () => {
  if (modDrag) modDrag = null;
});
```

**Step 3: Commit**

```bash
git add unredact/static/app.js
git commit -m "feat: remap shift+drag to gap width, remove manual marking"
```
