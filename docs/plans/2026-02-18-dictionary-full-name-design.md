# Dictionary-Based Full Name Solver

## Problem

The Rust DFS full-name solver enumerates all character combinations matching a target pixel width, then filters against name dictionaries. This produces millions of gibberish results (e.g., "Aa Aaaaaiii") and caused OOM crashes before the max_results cap was added. Even with the cap, the results are largely useless without dictionary filtering, and the search space is enormous.

## Solution

Replace the primary full-name solve path with dictionary-style matching against focused name lists extracted from the associates database. This mirrors how email solving works: measure known strings against the target width, return matches.

## Data Files

`build_associates.py` gains a new output step producing two files:

- `unredact/data/associate_first_names.txt` — one lowercase name per line
- `unredact/data/associate_last_names.txt` — one lowercase name per line

**Sources:**
- Canonical person names from `persons_registry.json`, split into first (first token) and last (last token)
- Nickname expansions from the `NICKNAMES` mapping (e.g., "jeff" for "jeffrey", "bill" for "william")
- Filtered to alpha-only strings of length >= 2, deduplicated, sorted

**Expected sizes:** ~740 first names, ~1,055 last names. Cartesian product: ~717K combinations.

## Solve Pipeline

### New function: `solve_full_name_dictionary()`

Location: `unredact/pipeline/dictionary.py`

Parameters: `font`, `target_width`, `tolerance`, `left_context`, `right_context`, `uppercase_only`

Behavior:
1. Load first and last name lists (cached at module level, like emails)
2. For each `(first, last)` in the Cartesian product:
   - Apply casing: `first.title() + " " + last.title()` for capitalized, `(first + " " + last).upper()` for caps
   - Measure width with `font.getlength()`, accounting for left/right context
   - If `abs(width - target) <= tolerance`, collect as a `SolveResult`
3. Return results sorted by `(error, text)`

Names are stored lowercase. Casing is applied at solve time based on the mode.

### Associate variant matching

When running a full-name solve, also check all multi-word associate name variants from `associates.json` (full names, nickname+last, initial+last like "J. Epstein"). These ~5K entries are dictionary-matched the same way. This is auto-included in the full-name solve, not a separate mode.

### Integration in app.py

When `charset_name` is `full_name_capitalized` or `full_name_caps`:
1. Run `solve_full_name_dictionary()` — stream results with `source: "names"`
2. Run associate variant matching — stream results with `source: "associates"`
3. Then run the existing Rust enumerate solver (existing behavior preserved, just runs after dictionary results)

Deduplication via `found_texts` set prevents duplicates across the three phases.

## Performance

~717K Cartesian product combinations + ~5K associate variants. PIL's `font.getlength()` handles ~500K-1M calls/sec. Expected solve time: ~1-1.5 seconds in Python. Acceptable for an interactive tool. If profiling shows this is too slow, the dictionary logic could move to Rust as a follow-up.

## What stays

The Rust DFS full-name solver (`/solve/full-name` endpoint, `full_name.rs`) remains available as the enumerate mode fallback. The max_results cap prevents OOM. The dictionary approach becomes the primary path.

## Testing

- Unit test for `solve_full_name_dictionary()` with mock font
- Data test verifying `build_associates.py` generates expected name files
- Known names check (e.g., "jeffrey"/"jeff" in first names, "epstein" in last names)
- Integration test adapting `test_full_name_stress.py` for the dictionary path
