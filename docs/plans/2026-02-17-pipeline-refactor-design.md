# Pipeline Refactor: LLM-Guided Redaction Detection

## Problem

The current pipeline runs OCR per-redaction (slow, inconsistent crop boundaries), uses OpenCV blind (merges adjacent redactions, false positives), and does font detection on neighboring lines (wrong font when lines differ). OCR artifacts like `[` and `|` near redactions poison font detection and text extraction.

## Solution

Restructure the pipeline to: OCR once per page, use an LLM to identify redactions from the OCR text, guide OpenCV with the LLM's output for precise bounding boxes, and mask redactions for font detection.

## Data Flow

```
Upload PDF
  → Rasterize pages (existing)
  → For each page (streamed via SSE):
    1. OCR full page → list[OcrLine] with char positions
    2. LLM pass (Haiku) → identifies redaction locations + clean text segments
    3. Guided OpenCV → pixel-precise bounding box per LLM-identified redaction
    4. Font detection → mask redaction white, Dice score full masked line
    5. Build analysis → segments, font, offsets per redaction
  → Store results in _docs, stream progress to frontend
```

## LLM Pass

**Model:** Claude Haiku (fast, cheap, sufficient for pattern detection)

**Input:** OCR text with line numbers and char positions. The LLM identifies broken text patterns (artifacts like `[`, `|`, unnatural gaps) that indicate redactions.

**Output (structured via tool use):**
```json
{
  "redactions": [
    {
      "line": 2,
      "left_text": "let",
      "right_text": "or",
      "left_char_index": 18,
      "right_char_index": 23
    }
  ]
}
```

The char indices map back to OCR positions, giving approximate x-coordinates for the guided OpenCV search. The clean `left_text`/`right_text` are used directly, avoiding artifact poisoning.

## Guided OpenCV

For each LLM-identified redaction:
1. Get x-range from last char of `left_text` to first char of `right_text` (from OCR data)
2. Get y-range from the line's vertical extent
3. Add padding (±10px)
4. Search only that region for a dark rectangle (threshold + contour)
5. Return pixel-precise bounding box

Benefits over blind OpenCV:
- No merging — each redaction searched independently
- No false positives — only look where the LLM flagged
- Correct count — LLM determines the number of redactions

## Font Detection with Masking

For each redacted line:
1. Crop the full line from the page image
2. Paint all redaction boxes on this line white
3. Use the LLM's clean text (no artifacts) for scoring
4. Run pixel-based Dice scoring against the masked crop
5. Cache result per line (multiple redactions share the font)

## API Changes

Upload becomes the main pipeline entry point. The old `/api/redaction/spot` and `/api/redaction/analyze` endpoints are removed — all analysis is pre-computed.

- `POST /api/upload` → returns `doc_id`, opens SSE stream for progress
- SSE events: `page_ocr_done`, `page_redactions_found`, `page_analysis_complete`
- `GET /api/doc/{id}/page/{p}/data` → returns all pre-computed redaction data

## Files Changed

- `unredact/app.py` — New upload pipeline with SSE streaming, remove spot/analyze endpoints
- `unredact/pipeline/llm_detect.py` — New: LLM-based redaction detection from OCR text
- `unredact/pipeline/detect_redactions.py` — Refactor: guided search within a region instead of blind whole-page scan
- `unredact/pipeline/font_detect.py` — Add masking support, use clean text from LLM
- `unredact/pipeline/ocr.py` — No changes (already returns what we need)
- `unredact/static/` — Frontend updates to consume pre-computed data instead of click-to-analyze
