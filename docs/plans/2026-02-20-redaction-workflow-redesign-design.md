# Redaction Workflow Redesign

## Problem

Two issues blocking release:

1. Auto-detection runs on upload — it should be a separate action
2. Double-click (spot) redactions are dead-ends — no analysis, no popover, no solver access

Root cause: manually-created redactions skip the analysis pipeline entirely and arrive as `"unanalyzed"` with no `analysis` object or `overrides`. The popover, text edit bar, canvas rendering, and solver all require these to exist.

## Design Decisions

- **Primary workflow is manual**: users double-click redactions they want to investigate
- **Auto-detect ("Scan for Redactions") is secondary**: a dev/AI tool for batch processing
- **Inline editing on canvas**: left/right text editable directly on the document, not in a disconnected panel
- **OCR runs at upload time**: cached and available for both manual and auto-detect flows
- **Text cleaning replaces image masking**: filter OCR characters by overlap with redaction boxes instead of whiting out image regions

## User Journey

### 1. Upload
- Clean upload screen, drop a PDF
- PDF rasterizes, page image displays
- OCR runs automatically in background (brief "Processing..." indicator)
- When OCR completes, document is ready for interaction

### 2. Mark & Analyze Redactions (Primary Flow)
- User examines document (zoom, pan)
- **Double-click on any black bar**:
  - Brief loading indicator
  - Backend finds bounding box (OpenCV connected components)
  - Backend runs mini-pipeline: font detection, left/right text extraction from cached OCR, pixel alignment
  - Returns fully analyzed redaction
- Canvas zooms to center the redaction line
- Inline editing activates:
  - Left/right text render at detected positions on the canvas
  - Editable `<input>` overlays at text positions (feels like typing on the document)
  - Gap (redaction box) highlighted with pixel width shown
  - Small floating toolbar: font selector, size, offset fine-tune, gap width, **Solve** button

### 3. Fine-Tune & Solve
- Adjust left/right text by typing directly on the document
- Ctrl+drag: fine-tune text offset (X/Y pixel alignment)
- Shift+drag: adjust gap width
- Drag box edges: resize the redaction bounding box (new)
- Click **Solve**: solver streams results, matched against associates

### 4. Navigate Between Redactions
- Left panel lists all marked redactions for the page
- Click one to pan and open inline editor
- Each shows status: analyzed, solved, with solution preview

### 5. Scan for Redactions (Secondary/Dev Flow)
- Button at top of left panel, above redaction list
- Disabled until OCR completes, then enabled
- Runs full LLM pipeline (uses cached OCR): detection + analysis for all redactions on page
- Each arrives analyzed and ready to solve
- Same inline editing experience when clicked

## Backend Changes

### New endpoint: `GET /api/doc/{id}/ocr` (SSE)
- Runs Tesseract on each page, caches `OcrLine[]` in document state
- Emits `page_ocr_complete` events (lightweight metadata, not full OCR data)
- Emits `ocr_complete` when all pages done
- Returns cached results if already run

### New function: `clean_ocr_chars(ocr_lines, redaction_bboxes)`
- Filters OCR characters: excludes any whose bounding box overlaps a redaction bbox
- Returns only clean characters
- Replaces `detect_font_masked()` image-masking approach
- Used by both auto-detect pipeline and spot pipeline

### New function: `analyze_spot_redaction(doc_id, page, bbox)`
- Takes known bounding box + cached OCR data
- Runs: `clean_ocr_chars()` → font detection (on clean chars) → left/right text extraction → pixel alignment
- Skips LLM detection (we already have the box)
- Returns full redaction with `analysis` + `overrides` + `status: "analyzed"`

### Modified: `POST /api/doc/{id}/page/{page}/spot`
- After finding bbox via OpenCV, calls `analyze_spot_redaction()`
- Returns fully populated redaction object

### Modified: `analyze_page()`
- Accepts pre-computed OCR data (no longer runs Tesseract internally)
- Uses `clean_ocr_chars()` instead of `detect_font_masked()`

### Modified: `GET /api/doc/{id}/analyze` (SSE)
- No longer auto-triggered on upload
- Uses cached OCR data from the `/ocr` endpoint
- Otherwise same flow: LLM detection → region finding → analysis

## Frontend Changes

### Upload flow (`main.js`)
- `uploadFile()` → `POST /api/upload` → page image loads
- Automatically starts `GET /api/doc/{id}/ocr` SSE → shows "Running OCR..." status
- When OCR completes: document ready, "Scan for Redactions" button enables
- No auto-analysis

### Inline editing (new)
- When a redaction activates, HTML `<input>` elements overlay the canvas at text positions
- Inputs transform with viewport (zoom/pan) — positioned using the same coordinate system
- Styled to blend with document (semi-transparent background, matched font size)
- Typing updates `overrides.leftText`/`overrides.rightText` and redraws canvas
- Floating toolbar anchors near the redaction with font/size/offset/gap/solve controls

### Double-click spot (revised)
- Response now includes full analysis data
- Creates redaction with `status: "analyzed"`, populated `analysis`/`overrides`
- Activates inline editor immediately

### Detect Redactions button (new)
- Top of left panel, above redaction list
- Disabled until OCR completes
- Click triggers `GET /api/doc/{id}/analyze` SSE
- Progress indicator while running

### Drag handles for bbox resize (new)
- When a redaction is active, small handles at box edges
- Drag to resize `x`, `y`, `w`, `h`

## Interaction Summary

| Interaction | Action |
|---|---|
| Double-click black bar | Create & analyze redaction, open inline editor |
| Ctrl+drag | Fine-tune text offset (X/Y alignment) |
| Shift+drag | Adjust gap width |
| Drag box edges | Resize redaction bounding box |
| Click redaction in list | Pan to it, open inline editor |
| "Scan for Redactions" | Run full LLM auto-detect pipeline |

## What Stays the Same

- Canvas rendering for overlays (boxes, preview text, solutions)
- Solver (Rust DFS, dictionary modes, SSE streaming)
- Associate matching + tier badges
- Viewport (zoom, pan, touch support)
- Page navigation
- Export (Ctrl+E)
