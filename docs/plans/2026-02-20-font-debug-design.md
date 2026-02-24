# Font Matching Debug Visualization

## Problem

Font detection sometimes works well and sometimes poorly, but there's no visibility into what the pixel scoring is doing — which candidates were tried, how they compared, or why the winner was chosen.

## Design

### Activation

- Env var: `UNREDACT_DEBUG=1`
- When set, the font detection pipeline saves debug images to `debug/` in the project root
- When unset, zero overhead — no image generation

### Output Structure

```
debug/
  font-match-<timestamp>/
    line-00_crop.png                              # raw page crop
    line-00_rank-1_TimesNewRoman_14px_0.87.png    # top candidate composite
    line-00_rank-2_Georgia_14px_0.82.png
    line-00_rank-3_Arial_13px_0.71.png
    line-00_rank-4_LiberationSerif_14px_0.70.png
    line-00_rank-5_DejaVuSerif_15px_0.68.png
    line-00_summary.png                           # all 5 tiled horizontally
    line-01_crop.png
    ...
```

### Composite Image Layout (per candidate)

Each candidate image is a vertical stack:

1. **Header**: font name, size, score — white text on dark background
2. **Page crop**: binarized ink (white on black)
3. **Rendered candidate**: binarized ink (white on black)
4. **Overlap map**: color-coded comparison
   - Green pixels: both page and rendered have ink (correct match)
   - Red pixels: page has ink but rendered doesn't (missing ink)
   - Blue pixels: rendered has ink but page doesn't (extra ink)

### Summary Image

All top 5 composites tiled side-by-side in a single image, ranked left-to-right by score. Quick visual comparison at a glance.

### Implementation Location

New module: `unredact/pipeline/font_debug.py`

- `_debug_enabled()` — check env var
- `_init_debug_dir()` — create timestamped subfolder, return Path
- `_save_candidate_image()` — generate one composite for a candidate
- `_save_summary_image()` — tile top N composites horizontally
- `_save_crop_image()` — save the raw page crop

### Integration Points

Modify `_full_search()` to collect top-5 candidates (not just the best). When debug is enabled, after each scoring function completes its search, call the debug save functions.

Both `_score_font_line_pixel()` and `_score_font_masked_pixel()` produce the same intermediate data (page_bin, rendered_bin) needed for the overlap visualization. The debug module will re-render the top 5 candidates to generate the images (slight re-work, but keeps the hot path clean).

### .gitignore

Add `debug/` to `.gitignore`.
