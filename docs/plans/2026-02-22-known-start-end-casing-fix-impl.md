# Known Start/End Casing Fix Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Fix incorrect pixel width calculation when using known_start/known_end in full name and single name dictionary solving.

**Architecture:** Extract a helper `_case_unknown_portion()` that applies capitalized casing with word-boundary awareness to the unknown portion of a name. Replace the blanket `.lower()` override in both solve functions.

**Tech Stack:** Python, PIL/Pillow fonts, pytest

---

### Task 1: Add failing test for multi-word full name casing

**Files:**
- Modify: `tests/test_full_name_dictionary.py`

**Step 1: Write the failing test**

Add to `TestSolveFullNameDictionary`:

```python
@patch("unredact.pipeline.word_filter._get_associate_variants")
def test_known_start_multiword_casing(self, mock_variants):
    """Unknown portion of multi-word name should preserve word-boundary casing.

    Name: 'john doe', known_start='jo', unknown='hn Doe' (not 'hn doe').
    The width measurement must use 'hn Doe' not 'hn doe'.
    """
    mock_variants.return_value = ["john doe"]

    # Mock: measure "ohn Doe" with left context "o" from known_start
    # font.getlength("ohn Doe") should be called, NOT "ohn doe"
    font = _mock_font({
        "ohn Doe": 42.0,   # correct cased unknown
        "ohn doe": 38.0,   # wrong (current bug: all-lowercase)
    })

    results = solve_full_name_dictionary(
        font, 42.0, 1.0, known_start="o", casing="capitalized",
    )
    texts = [r.text for r in results]
    assert "John Doe" in texts
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_full_name_dictionary.py::TestSolveFullNameDictionary::test_known_start_multiword_casing -v`
Expected: FAIL — currently measures "ohn doe" (38.0) which is outside tolerance of 42.0 ± 1.0

---

### Task 2: Add helper `_case_unknown_portion` and fix `solve_full_name_dictionary`

**Files:**
- Modify: `unredact/pipeline/dictionary.py`

**Step 1: Add the helper function after `_apply_casing`**

```python
def _case_unknown_portion(unknown: str, known_start: str, casing: str) -> str:
    """Apply casing to the unknown portion of a name, respecting word boundaries.

    When known_start is set and casing is 'capitalized', the unknown text
    starts mid-word — its first fragment should be lowercase, but subsequent
    words (after spaces) should be title-cased.
    """
    if casing == "uppercase":
        return unknown.upper()
    if casing != "capitalized":
        return unknown  # lowercase — no change

    if not known_start:
        return unknown.title()

    # If known_start ends with a space, unknown starts a new word
    if known_start.endswith(" "):
        return unknown.title()

    # Unknown starts mid-word: first fragment lowercase, rest title-cased
    parts = unknown.split(" ")
    result = [parts[0].lower()]
    for part in parts[1:]:
        result.append(part.title() if part else "")
    return " ".join(result)
```

**Step 2: Replace lines 121-123 in `solve_full_name_dictionary`**

Change:
```python
            unknown_display = _apply_casing(unknown_raw, casing)
            if known_start and casing == "capitalized":
                unknown_display = unknown_raw.lower()
```

To:
```python
            unknown_display = _case_unknown_portion(unknown_raw, known_start, casing)
```

**Step 3: Fix `effective_left` to use cased known_start (line 125)**

Change:
```python
            effective_left = known_start[-1] if known_start else left_context
```

To:
```python
            effective_left = _apply_casing(known_start, casing)[-1] if known_start else left_context
```

**Step 4: Run the failing test to verify it passes**

Run: `pytest tests/test_full_name_dictionary.py -v`
Expected: ALL PASS

---

### Task 3: Add failing test for single name casing and fix `solve_name_dictionary`

**Files:**
- Modify: `tests/test_name_dictionary.py`
- Modify: `unredact/pipeline/dictionary.py`

**Step 1: Write the failing test**

Add to `TestSolveNameDictionary`:

```python
@patch("unredact.pipeline.word_filter._get_associate_firsts")
@patch("unredact.pipeline.word_filter._get_associate_lasts")
def test_known_start_capitalized_casing(self, mock_lasts, mock_firsts):
    """With known_start and capitalized casing, unknown part is mid-word lowercase."""
    mock_firsts.return_value = ["john"]
    mock_lasts.return_value = []

    # known_start="j", unknown="ohn", should measure "ohn" (lowercase)
    font = _mock_font({"ohn": 21.0})

    results = solve_name_dictionary(
        font, 21.0, 1.0, known_start="j", casing="capitalized",
    )
    texts = [r.text for r in results]
    assert "John" in texts
```

**Step 2: Run test to verify it passes (existing behavior is correct for single names)**

Run: `pytest tests/test_name_dictionary.py::TestSolveNameDictionary::test_known_start_capitalized_casing -v`
Expected: PASS (single-word lowercase already works)

**Step 3: Replace lines 218-225 in `solve_name_dictionary`**

Change:
```python
        # Apply same casing to the unknown portion
        if casing == "uppercase":
            unknown_display = unknown.upper()
        elif casing == "capitalized":
            # Mid-word after known_start: lowercase. At word start: title case.
            unknown_display = unknown.lower() if known_start else unknown.title()
        else:
            unknown_display = unknown
```

To:
```python
        unknown_display = _case_unknown_portion(unknown, known_start, casing)
```

**Step 4: Fix `effective_left` to use cased known_start (line 229)**

Change:
```python
        effective_left = known_start[-1] if known_start else left_context
```

To:
```python
        effective_left = _apply_casing(known_start, casing)[-1] if known_start else left_context
```

**Step 5: Run all tests to verify nothing broke**

Run: `pytest tests/test_name_dictionary.py tests/test_full_name_dictionary.py -v`
Expected: ALL PASS

---

### Task 4: Commit

**Step 1: Commit the changes**

```bash
git add unredact/pipeline/dictionary.py tests/test_full_name_dictionary.py tests/test_name_dictionary.py
git commit -m "fix: known_start/end casing for multi-word name width calculation"
```
