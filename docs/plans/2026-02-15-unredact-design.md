# Unredact: AI-Powered Redaction Analysis for Epstein Files

## Overview

A local web application that analyzes redacted PDFs from the Epstein case files, detects fonts and redaction boxes, and uses character width constraints + Claude API to guess what text is hidden under redactions. Users visually verify guesses via a green text overlay rendered on top of the original document — if the overlay aligns with the surrounding visible text, the guess fits.

## Approach: Image-First Pipeline

All PDFs are treated as images. Each page is rasterized to a high-DPI PNG, then processed through computer vision (OCR, redaction detection, font analysis). This is the right approach because the Epstein files are scanned documents — there is no extractable text layer.

## Core Pipeline

```
PDF Upload → Rasterize → OCR + Font Detection → Redaction Detection → Constraint Solving → Claude Guessing → Visual Overlay
```

### Stage 1: Font Matching Overlay (Foundation)

The critical proof-of-concept. If we can render green text that perfectly overlaps the real document text, everything else follows.

1. **Rasterize** PDF pages to high-DPI PNGs using pdf2image (poppler backend)
2. **OCR** visible text with Tesseract at character-level granularity, extracting bounding boxes for every character
3. **Detect the font** by rendering candidate fonts (Times New Roman, Arial, Courier, Calibri, etc.) at various sizes and comparing against the actual character shapes/widths measured from OCR
4. **Calibrate rendering** by rendering a known visible word with the detected font, comparing pixel width to the actual, and computing a correction factor
5. **Render green overlay** of the known OCR'd text on top of the document image. The overlay should align character-for-character with the original. This is the validation that our font pipeline works.

### Stage 2: Redaction Detection

1. **OpenCV pipeline** to find black filled rectangles on each page
2. **Classify each redaction**: which text line it belongs to, what visible text is before/after it
3. **Measure exact pixel dimensions** of each redaction box
4. Display redaction bounding boxes highlighted in the web UI

### Stage 3: Constraint Solver

1. Build a **per-character width table** from the calibrated font (not just averages — `M` is ~2x wider than `i`)
2. For each redaction: compute which character counts and string widths are physically valid
3. Handle proportional vs monospace fonts (monospace simplifies to box_width / char_width)
4. Account for multi-word redactions (space widths measured from visible word gaps) and email characters (`@`, `.`, `<`, `>`)
5. Filter candidates from the known associates database by width constraint alone
6. Allow ±8% tolerance for scan distortion and kerning differences

### Stage 4: Claude Integration

1. **Batch all redactions per page** into a single Claude API call for efficiency and cross-reference reasoning
2. Prompt includes:
   - Full page text with `[REDACTED_N]` markers
   - Character width constraints for each redaction
   - Field type inference (name, email, etc.)
   - Known associates database entries
3. Claude returns ranked guesses with confidence scores and reasoning
4. **Composite confidence**: `final = 0.4 * llm_confidence + 0.6 * width_fit_score` (width fit weighted higher because it's objective)

### Stage 5: Polish for Release

- User can type custom guesses with live green overlay preview
- Toggle between guesses and watch overlay shift in real-time
- Export annotated PDFs with guesses rendered in red
- JSON report export
- Batch processing for multiple files

## Visual Verification Overlay

The key UX innovation. For each guess:

1. Render the entire text line (visible text + guessed text) in green, semi-transparent, on top of the original document image
2. Position the overlay to start exactly where the real text starts (from OCR baseline)
3. If the green text aligns with the real text on both sides of the redaction, the guess is the correct width
4. If green drifts after the redaction, the guess is too wide or narrow

Users can click through ranked guesses and watch alignment change. They can also type their own guesses with live overlay feedback.

Automated alignment scoring: compare green overlay character positions against OCR'd positions of visible text after the redaction. Characters aligning within a few pixels = high fit score.

## Known Associates Database

A curated JSON file shipped with the tool:

```json
{
  "associates": [
    {
      "name": "Ghislaine Maxwell",
      "aliases": ["GM", "G. Maxwell"],
      "role": "associate",
      "locations": ["NYC", "Palm Beach", "Little St. James"]
    }
  ]
}
```

Sourced from public reporting and previously unredacted documents. Users can add their own entries.

## Tech Stack

- **Language**: Python
- **Web framework**: FastAPI + uvicorn
- **Frontend**: Vanilla HTML/CSS/JS (no framework)
- **PDF rasterization**: pdf2image + poppler
- **OCR**: pytesseract + Tesseract
- **Image processing**: OpenCV (redaction detection), Pillow (font rendering)
- **LLM**: Anthropic Claude API (BYOK — user provides their own key)
- **Deployment**: Local-only. Users install via pip or Docker.

## Project Layout

```
unredact/
├── pyproject.toml
├── README.md
├── unredact/
│   ├── __init__.py
│   ├── app.py              # FastAPI server
│   ├── pipeline/
│   │   ├── rasterize.py    # PDF to images
│   │   ├── ocr.py          # Tesseract OCR + character bounding boxes
│   │   ├── font_detect.py  # Font identification + calibration
│   │   ├── redaction.py    # Black rectangle detection
│   │   ├── solver.py       # Width constraint solver
│   │   └── guesser.py      # Claude API integration
│   ├── data/
│   │   └── associates.json # Known names database
│   ├── fonts/              # Bundled font files for rendering
│   └── static/
│       ├── index.html
│       ├── style.css
│       └── app.js          # Overlay rendering, interaction
└── tests/
```

## Dependencies

- `pdf2image` + poppler system package
- `pytesseract` + Tesseract system package
- `opencv-python`
- `Pillow`
- `fastapi`
- `uvicorn`
- `anthropic`

## Scope & Constraints

- **Epstein files focused**: font profiles and associates database tuned for this corpus
- **Local-only**: no server deployment, no accounts, no storage
- **BYOK**: users must provide their own Anthropic API key
- **Privacy**: PDFs never leave the machine except text context sent to Claude API for guessing
