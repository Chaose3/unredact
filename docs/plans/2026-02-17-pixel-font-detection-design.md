# Pixel-Based Font Detection

## Problem

The current font detection uses word-width mean absolute error (MAE) to match fonts. This metric is too coarse — different fonts at different sizes can produce nearly identical word widths for short words. In practice, this means Arial at 44px can outscore Times New Roman at 50px even when the text is clearly TNR, because the system never looks at the actual glyph shapes.

## Solution

Replace word-width MAE with pixel-based normalized cross-correlation (NCC). Render each candidate font+size, compare the rendered pixels against the actual page image crop, and pick the candidate with highest pixel correlation.

## Core Algorithm

### Scoring Function

1. **Extract line region** from the page image (crop to OCR line bounding box)
2. **Binarize both images** — convert page crop and rendered text to binary (Otsu's threshold) to eliminate anti-aliasing noise and color/grayscale differences
3. **Render candidate text** using the candidate font+size onto a same-sized canvas
4. **Compute normalized cross-correlation (NCC)** between the two binary images (score from -1 to 1, higher is better)
5. **Alignment tolerance** — slide rendered image ±2px vertically and horizontally, take best NCC score to account for OCR bbox imprecision

### Why Binarize?

Page images have grayscale text with scan artifacts. Rendered text has clean anti-aliased edges. Binarization (Otsu's threshold) reduces both to the same representation: black ink vs white background. This makes NCC robust to image quality differences.

### NCC Formula

Only requires numpy:
```
ncc = sum((A - mean_A) * (B - mean_B)) / (std_A * std_B * N)
```

No OpenCV or scikit-image dependency needed.

## Size Search Optimization

### OCR-Constrained Size Range

Use the OCR line height to constrain the size search. For most fonts: `font_size ≈ line_height * 0.8–1.2`. A 60px-tall line searches 48–72px instead of 20–78px, cutting search space by ~60%.

### Coarse-to-Fine

1. **Coarse pass**: constrained range at 4px steps (~7 sizes per font × 8 fonts = ~56 evaluations)
2. **Fine pass**: ±3px around the winner at 1px steps (7 evaluations, one font)
3. **Total**: ~63 evaluations instead of 240

## Files Changed

- `unredact/pipeline/font_detect.py` — Replace `_score_font_line()` with pixel NCC scoring. Update `_full_search()` and `_fine_search()` for higher-is-better metric and page image parameter.
- `unredact/app.py` — Pass page image crop to font detection functions.
- `unredact/pipeline/overlay.py` — No changes (consumes same `FontMatch` interface).
- `unredact/pipeline/ocr.py` — No changes.

## Testing

- Unit: render known text in TNR at 50px, verify detection finds TNR at ~50px (not Arial)
- Unit: same for Arial, Courier New
- Integration: real PDF page crop, verify detection matches expected font
- Visual: overlay alignment test still produces reasonable scores
