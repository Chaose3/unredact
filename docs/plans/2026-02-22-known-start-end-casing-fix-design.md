# Fix known_start/known_end pixel width calculation

**Date**: 2026-02-22

## Problem

When `known_start` or `known_end` is set and casing is `"capitalized"`, both
`solve_full_name_dictionary` and `solve_name_dictionary` force the entire
unknown portion to lowercase. This causes incorrect pixel width measurements
for multi-word names because subsequent words lose their capital letters.

**Example**: Full name "John Doe", `known_start = "Jo"`, casing = `"capitalized"`

- `unknown_raw = "hn doe"`
- Current: `unknown_display = "hn doe"` (all lowercase)
- Correct: `unknown_display = "hn Doe"` (mid-word lowercase, word-boundary title case)

The width of "hn doe" vs "hn Doe" differs by several pixels due to the D/d
difference, causing valid matches to be rejected or wrong matches to be accepted.

### Secondary issue

`effective_left = known_start[-1]` uses the user's raw input casing. If the
user types "JO" but the rendered text is "Jo", the kerning context character
("O" vs "o") is wrong.

## Fix

### Smart word-boundary casing

Replace the blanket `.lower()` override with logic that:

1. If `known_start` doesn't end with a space (unknown starts mid-word):
   lowercase the first word fragment, title-case subsequent words.
2. If `known_start` ends with a space (unknown starts at a word boundary):
   apply normal `.title()` casing.

### Cased effective_left

Apply the same casing logic to `known_start` before extracting the last
character for kerning context.

## Scope

- `unredact/pipeline/dictionary.py`:
  - `solve_full_name_dictionary` (lines 121-123)
  - `solve_name_dictionary` (lines 218-223)
- No frontend changes needed
