# Mobile-Responsive Redesign Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace the broken tab-based mobile layout with a draggable bottom sheet that keeps the document viewer always visible, while preserving the desktop sidebar+overlay layout.

**Architecture:** A single `#bottom-sheet` element serves as the left sidebar on desktop (>768px) and a fixed-position bottom overlay on mobile (<=768px). On mobile, JS moves the popover, font-toolbar, and text-edit-bar DOM elements into the sheet's tab panes. The document viewer (`#right-panel`) is never hidden, keeping canvas coordinates valid. Sheet snap state (peek/half/full) is managed via CSS custom properties and touch events on the drag handle.

**Tech Stack:** Vanilla CSS (media queries, custom properties, transitions), vanilla JS (DOM manipulation, touch events, matchMedia).

**Design doc:** `docs/plans/2026-02-21-mobile-responsive-design.md`

---

### Task 1: Restructure HTML and update DOM refs

**Files:**
- Modify: `unredact/static/index.html`
- Modify: `unredact/static/dom.js`

**Step 1: Replace the mobile-tabs and left-panel with the bottom-sheet container**

In `unredact/static/index.html`, replace the `#mobile-tabs` div (lines 36-39) and `#left-panel` div (lines 40-44) with:

```html
        <div id="bottom-sheet">
          <div id="sheet-handle"><div class="handle-bar"></div></div>
          <div id="sheet-tabs">
            <button class="sheet-tab active" data-tab="solve">Solve</button>
            <button class="sheet-tab" data-tab="edit">Edit</button>
            <button class="sheet-tab" data-tab="list">List</button>
          </div>
          <div id="sheet-content">
            <div id="tab-solve" class="tab-pane active"></div>
            <div id="tab-edit" class="tab-pane"></div>
            <div id="tab-list" class="tab-pane">
              <button id="detect-btn" disabled>Detect Redactions</button>
              <div id="ocr-status"></div>
              <div id="redaction-list"></div>
            </div>
          </div>
        </div>
```

The full `#workspace` block should now look like:

```html
      <div id="workspace">
        <div id="bottom-sheet">
          <div id="sheet-handle"><div class="handle-bar"></div></div>
          <div id="sheet-tabs">
            <button class="sheet-tab active" data-tab="solve">Solve</button>
            <button class="sheet-tab" data-tab="edit">Edit</button>
            <button class="sheet-tab" data-tab="list">List</button>
          </div>
          <div id="sheet-content">
            <div id="tab-solve" class="tab-pane active"></div>
            <div id="tab-edit" class="tab-pane"></div>
            <div id="tab-list" class="tab-pane">
              <button id="detect-btn" disabled>Detect Redactions</button>
              <div id="ocr-status"></div>
              <div id="redaction-list"></div>
            </div>
          </div>
        </div>

        <div id="right-panel">
          <div id="font-toolbar" hidden>
            <!-- ... unchanged ... -->
          </div>
          <div id="popover" hidden>
            <!-- ... unchanged ... -->
          </div>
          <div id="doc-container">
            <img id="doc-image" alt="Document page">
            <canvas id="overlay-canvas"></canvas>
          </div>
          <div id="text-edit-bar" hidden>
            <!-- ... unchanged ... -->
          </div>
        </div>
      </div>
```

Note: `#popover`, `#font-toolbar`, `#text-edit-bar` remain inside `#right-panel`. They stay there on desktop. On mobile, JS will move them into the sheet's tab panes (Task 5).

**Step 2: Update dom.js**

In `unredact/static/dom.js`, replace the `mobileTabs` and `leftPanel` exports (lines 57-58) with:

```javascript
export const bottomSheet = document.getElementById("bottom-sheet");
export const sheetHandle = document.getElementById("sheet-handle");
export const sheetTabs = document.getElementById("sheet-tabs");
export const tabSolve = document.getElementById("tab-solve");
export const tabEdit = document.getElementById("tab-edit");
export const tabList = document.getElementById("tab-list");
```

**Step 3: Update main.js imports**

In `unredact/static/main.js`, replace `leftPanel, mobileTabs` in the import block (line 8) with `bottomSheet, sheetTabs, tabSolve, tabEdit, tabList`:

```javascript
import {
  dropZone, fileInput, uploadSection, viewerSection, docImage,
  canvas, pageInfo, prevBtn, nextBtn, redactionListEl, detectBtn,
  rightPanel, bottomSheet, sheetTabs, tabSolve, tabEdit, tabList, fontSelect,
  solveAccept, gapValue, showToast,
} from './dom.js';
```

**Step 4: Delete the old switchTab and initMobileTabs functions**

In `main.js`, delete the entire mobile tab switching section (lines 359-378):

```javascript
// DELETE: lines 359-378 (switchTab function, initMobileTabs function)
```

Also delete `switchTab("document");` from the `activateRedaction` function (line 346). We'll add the new sheet behavior in Task 6.

Also delete `initMobileTabs();` from the init block (line 639).

**Step 5: Verify**

Open the app in a desktop browser. The page should load without JS errors. The left sidebar should show the Detect button and redaction list (from `#tab-list`). The sheet handle and tabs should be visible but unstyled (we'll hide them on desktop in Task 2). The popover, font toolbar, and text edit bar should still work when a redaction is selected (they're still inside `#right-panel`).

**Step 6: Commit**

```bash
git add unredact/static/index.html unredact/static/dom.js unredact/static/main.js
git commit -m "refactor: replace left-panel with bottom-sheet container"
```

---

### Task 2: Desktop CSS — sheet as sidebar

**Files:**
- Modify: `unredact/static/style.css`

**Step 1: Replace `#left-panel` rules with `#bottom-sheet` rules**

Find the `#left-panel` rule block (around line 138-144):

```css
/* Left panel — scrollable redaction list */
#left-panel {
  width: 280px;
  flex-shrink: 0;
  background: var(--bg-surface);
  border-radius: 8px;
  overflow-y: auto;
}
```

Replace with:

```css
/* Bottom sheet — sidebar on desktop, bottom overlay on mobile */
#bottom-sheet {
  width: 280px;
  flex-shrink: 0;
  background: var(--bg-surface);
  border-radius: 8px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
}

/* Handle and tabs hidden on desktop */
#sheet-handle { display: none; }
#sheet-tabs { display: none; }

/* Tab panes: only list visible on desktop */
#sheet-content {
  flex: 1;
  overflow-y: auto;
}
.tab-pane { display: none; }
#tab-list { display: block; }
```

**Step 2: Add sheet handle and tab styling (will only show on mobile)**

Add these rules after the `#bottom-sheet` block (these define the base look, mobile CSS will make them visible):

```css
#sheet-handle {
  flex-shrink: 0;
  padding: 8px 0;
  cursor: grab;
  touch-action: none;
}
#sheet-handle:active { cursor: grabbing; }

.handle-bar {
  width: 36px;
  height: 4px;
  background: var(--text-muted);
  border-radius: 2px;
  margin: 0 auto;
}

#sheet-tabs {
  flex-shrink: 0;
  display: flex;
  gap: 0;
  border-bottom: 1px solid var(--border);
  padding: 0 8px;
}

.sheet-tab {
  flex: 1;
  padding: 6px 0;
  background: none;
  border: none;
  color: var(--text-dim);
  font-size: 0.8rem;
  font-weight: 600;
  cursor: pointer;
  border-bottom: 2px solid transparent;
  transition: color 0.15s, border-color 0.15s;
}
.sheet-tab.active {
  color: var(--accent);
  border-bottom-color: var(--accent);
}
```

**Step 3: Delete old mobile CSS**

Delete the entire mobile section at the bottom of style.css — the `.mobile-tabs` base rule (lines 777-779) AND the full `@media (max-width: 768px)` block (lines 781-902). We'll rewrite mobile CSS from scratch in Task 3.

**Step 4: Verify**

Desktop browser: the app should look identical to before. The sidebar shows the List tab content (detect button + redaction list). Sheet handle and tabs are hidden. Popover and toolbars still work as overlays in the right panel.

**Step 5: Commit**

```bash
git add unredact/static/style.css
git commit -m "feat: desktop CSS for bottom-sheet as sidebar"
```

---

### Task 3: Mobile CSS — sheet as bottom overlay

**Files:**
- Modify: `unredact/static/style.css`

**Step 1: Add the mobile media query block**

Append to the end of `style.css`:

```css
/* ── Mobile (≤768px) ── */

@media (max-width: 768px) {
  /* Header: compact */
  header { padding: 0.5rem 1rem; }
  header h1 { font-size: 1.1rem; }
  .subtitle { font-size: 0.7rem; }

  /* Main: tighter padding */
  main { padding: 0.5rem; }

  /* Controls bar: wrap on narrow screens */
  #controls {
    flex-wrap: wrap;
    gap: 0.5rem;
    padding: 0.4rem 0.6rem;
  }

  /* Workspace: full column layout */
  #workspace {
    flex-direction: column;
    position: relative;
  }

  /* Right panel: always visible, takes available space above sheet */
  #right-panel {
    flex: 1;
    min-height: 0;
    border-radius: 0;
  }

  /* Bottom sheet: fixed overlay at bottom */
  #bottom-sheet {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    width: 100%;
    height: var(--sheet-height, 60px);
    z-index: 40;
    border-radius: 12px 12px 0 0;
    background: var(--bg-surface);
    border-top: 1px solid var(--border);
    transition: height 0.25s ease-out;
    overflow: hidden;
  }

  /* Handle and tabs: visible on mobile */
  #sheet-handle { display: block; }
  #sheet-tabs { display: flex; }

  /* Sheet content: fill remaining space, scroll */
  #sheet-content {
    flex: 1;
    overflow-y: auto;
    padding: 0 8px 8px;
  }

  /* All tab panes hidden by default; .active shown */
  .tab-pane { display: none; }
  .tab-pane.active { display: block; }

  /* ── Elements moved into sheet tabs on mobile ── */

  /* Popover: static flow inside solve tab (no close button needed) */
  #popover:not([hidden]) {
    position: static;
    width: 100%;
    background: transparent;
    border: none;
    border-radius: 0;
    padding: 0;
    backdrop-filter: none;
    z-index: auto;
    overflow-y: visible;
  }
  .popover-header { display: none; }

  /* Font toolbar: static flow inside edit tab */
  #font-toolbar:not([hidden]) {
    position: static;
    width: 100%;
    right: auto;
    left: auto;
    top: auto;
    border-radius: 0;
    border: none;
    background: transparent;
    backdrop-filter: none;
    z-index: auto;
    flex-wrap: wrap;
    gap: 0.6rem;
    padding: 8px 0;
  }

  /* Text edit bar: static flow inside edit tab */
  #text-edit-bar:not([hidden]) {
    position: static;
    width: 100%;
    left: auto;
    right: auto;
    bottom: auto;
    border-radius: 0;
    border: none;
    border-top: 1px solid var(--border);
    background: transparent;
    backdrop-filter: none;
    z-index: auto;
    padding: 8px 0;
    margin-top: 8px;
  }

  /* Touch targets: larger buttons */
  .tb-btn {
    width: 2.2rem;
    height: 2.2rem;
    font-size: 0.85rem;
  }

  /* Solve controls: larger inputs for touch */
  .solve-controls select,
  .solve-controls input {
    padding: 4px 6px;
    font-size: 0.85rem;
  }
  .solve-btn { padding: 8px 20px; }

  /* Delete button: always visible on touch (no hover) */
  .redaction-delete { opacity: 1; }

  /* Viewer section needs bottom margin for sheet peek height */
  #viewer-section {
    padding-bottom: var(--sheet-height, 60px);
  }
}
```

**Step 2: Verify**

Open Chrome DevTools, toggle device toolbar to a phone preset (e.g. iPhone 14, 390x844):

- The right panel (document viewer) should fill the screen
- A thin bottom sheet should be visible at the bottom (60px, peek height) with a handle bar
- The sheet tabs should be visible: Solve | Edit | List
- At this point, tabs won't work yet (JS not wired) but the layout should be correct
- Desktop should still look exactly the same

**Step 3: Commit**

```bash
git add unredact/static/style.css
git commit -m "feat: mobile CSS for bottom sheet overlay"
```

---

### Task 4: Sheet snap behavior (JS)

**Files:**
- Modify: `unredact/static/main.js`

**Step 1: Add sheet snap constants and state**

At the top of `main.js`, after the imports, add:

```javascript
// ── Sheet snap management ──

const SNAP_PEEK = 60;
const SNAP_HALF_RATIO = 0.45;
const SNAP_FULL_RATIO = 0.9;

/** @type {'peek'|'half'|'full'} */
let sheetSnap = 'peek';

function getSnapHeight(snap) {
  const vh = window.innerHeight;
  switch (snap) {
    case 'peek': return SNAP_PEEK;
    case 'half': return Math.round(vh * SNAP_HALF_RATIO);
    case 'full': return Math.round(vh * SNAP_FULL_RATIO);
  }
}

function setSheetSnap(snap) {
  sheetSnap = snap;
  const h = getSnapHeight(snap);
  document.documentElement.style.setProperty('--sheet-height', h + 'px');
  bottomSheet.style.height = h + 'px';
}
```

**Step 2: Add sheet drag handling**

Below the snap functions, add:

```javascript
function initSheetDrag() {
  const handle = document.getElementById('sheet-handle');
  if (!handle) return;

  let dragState = null;

  handle.addEventListener('touchstart', (e) => {
    if (e.touches.length !== 1) return;
    dragState = {
      startY: e.touches[0].clientY,
      startHeight: bottomSheet.offsetHeight,
    };
    bottomSheet.style.transition = 'none';
  }, { passive: true });

  handle.addEventListener('touchmove', (e) => {
    if (!dragState) return;
    e.preventDefault();
    const dy = dragState.startY - e.touches[0].clientY;
    const newH = Math.max(SNAP_PEEK, Math.min(
      getSnapHeight('full'),
      dragState.startHeight + dy
    ));
    bottomSheet.style.height = newH + 'px';
    document.documentElement.style.setProperty('--sheet-height', newH + 'px');
  }, { passive: false });

  const endDrag = () => {
    if (!dragState) return;
    bottomSheet.style.transition = '';
    const currentH = bottomSheet.offsetHeight;

    // Find nearest snap point
    const peekH = getSnapHeight('peek');
    const halfH = getSnapHeight('half');
    const fullH = getSnapHeight('full');

    const peekDist = Math.abs(currentH - peekH);
    const halfDist = Math.abs(currentH - halfH);
    const fullDist = Math.abs(currentH - fullH);

    if (peekDist <= halfDist && peekDist <= fullDist) setSheetSnap('peek');
    else if (halfDist <= fullDist) setSheetSnap('half');
    else setSheetSnap('full');

    dragState = null;
  };

  handle.addEventListener('touchend', endDrag);
  handle.addEventListener('touchcancel', endDrag);
}
```

**Step 3: Initialize sheet on page load**

In the init block at the bottom of `main.js`, add (where `initMobileTabs()` used to be):

```javascript
// ── Initialize sheet (mobile only) ──
const isMobile = () => window.matchMedia('(max-width: 768px)').matches;

if (isMobile()) {
  setSheetSnap('peek');
  initSheetDrag();
}
```

**Step 4: Verify**

On mobile viewport: the sheet should be 60px tall (peek). Dragging the handle up should resize the sheet. Releasing should snap to the nearest snap point with a smooth transition. On desktop: no effect (sheet is a sidebar, handle is hidden).

**Step 5: Commit**

```bash
git add unredact/static/main.js
git commit -m "feat: sheet drag and snap behavior for mobile"
```

---

### Task 5: Tab switching and DOM element relocation

**Files:**
- Modify: `unredact/static/main.js`
- Modify: `unredact/static/dom.js`

**Step 1: Add DOM movement functions**

In `main.js`, add after the sheet snap code:

```javascript
// ── Mobile layout: move controls into sheet tabs ──

let elementsInSheet = false;

function moveElementsToSheet() {
  if (elementsInSheet) return;
  const popover = document.getElementById('popover');
  const fontToolbar = document.getElementById('font-toolbar');
  const textEditBar = document.getElementById('text-edit-bar');

  tabSolve.appendChild(popover);
  tabEdit.appendChild(fontToolbar);
  tabEdit.appendChild(textEditBar);
  elementsInSheet = true;
}

function moveElementsToPanel() {
  if (!elementsInSheet) return;
  const popover = document.getElementById('popover');
  const fontToolbar = document.getElementById('font-toolbar');
  const textEditBar = document.getElementById('text-edit-bar');

  rightPanel.insertBefore(fontToolbar, rightPanel.querySelector('#doc-container'));
  rightPanel.insertBefore(popover, rightPanel.querySelector('#doc-container'));
  rightPanel.appendChild(textEditBar);
  elementsInSheet = false;
}
```

**Step 2: Add tab switching**

```javascript
// ── Sheet tab switching ──

function switchSheetTab(tab) {
  sheetTabs.querySelectorAll('.sheet-tab').forEach(btn => {
    btn.classList.toggle('active', btn.dataset.tab === tab);
  });
  document.querySelectorAll('#sheet-content .tab-pane').forEach(pane => {
    pane.classList.toggle('active', pane.id === 'tab-' + tab);
  });
}

function initSheetTabs() {
  sheetTabs.addEventListener('click', (e) => {
    const btn = /** @type {HTMLElement} */ (e.target).closest('.sheet-tab');
    if (btn?.dataset.tab) switchSheetTab(btn.dataset.tab);
  });
}
```

**Step 3: Handle viewport changes**

```javascript
// ── Responsive layout management ──

function handleLayoutChange() {
  if (isMobile()) {
    moveElementsToSheet();
    setSheetSnap(sheetSnap);
  } else {
    moveElementsToPanel();
    document.documentElement.style.removeProperty('--sheet-height');
    bottomSheet.style.height = '';
  }
}

window.matchMedia('(max-width: 768px)').addEventListener('change', handleLayoutChange);
```

**Step 4: Wire up initialization**

Update the init block at the bottom of `main.js`. Replace the existing mobile init code with:

```javascript
// ── Initialize sheet and tabs ──
initSheetTabs();
initSheetDrag();
handleLayoutChange();
```

**Step 5: Verify**

Mobile viewport:
- Sheet tabs should be visible: Solve | Edit | List
- Clicking List tab should show the detect button and redaction list
- Clicking Solve or Edit should show empty panes (controls not moved yet until popover opens)
- Resizing from mobile to desktop should move elements back to their desktop positions

Desktop:
- Sidebar shows redaction list as before
- Popover, font toolbar, text edit bar work as overlays in right panel

**Step 6: Commit**

```bash
git add unredact/static/main.js
git commit -m "feat: sheet tab switching and responsive DOM relocation"
```

---

### Task 6: Integrate popover/main flow with sheet

**Files:**
- Modify: `unredact/static/popover.js`
- Modify: `unredact/static/main.js`

**Step 1: Export isMobile and sheet control from main.js**

We need popover.js to be able to control the sheet. The cleanest way is to have main.js export sheet control functions. But since popover.js can't import from main.js (it would create a circular dependency), we'll use the existing callback pattern.

In `main.js`, add a function that popover.js can call via a registered callback:

```javascript
/** @type {((tab: string, snap: string) => void)|null} */
let _onSheetChange = null;
export function setOnSheetChange(fn) { _onSheetChange = fn; }
```

Wait — main.js doesn't export anything currently. Instead, let's keep it simpler: have main.js provide sheet helpers to popover.js via the existing pattern.

Actually, the cleanest approach: move `switchSheetTab`, `setSheetSnap`, and `isMobile` into a shared location. But to minimize changes, let's just have main.js handle all sheet interactions, and popover.js just calls `openPopover`/`closePopover` as before. Main.js reacts to those calls.

**Revised approach — modify main.js's activateRedaction and add a popover close handler:**

In `main.js`, update `activateRedaction`:

```javascript
function activateRedaction(id) {
  const r = state.redactions[id];
  if (!r) return;

  state.activeRedaction = id;

  state.panX = r.x + r.w / 2;
  state.panY = r.y + r.h / 2;
  applyTransform(true);

  renderRedactionList();
  renderCanvas();

  if (r.status === "analyzed" || r.status === "approved") {
    openPopover(id);
    if (isMobile()) {
      switchSheetTab('solve');
      setSheetSnap('half');
    }
  }
}
```

**Step 2: Update the popover close handler in main.js**

Find `setOnPopoverClose(stopSolve)` in the init block and change it to:

```javascript
setOnPopoverClose(() => {
  stopSolve();
  if (isMobile()) {
    switchSheetTab('list');
    setSheetSnap('peek');
  }
});
```

**Step 3: Update popover.js closePopover**

In `popover.js`, update `closePopover` to also deselect the active redaction on mobile:

```javascript
export function closePopover() {
  popoverEl.hidden = true;
  fontToolbar.hidden = true;
  textEditBar.hidden = true;
  if (_onClose) _onClose();
}
```

This stays the same — the `_onClose` callback in main.js now handles the sheet snap.

**Step 4: Handle popover open on mobile**

The `openPopover` function in `popover.js` sets `popoverEl.hidden = false`, `fontToolbar.hidden = false`, `textEditBar.hidden = false`. On mobile, these elements are inside the sheet's tab panes. Setting `hidden = false` removes the `[hidden]` attribute, which lets them display (since the `[hidden] { display: none !important; }` rule hides them when hidden).

This should work as-is! When the Solve tab is active and `popoverEl.hidden = false`, the popover shows inside the tab. When the Edit tab is active, `fontToolbar.hidden = false` and `textEditBar.hidden = false` make those elements visible.

**Step 5: Verify**

Mobile viewport:
1. Upload a PDF
2. Click "Detect Redactions"
3. Once redactions appear in the List tab, tap one
4. Sheet should snap to half, Solve tab should be active, solver controls visible
5. Tapping Edit tab should show font toolbar and text edit bar
6. Closing popover (via close button on desktop, or just deselecting) should snap sheet to peek

Desktop: all behavior unchanged.

**Step 6: Commit**

```bash
git add unredact/static/main.js unredact/static/popover.js
git commit -m "feat: wire sheet snap and tab switching to popover flow"
```

---

### Task 7: Polish, touch targets, and cleanup

**Files:**
- Modify: `unredact/static/style.css`
- Modify: `unredact/static/main.js`

**Step 1: Add tap-on-canvas to deselect on mobile**

In `main.js`, in the canvas `mousedown` listener (the first one, around line 382), add deselection when tapping empty area on mobile:

After the existing `if (hit)` block, add an `else`:

```javascript
  if (hit) {
    e.stopPropagation();
    activateRedaction(hit.id);
  } else if (isMobile() && state.activeRedaction) {
    state.activeRedaction = null;
    closePopover();
    renderRedactionList();
    renderCanvas();
  }
```

**Step 2: Add redaction list tap behavior for mobile**

In `main.js`, in the `renderRedactionList` function, the `div.addEventListener("click", () => activateRedaction(r.id))` already exists. On mobile, tapping a redaction in the List tab should switch to Solve tab and snap to half. This is already handled in `activateRedaction` from Task 6. No changes needed.

**Step 3: Verify the full mobile workflow**

Using Chrome DevTools device toolbar (e.g. iPhone 14):

1. Load the page — upload drop zone visible, no sheet yet (viewer section hidden)
2. Upload a PDF — viewer section appears, sheet visible at peek (60px)
3. List tab shows "Detect Redactions" button
4. Click "Detect Redactions" — redactions appear in list
5. Tap a redaction in list — sheet snaps to half, Solve tab active with controls
6. Run solver — results appear, scrollable in sheet
7. Tap a result — preview shows on document above
8. Tap Edit tab — font controls and text inputs visible
9. Adjust font/position — changes reflected in document preview above
10. Tap Accept — solution saved
11. Tap empty canvas area — sheet snaps to peek, deselects redaction
12. Drag handle up — sheet expands to full
13. Drag handle down — sheet snaps to peek
14. Pinch-zoom on document — works (document viewer never hidden)
15. Single-finger pan — works

**Step 4: Test desktop is unchanged**

Resize to >768px:
- Left sidebar with redaction list
- Popover overlay in right panel
- Font toolbar at top of right panel
- Text edit bar at bottom
- All zoom/pan/interaction unchanged

**Step 5: Commit**

```bash
git add unredact/static/style.css unredact/static/main.js
git commit -m "fix: mobile polish — tap-to-deselect and workflow fixes"
```
