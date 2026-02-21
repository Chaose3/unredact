# Mobile-Friendly Frontend Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Make the webapp usable on mobile devices (≤768px) with tab-based navigation, full-screen solver modal, and single-finger touch pan.

**Architecture:** CSS media queries restyle the existing desktop layout. A small JS module handles tab switching. No new pages, no server detection, no backend changes.

**Tech Stack:** Vanilla CSS media queries, vanilla JS DOM manipulation.

**Design doc:** `docs/plans/2026-02-21-mobile-friendly-design.md`

---

### Task 1: Add tab bar HTML

**Files:**
- Modify: `unredact/static/index.html:35` (inside `#workspace`, before `#left-panel`)

**Step 1: Add the tab bar markup**

Insert after `<div id="workspace">` (line 35) and before `<div id="left-panel">` (line 36):

```html
        <div id="mobile-tabs" class="mobile-tabs">
          <button class="mobile-tab active" data-tab="document">Document</button>
          <button class="mobile-tab" data-tab="redactions">Redactions</button>
        </div>
```

**Step 2: Add DOM references**

In `unredact/static/dom.js`, add at the end of the export block (before the `showToast` function, around line 57):

```javascript
export const mobileTabs = document.getElementById("mobile-tabs");
export const leftPanel = document.getElementById("left-panel");
```

**Step 3: Verify manually**

Open the app in a browser. The tab bar should be invisible on desktop (we'll hide it with CSS in Task 3). On narrow viewports it won't be styled yet — that's expected.

**Step 4: Commit**

```bash
git add unredact/static/index.html unredact/static/dom.js
git commit -m "feat: add mobile tab bar markup"
```

---

### Task 2: Add tab switching JS

**Files:**
- Modify: `unredact/static/main.js` (add tab init + auto-switch on redaction select)

**Step 1: Add tab switching logic**

At the top of `main.js`, add to the imports from `dom.js` (line 3): `mobileTabs`, `leftPanel`, `rightPanel`.

Note: `rightPanel` is already imported via `viewport.js`. Check current imports in `main.js` — if `rightPanel` and `leftPanel` aren't imported there, add them.

Then add a new function after the `deleteRedaction` function (~line 358):

```javascript
// ── Mobile tab switching ──

function switchTab(tab) {
  if (!mobileTabs) return;
  mobileTabs.querySelectorAll(".mobile-tab").forEach(btn => {
    btn.classList.toggle("active", btn.dataset.tab === tab);
  });
  leftPanel.classList.toggle("mobile-active", tab === "redactions");
  rightPanel.classList.toggle("mobile-active", tab === "document");
}

function initMobileTabs() {
  if (!mobileTabs) return;
  mobileTabs.addEventListener("click", (e) => {
    const btn = /** @type {HTMLElement} */ (e.target).closest(".mobile-tab");
    if (btn?.dataset.tab) switchTab(btn.dataset.tab);
  });
  // Default: Document tab active
  switchTab("document");
}
```

**Step 2: Call initMobileTabs from init**

Find the main init block (look for where `initViewport()`, `initPopover()`, `initSolver()` are called). Add `initMobileTabs();` alongside them.

**Step 3: Auto-switch to Document tab on redaction select**

In the `activateRedaction` function (~line 329), add at the end (before the closing `}`):

```javascript
  switchTab("document");
```

This ensures that tapping a redaction in the list on mobile switches to the Document view.

**Step 4: Verify manually**

At this point the JS is wired up but has no visual effect (CSS not added yet). No errors in console.

**Step 5: Commit**

```bash
git add unredact/static/main.js
git commit -m "feat: add mobile tab switching logic"
```

---

### Task 3: Add responsive CSS

**Files:**
- Modify: `unredact/static/style.css` (append media query block at end of file)

**Step 1: Add the full media query block**

Append to the end of `style.css`:

```css
/* ── Mobile responsive (≤768px) ── */

.mobile-tabs {
  display: none;
}

@media (max-width: 768px) {
  /* Header: compact */
  header {
    padding: 0.5rem 1rem;
  }
  header h1 {
    font-size: 1.1rem;
  }
  .subtitle {
    font-size: 0.7rem;
  }

  /* Main: tighter padding */
  main {
    padding: 0.5rem;
  }

  /* Controls bar: wrap on narrow screens */
  #controls {
    flex-wrap: wrap;
    gap: 0.5rem;
    padding: 0.4rem 0.6rem;
  }

  /* Tab bar: visible */
  .mobile-tabs {
    display: flex;
    gap: 0;
    flex-shrink: 0;
    border-bottom: 1px solid var(--border);
    background: var(--bg-surface);
    border-radius: 8px 8px 0 0;
  }
  .mobile-tab {
    flex: 1;
    padding: 0.6rem 0;
    background: none;
    border: none;
    color: var(--text-dim);
    font-size: 0.85rem;
    font-weight: 600;
    cursor: pointer;
    border-bottom: 2px solid transparent;
    transition: color 0.15s, border-color 0.15s;
  }
  .mobile-tab.active {
    color: var(--accent);
    border-bottom-color: var(--accent);
  }

  /* Workspace: stack vertically */
  #workspace {
    flex-direction: column;
  }

  /* Left panel: hidden by default, full-width when active */
  #left-panel {
    display: none;
    width: 100%;
    border-radius: 0 0 8px 8px;
  }
  #left-panel.mobile-active {
    display: block;
  }

  /* Right panel: hidden by default, full-size when active */
  #right-panel {
    display: none;
    border-radius: 0 0 8px 8px;
  }
  #right-panel.mobile-active {
    display: block;
    flex: 1;
  }

  /* Popover: full-screen modal */
  #popover {
    position: fixed;
    inset: 0;
    width: 100%;
    border-radius: 0;
    z-index: 50;
    background: rgba(20, 20, 40, 0.98);
    backdrop-filter: none;
  }

  /* Font toolbar: full-width, no right offset */
  #font-toolbar {
    right: 0.5rem;
    font-size: 0.7rem;
    gap: 0.4rem;
  }

  /* Touch targets: larger buttons */
  .tb-btn {
    width: 2.2rem;
    height: 2.2rem;
    font-size: 0.85rem;
  }

  /* Solve controls: slightly larger inputs for touch */
  .solve-controls select,
  .solve-controls input {
    padding: 4px 6px;
    font-size: 0.85rem;
  }
  .solve-btn {
    padding: 8px 20px;
  }

  /* Popover close button: larger touch target */
  .popover-close-btn {
    width: 2.2rem;
    height: 2.2rem;
    font-size: 0.9rem;
  }
}
```

**Step 2: Test at different widths**

Open Chrome DevTools, toggle device toolbar. Check:
- At 768px and below: tabs visible, panels toggle, popover is full-screen
- At 769px and above: no visual change from current desktop layout

**Step 3: Commit**

```bash
git add unredact/static/style.css
git commit -m "feat: add responsive CSS for mobile layout"
```

---

### Task 4: Add single-finger touch pan

**Files:**
- Modify: `unredact/static/viewport.js:157-196` (touch handling section)

**Step 1: Add single-finger pan alongside existing two-finger pinch**

Replace the entire touch handling section (lines 157-196) with:

```javascript
  // Touch: single-finger pan + two-finger pinch-zoom
  let lastTouches = null;
  let singleTouch = null;

  rightPanel.addEventListener("touchstart", (e) => {
    if (isPopoverArea(e)) return;
    if (e.touches.length === 2) {
      e.preventDefault();
      singleTouch = null;
      lastTouches = Array.from(e.touches);
    } else if (e.touches.length === 1) {
      singleTouch = {
        startX: e.touches[0].clientX,
        startY: e.touches[0].clientY,
        startPanX: state.panX,
        startPanY: state.panY,
        moved: false,
      };
    }
  }, { passive: false });

  rightPanel.addEventListener("touchmove", (e) => {
    if (isPopoverArea(e)) return;
    if (e.touches.length === 2 && lastTouches) {
      e.preventDefault();
      singleTouch = null;
      const [t0, t1] = e.touches;
      const [p0, p1] = lastTouches;

      const oldDist = Math.hypot(p1.clientX - p0.clientX, p1.clientY - p0.clientY);
      const newDist = Math.hypot(t1.clientX - t0.clientX, t1.clientY - t0.clientY);
      const zoomDelta = newDist / oldDist;

      const oldMidX = (p0.clientX + p1.clientX) / 2;
      const oldMidY = (p0.clientY + p1.clientY) / 2;
      const newMidX = (t0.clientX + t1.clientX) / 2;
      const newMidY = (t0.clientY + t1.clientY) / 2;

      state.panX -= (newMidX - oldMidX) / state.zoom;
      state.panY -= (newMidY - oldMidY) / state.zoom;

      const rect = rightPanel.getBoundingClientRect();
      const sx = newMidX - rect.left;
      const sy = newMidY - rect.top;
      zoomTo(state.zoom * zoomDelta, sx, sy, false);

      lastTouches = Array.from(e.touches);
    } else if (e.touches.length === 1 && singleTouch) {
      const dx = e.touches[0].clientX - singleTouch.startX;
      const dy = e.touches[0].clientY - singleTouch.startY;
      if (Math.abs(dx) > 2 || Math.abs(dy) > 2) singleTouch.moved = true;
      if (singleTouch.moved) {
        e.preventDefault();
        state.panX = singleTouch.startPanX - dx / state.zoom;
        state.panY = singleTouch.startPanY - dy / state.zoom;
        applyTransform(false);
      }
    }
  }, { passive: false });

  rightPanel.addEventListener("touchend", (e) => {
    if (e.touches.length === 0) {
      lastTouches = null;
      singleTouch = null;
    } else if (e.touches.length === 1) {
      lastTouches = null;
    }
  });
```

Key behaviors:
- Single finger drag: pans the document (same as mouse drag)
- Two finger pinch: zooms (existing behavior, preserved)
- Short taps (no movement): fall through to click handlers for redaction selection
- `isPopoverArea` check prevents pan when touching popover/toolbar controls

**Step 2: Test on mobile or with Chrome DevTools touch simulation**

- Single finger drag should pan the document
- Pinch zoom should still work
- Tapping a redaction should still select it (not pan)

**Step 3: Commit**

```bash
git add unredact/static/viewport.js
git commit -m "feat: add single-finger touch pan for mobile"
```

---

### Task 5: Final integration test

**Step 1: Test full mobile workflow**

Using Chrome DevTools device toolbar (e.g. iPhone 14 or Pixel 7 preset):

1. Upload a PDF — drop zone should be usable
2. Tab bar visible: "Document" and "Redactions"
3. Click "Redactions" tab — left panel shows, right panel hides
4. Click "Detect Redactions" — redactions appear in list
5. Click a redaction — auto-switches to Document tab, pans to redaction
6. Popover opens as full-screen modal
7. Run solver — results list scrollable
8. Click a result — preview renders on document
9. Close popover with X
10. Single-finger pan works
11. Pinch-zoom works
12. Font toolbar wraps properly, buttons are tap-friendly

**Step 2: Test desktop is unchanged**

Resize browser to >768px. Verify:
- No tab bar visible
- Split panel layout intact
- Popover as overlay in right panel (not full-screen)
- All existing functionality works

**Step 3: Final commit if any fixes needed**

```bash
git add -A
git commit -m "fix: mobile layout adjustments"
```
