# Constraint Solver Design

## Overview

Given a pixel-width gap from a human-verified redaction in the frontend, the solver enumerates all strings that perfectly fill that gap using precise font metrics (including kerning). Results stream to the frontend in real-time via SSE.

## API Contract

### `POST /api/solve` (SSE response)

Request body:

```json
{
  "doc_id": "abc123",
  "page": 1,
  "line_index": 3,
  "font_id": "times-new-roman",
  "font_size": 46,
  "gap_width_px": 142.5,
  "tolerance_px": 0.0,
  "left_context": "o",
  "right_context": " ",
  "hints": {
    "charset": "lowercase",
    "min_length": 4,
    "max_length": 8
  },
  "mode": "enumerate"
}
```

Modes: `"enumerate"` (full search), `"dictionary"` (wordlist only), `"both"` (dictionary first, then enumerate).

SSE response stream:

```
data: {"status": "running", "checked": 150000, "found": 0}
data: {"status": "match", "text": "Smith", "width_px": 142.3, "error_px": 0.2}
data: {"status": "done", "total_checked": 4200000, "total_found": 2}
```

### `DELETE /api/solve/{solve_id}`

Cancels a running solve by terminating its worker pool.

### Dictionary endpoints

- `POST /api/dictionary` — upload a wordlist (name + entries)
- `GET /api/dictionary` — list loaded dictionaries
- `DELETE /api/dictionary/{name}` — remove a wordlist

## Width Table & Font Metrics Engine

The core data structure is a precomputed 2D width table for a given font + size + charset.

### Width table construction

For every `(prev_char, next_char)` pair in the allowed charset:

```python
width_table[i][j] = font.getlength(prev + next) - font.getlength(prev)
```

This captures kerning implicitly — `getlength("AV") - getlength("A")` gives the advance of "V" after "A", which is narrower than "V" alone.

### Boundary handling

- **Left edge**: `left_edge[char_idx] = font.getlength(left_context + char) - font.getlength(left_context)` — 1D array for the first redacted character.
- **Right edge**: `right_edge[char_idx] = font.getlength(char + right_context) - font.getlength(char) - font.getlength(right_context)` — kerning correction for the last redacted character against the right context.

### Pruning bounds

- `min_advance[prev_char_idx]` — narrowest possible next character after this char
- `max_advance[prev_char_idx]` — widest possible next character after this char

These enable instant branch pruning: "even filling the rest with the widest/narrowest chars can't hit the target."

### Caching

Width tables keyed by `(font_id, font_size, charset)` and cached in memory. Same font/size across multiple solve requests reuses the table.

## Branch-and-Bound Solver

### Core algorithm (per worker)

Each worker receives an assigned prefix and searches its subtree:

1. Start from the prefix's accumulated width and last character
2. Try each allowed next character, look up advance from width table
3. If `accumulated_width > target + tolerance` — prune (overshoot)
4. If `remaining_budget < min_advance[last_char]` — prune (can't fit anything)
5. If `accumulated_width + max_advance[last_char] * chars_remaining < target - tolerance` — prune (can't reach target)
6. If current length is within `[min_length, max_length]`, check if `abs(final_width - target) <= tolerance` with right-edge correction
7. Recurse

### Parallelization

1. Main process builds width table, generates all depth-2 prefixes (e.g., "aa".."zz" = 676 for lowercase)
2. Prune prefixes whose accumulated width already overshoots
3. Distribute surviving prefixes across `ProcessPoolExecutor(max_workers=os.cpu_count())`
4. Workers report progress via shared `multiprocessing.Value` counter
5. Main process collects results from futures, streams via SSE

### Prefix depth

- Alphabet <= 52 chars: depth 2 (up to ~2700 jobs)
- Goal: enough jobs for load balancing without overhead domination

## Dictionary Mode

Linear scan over wordlist entries, measuring rendered width of each:

```python
width = font.getlength(left_ctx + entry + right_ctx) - font.getlength(left_ctx) - right_correction
if abs(width - target) <= tolerance:
    yield (entry, width)
```

### Dictionary sources

- **Epstein associates**: curated list shipped with the project (public record names)
- **User-supplied wordlists**: uploaded via API, one entry per line, stored per-session

### "Both" mode

1. Dictionary scan first (milliseconds)
2. Stream dictionary matches immediately
3. Kick off full enumeration in parallel
4. Stream enumeration matches, deduplicating against dictionary hits

## Frontend Integration

### Solve button

Appears in the floating toolbar when a line has a redaction gap (2+ segments). Targets the gap after the active segment.

### Solve panel

- **Hints**: charset dropdown (lowercase / uppercase / mixed / alphanumeric / custom), min/max length, tolerance slider (0 to ±5px in 0.5 steps)
- **Mode toggle**: Enumerate / Dictionary / Both
- **Progress bar**: nodes checked, updated via SSE
- **Results list**: matches appear real-time, sorted by error (closest to 0 first). Clicking a result inserts it into the gap for visual verification on the canvas.

### Wiring

```javascript
const es = new EventSource(`/api/solve?${params}`);
es.onmessage = (e) => {
    const data = JSON.parse(e.data);
    if (data.status === "match") addToResultsList(data);
    if (data.status === "running") updateProgress(data);
    if (data.status === "done") es.close();
};
```

### Cancel

Closing the panel or clicking "Stop" sends `DELETE /api/solve/{solve_id}` to terminate workers.
