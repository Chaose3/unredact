# Pixel-Aligned Text Positioning

## Problem

Font detection now correctly identifies the font, but the overlay text position is wrong:
1. Characters near the redaction edge are cut off ("Sent" becomes "Se") because the char filter uses `c.x + c.w <= rx` which misses chars whose right edge extends past the redaction
2. `offset_x` is calculated from OCR positions + PIL rendering width, but browser canvas rendering differs — resulting in misaligned overlay
3. `offset_y = 0` (hardcoded) — no vertical alignment at all

## Solution

### Fix 1: Character Filtering

Change char filtering to use center-point: `c.x + c.w / 2 < rx` instead of `c.x + c.w <= rx`. This captures characters whose center is left of the redaction, recovering "nt" from "Sent". Similarly for right: `c.x + c.w / 2 > rx + rw`.

### Fix 2: Pixel-Based Alignment Function

New `_align_text_to_page()` function:
1. Render the left text with the detected font onto a canvas
2. Binarize both rendered text and page image crop
3. Slide rendered text across the page crop (±20px X, ±10px Y) to find position with maximum Dice overlap
4. Return `(offset_x, offset_y)` — the pixel-perfect position

### Fix 3: Integration

Replace the OCR-based offset calculation in `_run_analysis()` with the pixel alignment result. The frontend already applies `line.x + offsetX` and `line.y + offsetY` — no frontend changes needed.

## Files Changed

- `unredact/pipeline/font_detect.py` — Add `_align_text_to_page()` function
- `unredact/app.py` — Fix char filtering, replace offset calculation with pixel alignment
