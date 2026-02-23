# LLM Validation of Solve Results Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a "Validate" button that sends all solve results to Claude Sonnet, scores them on contextual fit, and reorders the result list by score.

**Architecture:** New backend endpoint reads buffered solve results, sends them to Anthropic API with surrounding text context, returns scored results. Frontend shows a validate panel with editable context fields, replaces results with LLM-scored list.

**Tech Stack:** Python/FastAPI, Anthropic SDK (claude-sonnet-4-6, tool_use), vanilla JS frontend.

---

### Task 1: Create the LLM validation module

**Files:**
- Create: `unredact/pipeline/llm_validate.py`
- Test: `tests/test_llm_validate.py`

**Step 1: Write the failing test**

Create `tests/test_llm_validate.py`:

```python
"""Tests for LLM solve result validation."""

import pytest

from unredact.pipeline.llm_validate import build_validation_prompt, SCORE_TOOL


class TestBuildValidationPrompt:
    def test_basic_prompt_structure(self):
        candidates = ["Smith", "house", "running"]
        prompt = build_validation_prompt(
            left_context="Dear Mr.",
            right_context=", we are writing",
            candidates=candidates,
        )
        assert "Dear Mr." in prompt
        assert ", we are writing" in prompt
        assert "1. Smith" in prompt
        assert "2. house" in prompt
        assert "3. running" in prompt
        assert "[REDACTED]" in prompt

    def test_empty_context(self):
        prompt = build_validation_prompt(
            left_context="",
            right_context="",
            candidates=["word"],
        )
        assert "1. word" in prompt

    def test_score_tool_schema(self):
        assert SCORE_TOOL["name"] == "score_candidates"
        schema = SCORE_TOOL["input_schema"]
        assert "scores" in schema["properties"]
        items = schema["properties"]["scores"]["items"]
        assert "index" in items["properties"]
        assert "score" in items["properties"]
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_llm_validate.py -v`
Expected: FAIL — module doesn't exist

**Step 3: Implement the module**

Create `unredact/pipeline/llm_validate.py`:

```python
"""LLM-based validation of solve candidates.

Sends candidate words to Claude Sonnet with surrounding text context.
Returns a contextual fit score (0-100) for each candidate.
"""

from __future__ import annotations

import logging
import os

import anthropic

log = logging.getLogger(__name__)

_VALIDATION_MODEL = "claude-sonnet-4-6"

SCORE_TOOL = {
    "name": "score_candidates",
    "description": "Score each candidate word on how well it fits the redacted gap contextually.",
    "input_schema": {
        "type": "object",
        "properties": {
            "scores": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "index": {
                            "type": "integer",
                            "description": "1-based index of the candidate in the list.",
                        },
                        "score": {
                            "type": "integer",
                            "description": "Contextual fit score from 0-100.",
                            "minimum": 0,
                            "maximum": 100,
                        },
                    },
                    "required": ["index", "score"],
                },
            },
        },
        "required": ["scores"],
    },
}


def build_validation_prompt(
    left_context: str,
    right_context: str,
    candidates: list[str],
) -> str:
    """Build the prompt for LLM validation of solve candidates."""
    numbered = "\n".join(f"{i+1}. {c}" for i, c in enumerate(candidates))
    return (
        "You are analyzing a redacted document. A section of text has been "
        "blacked out. The text surrounding the redaction reads:\n\n"
        f'Left context: "{left_context}"\n'
        "[REDACTED]\n"
        f'Right context: "{right_context}"\n\n'
        "Below is a list of candidate words/phrases that fit the redacted "
        "space by pixel width. Score each from 0-100 on how well it fits "
        "contextually:\n\n"
        "- 90-100: Near-certain fit (grammatically correct, semantically "
        "meaningful, contextually expected)\n"
        "- 60-89: Plausible (makes sense but not the most likely)\n"
        "- 30-59: Unlikely (grammatically possible but doesn't make much sense)\n"
        "- 0-29: Very poor fit (nonsensical, wrong part of speech, doesn't "
        "work in context)\n\n"
        'Example: If left context is "Dear Mr." and right is ", we are '
        'writing to inform you":\n'
        '- "Smith" -> 95 (common surname, perfect fit)\n'
        '- "house" -> 5 (not a surname, makes no sense after "Mr.")\n\n'
        f"Candidates:\n{numbered}"
    )


# Reuse the client from llm_detect
from unredact.pipeline.llm_detect import _get_client


async def validate_candidates(
    left_context: str,
    right_context: str,
    candidates: list[str],
) -> list[int]:
    """Score candidates using LLM. Returns list of scores in same order as candidates.

    Raises on API error — caller should handle.
    """
    if not candidates:
        return []

    prompt = build_validation_prompt(left_context, right_context, candidates)
    client = _get_client()

    response = await client.messages.create(
        model=_VALIDATION_MODEL,
        max_tokens=16384,
        tools=[SCORE_TOOL],
        tool_choice={"type": "tool", "name": "score_candidates"},
        messages=[{"role": "user", "content": prompt}],
    )

    # Parse tool response
    scores = [0] * len(candidates)
    for block in response.content:
        if block.type == "tool_use" and block.name == "score_candidates":
            for item in block.input.get("scores", []):
                idx = item.get("index", 0) - 1  # 1-based to 0-based
                score = item.get("score", 0)
                if 0 <= idx < len(candidates):
                    scores[idx] = score
            break

    return scores
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_llm_validate.py -v`
Expected: PASS (tests only check prompt building and schema, not API calls)

**Step 5: Commit**

```bash
git add unredact/pipeline/llm_validate.py tests/test_llm_validate.py
git commit -m "feat: add LLM validation module with prompt and tool schema"
```

---

### Task 2: Add validation endpoint to app.py

**Files:**
- Modify: `unredact/app.py` (add endpoint after `get_solve_results`)
- Test: `tests/test_app.py`

**Step 1: Write the failing test**

Add to `tests/test_app.py`. This test mocks the Anthropic API call to avoid real API usage:

```python
@pytest.mark.anyio
async def test_validate_endpoint(monkeypatch):
    """POST /api/solve/{id}/validate should return LLM-scored results."""
    from unredact.app import _solve_results
    from unredact.pipeline import llm_validate

    fake_id = "validate_test"
    _solve_results[fake_id] = [
        {"text": "Smith", "width_px": 50.0, "error_px": 0.1, "source": "names"},
        {"text": "house", "width_px": 50.0, "error_px": 0.2, "source": "words"},
        {"text": "running", "width_px": 50.0, "error_px": 0.3, "source": "words"},
    ]

    # Mock validate_candidates to avoid real API call
    async def mock_validate(left, right, candidates):
        return [95, 10, 5]

    monkeypatch.setattr(llm_validate, "validate_candidates", mock_validate)

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.post(f"/api/solve/{fake_id}/validate", json={
            "left_context": "Dear Mr.",
            "right_context": ", we are writing",
        })
        assert resp.status_code == 200
        data = resp.json()
        assert len(data["results"]) == 3
        assert data["total"] == 3
        # Should be sorted by llm_score descending
        assert data["results"][0]["text"] == "Smith"
        assert data["results"][0]["llm_score"] == 95
        assert data["results"][1]["llm_score"] == 10
        assert data["results"][2]["llm_score"] == 5

    _solve_results.pop(fake_id, None)


@pytest.mark.anyio
async def test_validate_unknown_solve_id():
    """POST /api/solve/{id}/validate with unknown id should 404."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.post("/api/solve/nonexistent/validate", json={
            "left_context": "test",
            "right_context": "test",
        })
        assert resp.status_code == 404
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_app.py::test_validate_endpoint -v`
Expected: FAIL — endpoint doesn't exist

**Step 3: Add the endpoint**

Add after the `get_solve_results` endpoint in `unredact/app.py` (around line 586):

```python
class ValidateRequest(BaseModel):
    left_context: str = ""
    right_context: str = ""


@app.post("/api/solve/{solve_id}/validate")
async def validate_solve(solve_id: str, req: ValidateRequest):
    if solve_id not in _solve_results:
        return JSONResponse({"error": "solve not found"}, status_code=404)

    from unredact.pipeline.llm_validate import validate_candidates

    buf = _solve_results[solve_id]
    candidates = [r["text"] for r in buf]

    try:
        scores = await validate_candidates(
            req.left_context, req.right_context, candidates,
        )
    except Exception as e:
        return JSONResponse({"error": str(e)}, status_code=500)

    # Merge scores and sort by score descending
    scored = []
    for i, result in enumerate(buf):
        scored.append({**result, "llm_score": scores[i] if i < len(scores) else 0})
    scored.sort(key=lambda x: x["llm_score"], reverse=True)

    return {"results": scored, "total": len(scored)}
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_app.py::test_validate_endpoint tests/test_app.py::test_validate_unknown_solve_id -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/app.py tests/test_app.py
git commit -m "feat: add POST /api/solve/{id}/validate endpoint"
```

---

### Task 3: Add Validate button and context panel to HTML

**Files:**
- Modify: `unredact/static/index.html:149-157` (solve actions area)
- Modify: `unredact/static/dom.js` (add new element references)

**Step 1: Add HTML elements**

In `index.html`, after the `solve-accept` button (line 152) and before the status span, add:

```html
<button id="solve-validate" class="solve-btn" hidden>Validate</button>
```

After the `solve-load-more` button (line 156), add the validate panel:

```html
<div id="validate-panel" hidden>
  <label>Left context <input type="text" id="validate-left" spellcheck="false" autocomplete="off"></label>
  <label>Right context <input type="text" id="validate-right" spellcheck="false" autocomplete="off"></label>
  <button id="validate-run" class="solve-btn">Run</button>
</div>
```

**Step 2: Add DOM references**

Add to `unredact/static/dom.js` after the `solveLoadMore` line:

```javascript
export const solveValidate = /** @type {HTMLButtonElement} */ (document.getElementById("solve-validate"));
export const validatePanel = document.getElementById("validate-panel");
export const validateLeft = /** @type {HTMLInputElement} */ (document.getElementById("validate-left"));
export const validateRight = /** @type {HTMLInputElement} */ (document.getElementById("validate-right"));
export const validateRun = /** @type {HTMLButtonElement} */ (document.getElementById("validate-run"));
```

**Step 3: Commit**

```bash
git add unredact/static/index.html unredact/static/dom.js
git commit -m "feat: add validate button and context panel HTML"
```

---

### Task 4: Wire up validate UI in solver.js

**Files:**
- Modify: `unredact/static/solver.js`

**Step 1: Add imports**

Add `solveValidate`, `validatePanel`, `validateLeft`, `validateRight`, `validateRun` to the import from `./dom.js`.

**Step 2: Show validate button when solve completes**

In `handleSolveEvent`, in the `done` handler (after `activeEventSource = null`), add:

```javascript
if (totalFound > 0) {
  solveValidate.hidden = false;
}
```

**Step 3: Hide validate button on new solve**

In `startSolve`, after the existing reset lines, add:

```javascript
solveValidate.hidden = true;
validatePanel.hidden = true;
```

**Step 4: Add validate button click handler — toggle the panel**

Clicking "Validate" toggles the validate panel and pre-populates the context inputs:

```javascript
function showValidatePanel() {
  const id = state.activeRedaction;
  if (!id) return;
  const r = state.redactions[id];
  if (!r || !r.analysis) return;

  const a = r.analysis;
  const o = r.overrides || {};
  const leftText = o.leftText ?? (a.segments.length > 0 ? a.segments[0].text : "");
  const rightText = o.rightText ?? (a.segments.length > 1 ? a.segments[1].text : "");

  validateLeft.value = leftText;
  validateRight.value = rightText;
  validatePanel.hidden = false;
}
```

**Step 5: Add runValidation function**

```javascript
async function runValidation() {
  if (!currentSolveId) return;
  validateRun.disabled = true;
  validateRun.textContent = "Validating...";
  solveStatus.textContent = `Validating ${totalFound} results...`;

  try {
    const resp = await fetch(`/api/solve/${currentSolveId}/validate`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        left_context: validateLeft.value,
        right_context: validateRight.value,
      }),
    });
    if (!resp.ok) throw new Error("Validation failed");
    const data = await resp.json();

    // Replace results with scored list
    solveResults.innerHTML = "";
    displayedCount = 0;
    solveLoadMore.hidden = true;

    const redactionId = state.activeRedaction;
    for (const item of data.results) {
      renderScoredResult(item, redactionId);
    }

    solveStatus.textContent = `Validated. ${data.total} results scored.`;
    validatePanel.hidden = true;
  } catch (err) {
    solveStatus.textContent = "Validation error: " + err.message;
  } finally {
    validateRun.disabled = false;
    validateRun.textContent = "Run";
  }
}
```

**Step 6: Add renderScoredResult function**

This is like `handleSolveEvent` for `match` status, but also renders the LLM score badge:

```javascript
function renderScoredResult(data, redactionId) {
  // Reuse handleSolveEvent for the base result rendering
  handleSolveEvent({ status: "match", ...data }, redactionId);

  // Add score badge to the last-inserted result
  if (data.llm_score !== undefined) {
    const results = solveResults.children;
    // Find the result we just inserted (it has this text)
    for (const el of results) {
      const textEl = el.querySelector(".result-text");
      if (textEl && textEl.textContent === data.text && !el.querySelector(".llm-score")) {
        const badge = document.createElement("span");
        badge.className = "llm-score";
        const score = data.llm_score;
        if (score >= 70) badge.classList.add("score-high");
        else if (score >= 30) badge.classList.add("score-mid");
        else badge.classList.add("score-low");
        badge.textContent = String(score);
        badge.title = "LLM contextual fit score";
        el.prepend(badge);
        break;
      }
    }
  }
}
```

**Step 7: Wire up in initSolver**

Add to `initSolver`:

```javascript
solveValidate.addEventListener("click", showValidatePanel);
validateRun.addEventListener("click", runValidation);
```

**Step 8: Commit**

```bash
git add unredact/static/solver.js
git commit -m "feat: wire up validate button and LLM scoring in solver.js"
```

---

### Task 5: Add CSS for score badges and validate panel

**Files:**
- Modify: `unredact/static/style.css`

**Step 1: Add styles**

Append to `style.css`:

```css
/* LLM validation score badges */
.llm-score {
  flex-shrink: 0;
  padding: 2px 6px;
  border-radius: 3px;
  font-size: 0.75rem;
  font-weight: bold;
  margin-right: 4px;
}

.llm-score.score-high {
  background: var(--green, #2d8a4e);
  color: #fff;
}

.llm-score.score-mid {
  background: var(--yellow, #d29922);
  color: #000;
}

.llm-score.score-low {
  background: var(--red, #cf222e);
  color: #fff;
}

/* Validate panel */
#validate-panel {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  padding: 8px;
  background: var(--surface-2, #161b22);
  border-radius: 4px;
  margin-top: 4px;
}

#validate-panel label {
  display: flex;
  flex-direction: column;
  flex: 1;
  min-width: 120px;
  font-size: 0.75rem;
  color: var(--muted, #8b949e);
}

#validate-panel input {
  background: var(--surface-1, #0d1117);
  border: 1px solid var(--border, #30363d);
  color: var(--text, #e6edf3);
  padding: 4px 8px;
  border-radius: 3px;
  font-size: 0.85rem;
  margin-top: 2px;
}

#validate-panel .solve-btn {
  align-self: flex-end;
}
```

**Step 2: Verify the CSS variables exist**

Check `style.css` for existing `--green`, `--yellow`, `--red` variables. If they don't exist, use the fallback values already in the CSS above.

**Step 3: Commit**

```bash
git add unredact/static/style.css
git commit -m "feat: add CSS for LLM score badges and validate panel"
```

---

### Task 6: Manual integration test

**No files changed — verification only.**

**Step 1: Run all tests**

Run: `pytest tests/test_app.py tests/test_llm_validate.py -v`
Expected: All PASS

**Step 2: Manual test**

1. `make dev`
2. Upload a document, select a redaction
3. Run a word solve with Standard vocabulary, tolerance 2px
4. Wait for results to complete
5. Verify "Validate" button appears
6. Click "Validate" — verify context panel appears with pre-populated left/right text
7. Edit context if desired, click "Run"
8. Verify results are replaced with scored list, ordered by LLM score
9. Verify score badges appear (green/yellow/red)
10. Verify clicking a scored result still previews it on the canvas

**Step 3: Commit any fixes found during manual testing**
