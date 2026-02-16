# Spot Redaction Detection Design

## Problem

The current auto-detection pipeline (`detect_redactions.py`) using OpenCV morphological operations has three failure modes:
1. **False positives** — dark areas (shadows, header backgrounds) detected as redactions
2. **Multi-line merging** — adjacent redactions on different lines merged into one box
3. **Missed redactions** — small or low-contrast redactions not detected

## Solution

Replace auto-detection with user-guided spot detection. The user double-clicks directly on a redaction box. The backend uses connected-component analysis to find the exact extent of that dark region, then automatically runs OCR + font detection on the surrounding text.

## Backend

### New endpoint: `POST /api/redaction/spot`

**Request:**
```json
{
  "doc_id": "abc123",
  "page": 1,
  "click_x": 342,
  "click_y": 187
}
```

Coordinates in page-image pixels (same coordinate space as existing redaction boxes).

**Algorithm:**
1. Load the rasterized page image (already cached from upload)
2. Convert to grayscale, binarize with threshold 40 (dark pixels -> white mask, rest -> black)
3. Run `cv2.connectedComponents()` on the binary mask
4. Look up which label is at `(click_y, click_x)` in the label matrix
5. If label is 0 (background) -> return error
6. Extract all pixels with that label, compute bounding box
7. If bounding box area < 100px -> return error (noise/speck)
8. Pass the bounding box into the existing `analyze_redaction()` pipeline
9. Return the combined response

**Success response:**
```json
{
  "box": {"x": 310, "y": 180, "w": 150, "h": 16},
  "segments": [...],
  "gap": {...},
  "font": {...},
  "chars": [...],
  "line": {...},
  "offset_x": 5.2,
  "offset_y": 0.0
}
```

**Error response:**
```json
{"error": "no_redaction_found"}
```

### Upload changes

Remove `detect_redactions()` call from the upload handler. Upload only rasterizes pages. `GET /api/doc/{id}/page/{p}/data` returns `{redactions: []}` initially. The `detect_redactions.py` module stays in the codebase but is no longer called.

## Frontend

### Double-click handler (replaces zoom)

1. Double-click on canvas triggers spot detection
2. Convert click from screen space to page-image space (inverse transform for zoom/pan)
3. If click lands on an existing redaction -> activate it (current behavior), skip spot detection
4. Send `POST /api/redaction/spot`
5. While waiting: show loading indicator
6. On success:
   - Create redaction entry from returned `box`
   - Populate `analysis` from response
   - Set status to `analyzed`, initialize `overrides`
   - Add to `state.redactions` and sidebar list
   - Activate the redaction (center canvas, show green overlay, open toolbar)
7. On error: show toast "No redaction found at click point"

Redaction ID: `"m" + Date.now().toString(36)` (same as former manual marking).

### Input remapping

| Modifier | Old action | New action |
|----------|-----------|------------|
| Double-click | Zoom 2x | Spot redaction |
| Shift+drag | Manual mark redaction | Adjust gap width |
| Ctrl+drag | Adjust overlay offset | Adjust overlay offset (unchanged) |
| Ctrl+Shift+drag | Adjust gap width | (removed) |
| Plain drag | Pan | Pan (unchanged) |

### Sidebar

Starts empty. Redactions appear as user double-clicks them. Sorted top-to-bottom, left-to-right after each addition.
