# Redaction Approval Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** When a user accepts a solver result, merge left + solution + right into a persistent blue text overlay on the canvas, replacing the old "solved" status with a new "approved" status.

**Architecture:** Pure frontend change across 4 JS modules (solver.js, canvas.js, main.js, popover.js) and CSS. The "solved" status is replaced by "approved". A new `drawRedactionApproved()` renders the full merged line in blue. Clicking an approved redaction re-opens the analyze workflow with the previous solution pre-populated.

**Tech Stack:** Vanilla JS (ES modules), Canvas2D, CSS

---

### Task 1: Update acceptSolution to produce "approved" status

**Files:**
- Modify: `unredact/static/solver.js:197-212`

**Step 1: Update `acceptSolution()` function**

Replace the current `acceptSolution` function in `solver.js` (lines 197-212) with:

```javascript
export function acceptSolution() {
  const id = state.activeRedaction;
  if (!id) return;
  const r = state.redactions[id];
  if (!r || !r.preview) return;

  const o = r.overrides || {};
  const leftText = o.leftText ?? "";
  const rightText = o.rightText ?? "";

  r.status = "approved";
  const solFontName = state.fonts.find(f => f.id === (o.fontId || r.analysis.font.id))?.name || r.analysis.font.name;
  r.solution = {
    text: r.preview,
    fontName: solFontName,
    fontSize: o.fontSize || r.analysis.font.size,
  };
  r.approvedText = leftText + r.preview + rightText;
  r.preview = null;
}
```

Key changes from the old version:
- `r.status = "approved"` instead of `"solved"`
- Computes `r.approvedText` by merging left + preview + right
- Everything else stays the same

**Step 2: Verify no other code sets status to "solved"**

Search for `"solved"` in JS files. The only place should be the old `acceptSolution` which we just changed.

**Step 3: Commit**

```bash
git add unredact/static/solver.js
git commit -m "feat: acceptSolution produces 'approved' status with merged text"
```

---

### Task 2: Replace drawRedactionSolution with drawRedactionApproved

**Files:**
- Modify: `unredact/static/canvas.js:90-119` (renderCanvas), `canvas.js:260-319` (drawRedactionSolution)

**Step 1: Replace `drawRedactionSolution` with `drawRedactionApproved`**

Delete the entire `drawRedactionSolution` function (lines 260-319) and replace with:

```javascript
/**
 * @param {import('./types.js').Redaction} r
 * @param {boolean} isActive
 */
function drawRedactionApproved(r, isActive) {
  if (!r.analysis || !r.approvedText) return;
  const a = r.analysis;
  const o = r.overrides || {};
  const fontName = state.fonts.find(f => f.id === (o.fontId ?? a.font.id))?.name ?? a.font.name;
  const fontSize = o.fontSize ?? a.font.size;

  const startX = a.line.x + (o.offsetX ?? 0);
  const startY = a.line.y + (o.offsetY ?? 0);

  ctx.textBaseline = "top";
  ctx.font = `${fontSize}px "${fontName}"`;
  ctx.fillStyle = "rgba(30, 100, 255, 0.9)";
  ctx.fillText(r.approvedText, startX, startY);

  if (isActive) {
    ctx.strokeStyle = "rgba(30, 100, 255, 0.3)";
    ctx.lineWidth = 1;
    ctx.strokeRect(a.line.x, a.line.y, a.line.w, a.line.h);
  }
}
```

**Step 2: Update renderCanvas to use the new status**

In `renderCanvas()` (around line 105), change:

```javascript
if (r.status === "solved" && r.solution) {
  drawRedactionSolution(r, isActive);
```

to:

```javascript
if (r.status === "approved" && r.approvedText) {
  drawRedactionApproved(r, isActive);
```

**Step 3: Commit**

```bash
git add unredact/static/canvas.js
git commit -m "feat: replace drawRedactionSolution with drawRedactionApproved (blue merged text)"
```

---

### Task 3: Update main.js references from "solved" to "approved"

**Files:**
- Modify: `unredact/static/main.js`

**Step 1: Update `statusLabel()` (around line 289)**

Change:
```javascript
case "solved": return "solved";
```
to:
```javascript
case "approved": return "approved";
```

**Step 2: Update `redactionInfoText()` (around line 300)**

Change:
```javascript
if (r.status === "solved" && r.solution) {
  return r.solution.text;
}
if ((r.status === "analyzed" || r.status === "solved") && r.analysis) {
```

to:
```javascript
if (r.status === "approved" && r.approvedText) {
  return r.approvedText.length > 30
    ? r.approvedText.slice(0, 30) + "..."
    : r.approvedText;
}
if (r.status === "analyzed" && r.analysis) {
```

**Step 3: Update `activateRedaction()` (around line 330)**

Change:
```javascript
if (r.status === "analyzed" || r.status === "solved") {
  openPopover(id);
}
```

to:
```javascript
if (r.status === "analyzed" || r.status === "approved") {
  openPopover(id);
}
```

**Step 4: Update `exportAnnotations()` (around line 557)**

Add `r.approvedText` to the export entry. After the existing `if (r.solution)` block:

```javascript
if (r.approvedText) {
  entry.approvedText = r.approvedText;
}
```

**Step 5: Commit**

```bash
git add unredact/static/main.js
git commit -m "feat: update main.js references from solved to approved status"
```

---

### Task 4: Update popover to handle approved redactions

**Files:**
- Modify: `unredact/static/popover.js:31-58`

**Step 1: Update `openPopover()` to pre-populate approved state**

When opening an approved redaction, the previous solution should show as a preview in the redaction marker, and the Accept button should be visible. Update the `openPopover` function:

After the existing `solveAccept.hidden = !!(r.preview === null);` line (around line 42), add handling for approved redactions:

```javascript
// For approved redactions, restore the solution as preview
if (r.status === "approved" && r.solution) {
  r.preview = r.solution.text;
  solveAccept.hidden = true;  // Already approved, no need to show accept
  redactionMarker.textContent = r.solution.text;
  redactionMarker.className = "redaction-marker preview";
} else {
  solveAccept.hidden = r.preview === null;
}
```

Replace the old `solveAccept.hidden = !!(r.preview === null);` line with the block above.

Also, when re-opening an approved redaction, the status should revert to "analyzed" so the canvas shows the analyze view (left text + gap + right text with preview) instead of the merged blue line. Add before the `popoverEl.hidden = false;` line:

```javascript
// Revert approved to analyzed for re-editing
if (r.status === "approved") {
  r.status = "analyzed";
}
```

**Step 2: Commit**

```bash
git add unredact/static/popover.js
git commit -m "feat: popover handles approved redactions with solution pre-populated"
```

---

### Task 5: Update CSS for approved status badge

**Files:**
- Modify: `unredact/static/style.css:230-233`

**Step 1: Replace `.status-solved` with `.status-approved`**

Change:
```css
.redaction-status.status-solved {
  color: var(--accent);
  background: rgba(0, 212, 116, 0.15);
}
```

to:
```css
.redaction-status.status-approved {
  color: var(--blue);
  background: rgba(66, 165, 245, 0.15);
}
```

**Step 2: Commit**

```bash
git add unredact/static/style.css
git commit -m "feat: blue CSS badge for approved status"
```

---

### Task 6: Verify and final commit

**Step 1: Manual verification checklist**

Open the app and test the following flow:
1. Upload a PDF, wait for OCR
2. Double-click to create a redaction
3. Open the popover, run the solver
4. Click a result to preview it (yellow text in gap)
5. Click Accept -> should see full blue merged text on canvas, status "approved" in list
6. Click the approved redaction -> should reopen popover with solution as preview
7. Re-run solver, accept a different result -> blue text updates
8. Navigate to another page and back -> blue text still visible
9. Ctrl+E export -> JSON includes `approvedText`

**Step 2: Run existing tests**

```bash
python -m pytest tests/ -v --ignore=tests/test_full_name_stress.py
```

Expected: all tests pass (this is a frontend-only change, backend tests unaffected).

**Step 3: Final commit if any fixups needed**

```bash
git add -A
git commit -m "fix: approval feature cleanup"
```
