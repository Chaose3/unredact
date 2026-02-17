# Frontend Modularization Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Split the monolithic 1,436-line `app.js` into 9 ES modules with JSDoc types for better maintainability and agent visibility.

**Architecture:** ES modules with `<script type="module">`, JSDoc `@typedef` for type safety via `@ts-check`, callback pattern to break circular dependencies. No build step — browsers load modules natively via FastAPI's existing static file serving.

**Tech Stack:** Vanilla JavaScript ES modules, JSDoc + `@ts-check`, `jsconfig.json` for VS Code

**Design doc:** `docs/plans/2026-02-17-frontend-modularization-design.md`

**Key design decisions:**
- Circular dependency between popover↔solver broken via `setOnPopoverClose(callback)` pattern
- `acceptSolution()` split: solver.js does state mutation, main.js handles UI updates (renderRedactionList)
- Modules with event listeners use `init*()` functions called by main.js (explicit side effects)
- DOM elements typed with JSDoc casts only where subtype APIs are needed (`.value`, `.src`, etc.)

---

### Task 1: Create types.js

**Files:** Create `unredact/static/types.js`

**Step 1: Write types.js**

```js
// @ts-check
/** JSDoc type definitions — the data contract layer for all modules. */

/**
 * @typedef {Object} Font
 * @property {string} id
 * @property {string} name
 * @property {boolean} available
 */

/**
 * @typedef {Object} Segment
 * @property {string} text
 */

/**
 * @typedef {Object} AnalysisFont
 * @property {string} id
 * @property {string} name
 * @property {number} size
 */

/**
 * @typedef {Object} AnalysisGap
 * @property {number} x
 * @property {number} y
 * @property {number} w
 * @property {number} h
 */

/**
 * @typedef {Object} AnalysisLine
 * @property {number} x
 * @property {number} y
 * @property {number} w
 * @property {number} h
 */

/**
 * @typedef {Object} Analysis
 * @property {AnalysisFont} font
 * @property {number} [offset_x]
 * @property {number} [offset_y]
 * @property {AnalysisGap} gap
 * @property {AnalysisLine} line
 * @property {Segment[]} segments
 */

/**
 * @typedef {Object} Overrides
 * @property {string} fontId
 * @property {number} fontSize
 * @property {number} offsetX
 * @property {number} offsetY
 * @property {number} gapWidth
 * @property {string} leftText
 * @property {string} rightText
 */

/**
 * @typedef {Object} Solution
 * @property {string} text
 * @property {string} fontName
 * @property {number} fontSize
 */

/**
 * @typedef {Object} Redaction
 * @property {string} id
 * @property {number} x
 * @property {number} y
 * @property {number} w
 * @property {number} h
 * @property {number} page
 * @property {'unanalyzed'|'analyzing'|'analyzed'|'solved'|'error'} status
 * @property {Analysis|null} analysis
 * @property {Solution|null} solution
 * @property {string|null} preview
 * @property {Overrides} [overrides]
 */

/**
 * @typedef {Object} AssocEntry
 * @property {string} person_id
 * @property {string} match_type
 * @property {number} tier
 */

/**
 * @typedef {Object} Person
 * @property {string} name
 * @property {string} category
 */

/**
 * @typedef {Object} AssociatesData
 * @property {Object<string, AssocEntry[]>} names
 * @property {Object<string, Person>} persons
 * @property {string[]} [victim_names]
 * @property {Set<string>} [victim_set]
 */

/**
 * @typedef {Object} AppState
 * @property {string|null} docId
 * @property {number} pageCount
 * @property {number} currentPage
 * @property {Object<string, Redaction>} redactions
 * @property {string|null} activeRedaction
 * @property {Font[]} fonts
 * @property {boolean} fontsReady
 * @property {number} zoom
 * @property {number} panX
 * @property {number} panY
 * @property {AssociatesData|null} associates
 */

export {};
```

---

### Task 2: Create state.js

**Files:** Create `unredact/static/state.js`

**Step 1: Write state.js**

```js
// @ts-check
/** Application state — single source of truth for all UI and document data. */

/** @type {import('./types.js').AppState} */
export const state = {
  docId: null,
  pageCount: 0,
  currentPage: 1,
  redactions: {},
  activeRedaction: null,
  fonts: [],
  fontsReady: false,
  zoom: 1,
  panX: 0,
  panY: 0,
  associates: null,
};

/**
 * Get redactions for the current page, sorted top-to-bottom then left-to-right.
 * @returns {import('./types.js').Redaction[]}
 */
export function getPageRedactions() {
  return Object.values(state.redactions)
    .filter((r) => r.page === state.currentPage)
    .sort((a, b) => {
      if (Math.abs(a.y - b.y) > 5) return a.y - b.y;
      return a.x - b.x;
    });
}
```

---

### Task 3: Create dom.js

**Files:** Create `unredact/static/dom.js`

**Step 1: Write dom.js**

```js
// @ts-check
/** DOM element references and utility functions. */

export const dropZone = document.getElementById("drop-zone");
export const fileInput = /** @type {HTMLInputElement} */ (document.getElementById("file-input"));
export const uploadSection = document.getElementById("upload-section");
export const viewerSection = document.getElementById("viewer-section");
export const docImage = /** @type {HTMLImageElement} */ (document.getElementById("doc-image"));
export const canvas = /** @type {HTMLCanvasElement} */ (document.getElementById("overlay-canvas"));
export const ctx = /** @type {CanvasRenderingContext2D} */ (canvas.getContext("2d"));
export const pageInfo = document.getElementById("page-info");
export const prevBtn = /** @type {HTMLButtonElement} */ (document.getElementById("prev-page"));
export const nextBtn = /** @type {HTMLButtonElement} */ (document.getElementById("next-page"));
export const redactionListEl = document.getElementById("redaction-list");
export const zoomInBtn = document.getElementById("zoom-in");
export const zoomOutBtn = document.getElementById("zoom-out");
export const zoomFitBtn = document.getElementById("zoom-fit");
export const zoomLevel = document.getElementById("zoom-level");
export const rightPanel = document.getElementById("right-panel");
export const docContainer = document.getElementById("doc-container");
export const popoverEl = document.getElementById("popover");
export const popoverClose = document.getElementById("popover-close");
export const solveCharset = /** @type {HTMLSelectElement} */ (document.getElementById("solve-charset"));
export const solveTolerance = /** @type {HTMLInputElement} */ (document.getElementById("solve-tolerance"));
export const solveTolValue = document.getElementById("solve-tol-value");
export const solveMode = /** @type {HTMLSelectElement} */ (document.getElementById("solve-mode"));
export const solveFilter = /** @type {HTMLSelectElement} */ (document.getElementById("solve-filter"));
export const solveFilterPrefix = /** @type {HTMLInputElement} */ (document.getElementById("solve-filter-prefix"));
export const solveFilterSuffix = /** @type {HTMLInputElement} */ (document.getElementById("solve-filter-suffix"));
export const solveStart = document.getElementById("solve-start");
export const solveStop = document.getElementById("solve-stop");
export const solveAccept = document.getElementById("solve-accept");
export const solveStatus = document.getElementById("solve-status");
export const solveResults = document.getElementById("solve-results");
export const fontToolbar = document.getElementById("font-toolbar");
export const fontSelect = /** @type {HTMLSelectElement} */ (document.getElementById("font-select"));
export const sizeSlider = /** @type {HTMLInputElement} */ (document.getElementById("size-slider"));
export const sizeValue = document.getElementById("size-value");
export const sizeDown = document.getElementById("size-down");
export const sizeUp = document.getElementById("size-up");
export const posUp = document.getElementById("pos-up");
export const posDown = document.getElementById("pos-down");
export const posLeft = document.getElementById("pos-left");
export const posRight = document.getElementById("pos-right");
export const posReset = document.getElementById("pos-reset");
export const posDisplay = document.getElementById("pos-display");
export const gapDown = document.getElementById("gap-down");
export const gapUp = document.getElementById("gap-up");
export const gapValue = document.getElementById("gap-value");
export const textEditBar = document.getElementById("text-edit-bar");
export const leftTextInput = /** @type {HTMLInputElement} */ (document.getElementById("left-text-input"));
export const rightTextInput = /** @type {HTMLInputElement} */ (document.getElementById("right-text-input"));
export const redactionMarker = document.getElementById("redaction-marker");
export const textReset = document.getElementById("text-reset");
export const toastContainer = document.getElementById("toast-container");

/**
 * Show a toast notification.
 * @param {string} message
 * @param {'info'|'error'} [type]
 */
export function showToast(message, type = "info") {
  const el = document.createElement("div");
  el.className = `toast ${type}`;
  el.textContent = message;
  toastContainer.appendChild(el);
  setTimeout(() => el.remove(), 3000);
}

/**
 * Escape HTML special characters.
 * @param {string} text
 * @returns {string}
 */
export function escapeHtml(text) {
  const div = document.createElement("div");
  div.textContent = text;
  return div.innerHTML;
}
```

---

### Task 4: Create canvas.js

**Files:** Create `unredact/static/canvas.js`

**Step 1: Write canvas.js**

```js
// @ts-check
/** Canvas rendering — draws redaction overlays on the document image. */

import { state, getPageRedactions } from './state.js';
import { canvas, ctx, docImage } from './dom.js';

export function clearCanvas() {
  canvas.width = 0;
  canvas.height = 0;
}

export function renderCanvas() {
  if (!docImage.naturalWidth || !state.fontsReady) return;

  const redactions = getPageRedactions();

  canvas.width = docImage.naturalWidth;
  canvas.height = docImage.naturalHeight;
  canvas.style.width = docImage.naturalWidth + "px";
  canvas.style.height = docImage.naturalHeight + "px";

  ctx.clearRect(0, 0, canvas.width, canvas.height);

  for (const r of redactions) {
    const isActive = r.id === state.activeRedaction;

    if (r.status === "solved" && r.solution) {
      drawRedactionSolution(r, isActive);
    } else if (r.preview) {
      drawRedactionPreview(r, isActive);
    } else if (r.status === "analyzed" && r.analysis) {
      drawRedactionAnalyzed(r, isActive);
    } else {
      drawRedactionUnanalyzed(r, isActive);
    }
  }
}

/**
 * @param {import('./types.js').Redaction} r
 * @param {boolean} isActive
 */
function drawRedactionUnanalyzed(r, isActive) {
  const alpha = isActive ? 0.4 : 0.25;
  const borderAlpha = isActive ? 0.9 : 0.5;

  ctx.fillStyle = `rgba(66, 133, 244, ${alpha})`;
  ctx.fillRect(r.x, r.y, r.w, r.h);

  ctx.strokeStyle = `rgba(66, 133, 244, ${borderAlpha})`;
  ctx.lineWidth = isActive ? 2.5 : 1.5;
  ctx.strokeRect(r.x, r.y, r.w, r.h);
}

/**
 * @param {import('./types.js').Redaction} r
 * @param {boolean} isActive
 */
function drawRedactionAnalyzed(r, isActive) {
  if (!isActive) {
    drawRedactionUnanalyzed(r, false);
    return;
  }

  const a = r.analysis;
  const o = r.overrides || {};
  const fontName = state.fonts.find(f => f.id === (o.fontId ?? a.font.id))?.name ?? a.font.name;
  const fontSize = o.fontSize ?? a.font.size;
  const fontStr = `${fontSize}px "${fontName}"`;
  const gapW = o.gapWidth ?? a.gap.w;

  const startX = a.line.x + (o.offsetX ?? 0);
  const startY = a.line.y + (o.offsetY ?? 0);

  ctx.font = fontStr;
  ctx.textBaseline = "top";

  let cursorX = startX;

  const leftText = o.leftText ?? "";
  if (leftText) {
    ctx.fillStyle = "rgba(0, 200, 0, 0.7)";
    ctx.fillText(leftText, cursorX, startY);
    cursorX += ctx.measureText(leftText).width;
  }

  const pad = fontSize * 0.15;
  ctx.fillStyle = "rgba(211, 47, 47, 0.5)";
  ctx.fillRect(cursorX, startY - pad, gapW, fontSize + pad * 2);
  ctx.strokeStyle = "rgba(211, 47, 47, 0.8)";
  ctx.lineWidth = 2;
  ctx.strokeRect(cursorX, startY - pad, gapW, fontSize + pad * 2);

  ctx.fillStyle = "rgba(255, 255, 255, 0.8)";
  ctx.font = `bold ${Math.min(fontSize * 0.5, 16)}px sans-serif`;
  const label = `${Math.round(gapW)}px`;
  const labelW = ctx.measureText(label).width;
  ctx.fillText(label, cursorX + (gapW - labelW) / 2, startY + fontSize * 0.3);
  ctx.font = fontStr;

  cursorX += gapW;

  const rightText = o.rightText ?? "";
  if (rightText) {
    ctx.fillStyle = "rgba(0, 200, 0, 0.7)";
    ctx.fillText(rightText, cursorX, startY);
  }

  ctx.strokeStyle = "rgba(0, 200, 0, 0.3)";
  ctx.lineWidth = 1;
  ctx.strokeRect(a.line.x, a.line.y, a.line.w, a.line.h);
}

/**
 * @param {import('./types.js').Redaction} r
 * @param {boolean} isActive
 */
function drawRedactionPreview(r, isActive) {
  if (!r.analysis) return;
  const a = r.analysis;
  const o = r.overrides || {};
  const fontName = state.fonts.find(f => f.id === (o.fontId ?? a.font.id))?.name ?? a.font.name;
  const fontSize = o.fontSize ?? a.font.size;
  const fontStr = `${fontSize}px "${fontName}"`;
  const gapW = o.gapWidth ?? a.gap.w;

  if (isActive) {
    const startX = a.line.x + (o.offsetX ?? 0);
    const startY = a.line.y + (o.offsetY ?? 0);

    ctx.font = fontStr;
    ctx.textBaseline = "top";

    let cursorX = startX;

    const leftText = o.leftText ?? "";
    if (leftText) {
      ctx.fillStyle = "rgba(0, 200, 0, 0.7)";
      ctx.fillText(leftText, cursorX, startY);
      cursorX += ctx.measureText(leftText).width;
    }

    const pad = fontSize * 0.15;
    ctx.fillStyle = "rgba(255, 200, 0, 0.2)";
    ctx.fillRect(cursorX, startY - pad, gapW, fontSize + pad * 2);
    ctx.strokeStyle = "rgba(255, 200, 0, 0.8)";
    ctx.lineWidth = 2;
    ctx.strokeRect(cursorX, startY - pad, gapW, fontSize + pad * 2);

    ctx.fillStyle = "rgba(255, 200, 0, 0.9)";
    ctx.font = fontStr;
    ctx.fillText(r.preview, cursorX, startY);

    cursorX += gapW;

    const rightText = o.rightText ?? "";
    if (rightText) {
      ctx.fillStyle = "rgba(0, 200, 0, 0.7)";
      ctx.fillText(rightText, cursorX, startY);
    }

    ctx.strokeStyle = "rgba(0, 200, 0, 0.3)";
    ctx.lineWidth = 1;
    ctx.strokeRect(a.line.x, a.line.y, a.line.w, a.line.h);
  } else {
    const pad = fontSize * 0.1;
    ctx.fillStyle = "rgba(255, 200, 0, 0.12)";
    ctx.fillRect(a.gap.x, r.y - pad, gapW, r.h + pad * 2);

    ctx.strokeStyle = "rgba(255, 200, 0, 0.5)";
    ctx.lineWidth = 1;
    ctx.strokeRect(a.gap.x, r.y - pad, gapW, r.h + pad * 2);

    ctx.font = fontStr;
    ctx.textBaseline = "top";
    ctx.fillStyle = "rgba(255, 200, 0, 0.9)";
    ctx.fillText(r.preview, a.gap.x, a.line.y);
  }
}

/**
 * @param {import('./types.js').Redaction} r
 * @param {boolean} isActive
 */
function drawRedactionSolution(r, isActive) {
  if (!r.analysis) return;
  const a = r.analysis;
  const o = r.overrides || {};
  const fontName = state.fonts.find(f => f.id === (o.fontId ?? a.font.id))?.name ?? a.font.name;
  const fontSize = o.fontSize ?? a.font.size;
  const fontStr = `${fontSize}px "${fontName}"`;
  const gapW = o.gapWidth ?? a.gap.w;

  if (isActive) {
    const startX = a.line.x + (o.offsetX ?? 0);
    const startY = a.line.y + (o.offsetY ?? 0);

    ctx.font = fontStr;
    ctx.textBaseline = "top";

    let cursorX = startX;

    const leftText = o.leftText ?? "";
    if (leftText) {
      ctx.fillStyle = "rgba(0, 200, 0, 0.7)";
      ctx.fillText(leftText, cursorX, startY);
      cursorX += ctx.measureText(leftText).width;
    }

    const pad = fontSize * 0.15;
    ctx.fillStyle = "rgba(0, 212, 116, 0.15)";
    ctx.fillRect(cursorX, startY - pad, gapW, fontSize + pad * 2);
    ctx.strokeStyle = "rgba(0, 212, 116, 0.8)";
    ctx.lineWidth = 2;
    ctx.strokeRect(cursorX, startY - pad, gapW, fontSize + pad * 2);

    ctx.fillStyle = "rgba(0, 212, 116, 0.95)";
    ctx.font = fontStr;
    ctx.fillText(r.solution.text, cursorX, startY);

    cursorX += gapW;

    const rightText = o.rightText ?? "";
    if (rightText) {
      ctx.fillStyle = "rgba(0, 200, 0, 0.7)";
      ctx.fillText(rightText, cursorX, startY);
    }

    ctx.strokeStyle = "rgba(0, 200, 0, 0.3)";
    ctx.lineWidth = 1;
    ctx.strokeRect(a.line.x, a.line.y, a.line.w, a.line.h);
  } else {
    const pad = fontSize * 0.1;
    ctx.fillStyle = "rgba(0, 212, 116, 0.08)";
    ctx.fillRect(a.gap.x, r.y - pad, gapW, r.h + pad * 2);

    ctx.strokeStyle = "rgba(0, 212, 116, 0.4)";
    ctx.lineWidth = 1;
    ctx.strokeRect(a.gap.x, r.y - pad, gapW, r.h + pad * 2);

    ctx.font = fontStr;
    ctx.textBaseline = "top";
    ctx.fillStyle = "rgba(0, 212, 116, 0.95)";
    ctx.fillText(r.solution.text, a.gap.x, a.line.y);
  }
}
```

---

### Task 5: Create viewport.js

**Files:** Create `unredact/static/viewport.js`

**Step 1: Write viewport.js**

```js
// @ts-check
/** Viewport controls — zoom, pan, touch, resize, and coordinate transforms. */

import { state, getPageRedactions } from './state.js';
import { rightPanel, docContainer, zoomLevel, docImage, canvas, popoverEl, fontToolbar, textEditBar, zoomInBtn, zoomOutBtn, zoomFitBtn } from './dom.js';
import { renderCanvas } from './canvas.js';

/**
 * Apply the current zoom/pan transform to the document container.
 * @param {boolean} smooth
 */
export function applyTransform(smooth) {
  const pw = rightPanel.clientWidth;
  const ph = rightPanel.clientHeight;
  const tx = pw / 2 - state.panX * state.zoom;
  const ty = ph / 2 - state.panY * state.zoom;

  if (smooth) {
    docContainer.style.transition = "transform 0.25s ease-out";
  } else {
    docContainer.style.transition = "none";
  }

  docContainer.style.transform =
    `translate(${tx}px, ${ty}px) scale(${state.zoom})`;
  zoomLevel.textContent = `${Math.round(state.zoom * 100)}%`;
}

/**
 * Convert screen coordinates to document coordinates.
 * @param {number} sx
 * @param {number} sy
 * @returns {{ x: number, y: number }}
 */
export function screenToDoc(sx, sy) {
  const pw = rightPanel.clientWidth;
  const ph = rightPanel.clientHeight;
  return {
    x: (sx - pw / 2) / state.zoom + state.panX,
    y: (sy - ph / 2) / state.zoom + state.panY,
  };
}

/**
 * Zoom to a new level, optionally pivoting around screen coordinates.
 * @param {number} newZoom
 * @param {number} [pivotSX]
 * @param {number} [pivotSY]
 * @param {boolean} [smooth]
 */
export function zoomTo(newZoom, pivotSX, pivotSY, smooth) {
  newZoom = Math.max(0.1, Math.min(20, newZoom));
  if (pivotSX !== undefined) {
    const doc = screenToDoc(pivotSX, pivotSY);
    state.panX = doc.x;
    state.panY = doc.y;
    const pw = rightPanel.clientWidth;
    const ph = rightPanel.clientHeight;
    state.panX += (pw / 2 - pivotSX) / newZoom;
    state.panY += (ph / 2 - pivotSY) / newZoom;
  }
  state.zoom = newZoom;
  applyTransform(!!smooth);
}

export function zoomToFit() {
  if (!docImage.naturalWidth) return;
  const pw = rightPanel.clientWidth;
  const ph = rightPanel.clientHeight;
  const iw = docImage.naturalWidth;
  const ih = docImage.naturalHeight;
  state.zoom = Math.min(pw / iw, ph / ih) * 0.95;
  state.panX = iw / 2;
  state.panY = ih / 2;
  applyTransform(true);
}

/**
 * Test if document coordinates hit a redaction box.
 * @param {number} docX
 * @param {number} docY
 * @returns {import('./types.js').Redaction|null}
 */
export function hitTestRedaction(docX, docY) {
  const redactions = getPageRedactions();
  for (const r of redactions) {
    if (docX >= r.x && docX <= r.x + r.w && docY >= r.y && docY <= r.y + r.h) {
      return r;
    }
  }
  return null;
}

/** @param {Event} e */
function isPopoverArea(e) {
  return popoverEl.contains(/** @type {Node} */ (e.target)) ||
    fontToolbar.contains(/** @type {Node} */ (e.target)) ||
    textEditBar.contains(/** @type {Node} */ (e.target));
}

/** Set up all viewport event listeners. Call once from main.js. */
export function initViewport() {
  // Zoom buttons
  zoomInBtn.addEventListener("click", () => {
    zoomTo(state.zoom * 1.3, undefined, undefined, true);
  });
  zoomOutBtn.addEventListener("click", () => {
    zoomTo(state.zoom / 1.3, undefined, undefined, true);
  });
  zoomFitBtn.addEventListener("click", zoomToFit);

  // Mouse-wheel zoom toward cursor
  rightPanel.addEventListener("wheel", (e) => {
    if (isPopoverArea(e)) return;
    e.preventDefault();
    const rect = rightPanel.getBoundingClientRect();
    const sx = e.clientX - rect.left;
    const sy = e.clientY - rect.top;
    const factor = Math.pow(1.002, -e.deltaY);
    zoomTo(state.zoom * factor, sx, sy, false);
  }, { passive: false });

  // Click-drag pan
  let drag = null;

  rightPanel.addEventListener("mousedown", (e) => {
    if (e.button !== 0) return;
    if (isPopoverArea(e)) return;
    drag = {
      startX: e.clientX,
      startY: e.clientY,
      startPanX: state.panX,
      startPanY: state.panY,
      moved: false,
    };
    rightPanel.classList.add("panning");
    e.preventDefault();
  });

  window.addEventListener("mousemove", (e) => {
    if (!drag) return;
    const dx = e.clientX - drag.startX;
    const dy = e.clientY - drag.startY;
    if (Math.abs(dx) > 2 || Math.abs(dy) > 2) drag.moved = true;
    state.panX = drag.startPanX - dx / state.zoom;
    state.panY = drag.startPanY - dy / state.zoom;
    applyTransform(false);
  });

  window.addEventListener("mouseup", () => {
    if (drag) {
      drag = null;
      rightPanel.classList.remove("panning");
    }
  });

  // Touch: pinch-zoom + two-finger pan
  let lastTouches = null;

  rightPanel.addEventListener("touchstart", (e) => {
    if (e.touches.length === 2) {
      e.preventDefault();
      lastTouches = Array.from(e.touches);
    }
  }, { passive: false });

  rightPanel.addEventListener("touchmove", (e) => {
    if (e.touches.length === 2 && lastTouches) {
      e.preventDefault();
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
    }
  }, { passive: false });

  rightPanel.addEventListener("touchend", () => {
    lastTouches = null;
  });

  // Resize handling
  const resizeObserver = new ResizeObserver(() => {
    applyTransform(false);
    renderCanvas();
  });
  resizeObserver.observe(rightPanel);

  // On image load, fit to viewport
  docImage.addEventListener("load", () => {
    zoomToFit();
    renderCanvas();
  });

  // Canvas pointer style
  canvas.style.pointerEvents = "auto";
}
```

---

### Task 6: Create associates.js

**Files:** Create `unredact/static/associates.js`

**Step 1: Write associates.js**

```js
// @ts-check
/** Associate matching — fuzzy name lookup against the Epstein associate database. */

import { state } from './state.js';
import { solveFilterPrefix, solveFilterSuffix, popoverEl } from './dom.js';
import { escapeHtml } from './dom.js';

const MATCH_TYPE_WEIGHTS = {
  full: 4,
  nickname_full: 3,
  initial_last: 2,
  last: 2,
  first: 1,
  nickname: 1,
};

/**
 * Match text against the associate database.
 * @param {string} text
 * @returns {Array<{personId: string, personName: string, category: string, tier: number, matchType: string, score: number}>}
 */
export function matchAssociates(text) {
  if (!state.associates?.names) return [];

  const prefix = solveFilterPrefix.value.toLowerCase().trim();
  const suffix = solveFilterSuffix.value.toLowerCase().trim();
  const gapKey = text.toLowerCase().trim();

  const keysToTry = new Set([gapKey]);
  if (prefix || suffix) keysToTry.add(prefix + gapKey + suffix);
  if (prefix) keysToTry.add(prefix + gapKey);
  if (suffix) keysToTry.add(gapKey + suffix);

  const bestByPerson = new Map();

  for (const key of keysToTry) {
    const entries = state.associates.names[key];
    if (!entries) continue;
    const isComposite = key !== gapKey;

    for (const m of entries) {
      const person = state.associates.persons[m.person_id];
      const weight = MATCH_TYPE_WEIGHTS[m.match_type] || 1;
      let score = (4 - m.tier) * weight;
      if (isComposite) score += 3;

      const existing = bestByPerson.get(m.person_id);
      if (!existing || score > existing.score) {
        bestByPerson.set(m.person_id, {
          personId: m.person_id,
          personName: person?.name || "Unknown",
          category: person?.category || "other",
          tier: m.tier,
          matchType: isComposite ? `${m.match_type} (${key})` : m.match_type,
          score,
        });
      }
    }
  }

  return [...bestByPerson.values()].sort((a, b) => b.score - a.score);
}

/**
 * @param {number} tier
 * @returns {string}
 */
export function tierBadgeClass(tier) {
  if (tier === 1) return "tier-1";
  if (tier === 2) return "tier-2";
  return "tier-3";
}

/**
 * @param {number} tier
 * @returns {string}
 */
export function tierLabel(tier) {
  if (tier === 1) return "T1";
  if (tier === 2) return "T2";
  return "T3";
}

/**
 * @param {number} tier
 * @returns {string}
 */
export function tierDescription(tier) {
  if (tier === 1) return "Flight logs -- traveled with Epstein";
  if (tier === 2) return "Inner circle -- staff, financial, or frequently named";
  return "Named in Epstein case files";
}

/**
 * Check if text matches a known victim name.
 * @param {string} text
 * @returns {boolean}
 */
export function isVictimMatch(text) {
  const vs = state.associates?.victim_set;
  if (!vs || vs.size === 0) return false;
  const key = text.toLowerCase().trim();
  if (vs.has(key)) return true;
  const prefix = solveFilterPrefix.value.toLowerCase().trim();
  const suffix = solveFilterSuffix.value.toLowerCase().trim();
  if (prefix || suffix) {
    if (vs.has(prefix + key + suffix)) return true;
    if (prefix && vs.has(prefix + key)) return true;
    if (suffix && vs.has(key + suffix)) return true;
  }
  return false;
}

/**
 * Show a detail popup for associate matches.
 * @param {Array<{personName: string, tier: number, category: string, matchType: string}>} assocMatches
 * @param {HTMLElement} _anchorEl
 */
export function showAssocDetail(assocMatches, _anchorEl) {
  const old = document.getElementById("assoc-detail");
  if (old) old.remove();

  const popup = document.createElement("div");
  popup.id = "assoc-detail";

  let html = '<div class="assoc-detail-header">Possible associates<button class="assoc-detail-close">X</button></div>';
  html += '<div class="assoc-detail-list">';

  for (const m of assocMatches) {
    const cls = tierBadgeClass(m.tier);
    html += `<div class="assoc-detail-item">
      <span class="assoc-badge ${cls}">${tierLabel(m.tier)}</span>
      <div class="assoc-detail-info">
        <div class="assoc-detail-name">${escapeHtml(m.personName)}</div>
        <div class="assoc-detail-meta">${escapeHtml(tierDescription(m.tier))} · ${escapeHtml(m.category)} · matched on ${escapeHtml(m.matchType)}</div>
      </div>
    </div>`;
  }

  html += '</div>';
  popup.innerHTML = html;

  popoverEl.appendChild(popup);

  popup.querySelector(".assoc-detail-close").addEventListener("click", (e) => {
    e.stopPropagation();
    popup.remove();
  });

  const closeOnOutside = (e) => {
    if (!popup.contains(/** @type {Node} */ (e.target))) {
      popup.remove();
      document.removeEventListener("click", closeOnOutside, true);
    }
  };
  setTimeout(() => document.addEventListener("click", closeOnOutside, true), 0);
}
```

---

### Task 7: Create popover.js

**Files:** Create `unredact/static/popover.js`

**Step 1: Write popover.js**

```js
// @ts-check
/** Popover UI — font toolbar, text editing, and solve control panel. */

import { state } from './state.js';
import {
  popoverEl, fontToolbar, textEditBar, popoverClose,
  fontSelect, sizeSlider, sizeValue, sizeDown, sizeUp,
  posUp, posDown, posLeft, posRight, posReset, posDisplay,
  gapDown, gapUp, gapValue,
  leftTextInput, rightTextInput, redactionMarker, textReset,
  solveResults, solveStatus, solveStart, solveStop, solveAccept,
  solveTolerance, solveTolValue,
} from './dom.js';
import { renderCanvas } from './canvas.js';

/** @type {(() => void)|null} */
let _onClose = null;

/**
 * Register a callback to run when the popover closes (used to stop solver).
 * @param {() => void} fn
 */
export function setOnPopoverClose(fn) { _onClose = fn; }

/**
 * Open the popover for a redaction.
 * @param {string} id
 */
export function openPopover(id) {
  const r = state.redactions[id];
  if (!r || !r.analysis) return;

  solveFilterPrefix.value = "";
  solveFilterSuffix.value = "";

  solveResults.innerHTML = "";
  solveStatus.textContent = "";
  solveStart.hidden = false;
  solveStop.hidden = true;
  solveAccept.hidden = !!(r.preview === null);

  popoverEl.hidden = false;

  fontToolbar.hidden = false;
  fontSelect.value = r.overrides.fontId;
  sizeSlider.value = String(r.overrides.fontSize);
  sizeValue.textContent = String(r.overrides.fontSize);
  gapValue.textContent = String(Math.round(r.overrides.gapWidth));
  updatePosDisplay();

  textEditBar.hidden = false;
  leftTextInput.value = r.overrides.leftText;
  rightTextInput.value = r.overrides.rightText;
  redactionMarker.textContent = r.preview || "???";
  redactionMarker.className = r.preview ? "redaction-marker preview" : "redaction-marker";
}

export function closePopover() {
  popoverEl.hidden = true;
  fontToolbar.hidden = true;
  textEditBar.hidden = true;
  if (_onClose) _onClose();
}

export function updatePosDisplay() {
  const r = state.redactions[state.activeRedaction];
  if (!r?.overrides) return;
  posDisplay.textContent = `${Math.round(r.overrides.offsetX)}, ${Math.round(r.overrides.offsetY)}`;
}

function adjustSize(delta) {
  const r = state.redactions[state.activeRedaction];
  if (!r?.overrides) return;
  r.overrides.fontSize = Math.max(8, Math.min(120, r.overrides.fontSize + delta));
  sizeSlider.value = String(r.overrides.fontSize);
  sizeValue.textContent = String(r.overrides.fontSize);
  renderCanvas();
}

function nudge(dx, dy) {
  const r = state.redactions[state.activeRedaction];
  if (!r?.overrides) return;
  r.overrides.offsetX += dx;
  r.overrides.offsetY += dy;
  updatePosDisplay();
  renderCanvas();
}

function adjustGap(delta) {
  const r = state.redactions[state.activeRedaction];
  if (!r?.overrides) return;
  r.overrides.gapWidth = Math.max(1, r.overrides.gapWidth + delta);
  gapValue.textContent = String(Math.round(r.overrides.gapWidth));
  renderCanvas();
}

/** Set up all popover event listeners. Call once from main.js. */
export function initPopover() {
  popoverClose.addEventListener("click", closePopover);

  fontSelect.addEventListener("change", () => {
    const r = state.redactions[state.activeRedaction];
    if (!r?.overrides) return;
    r.overrides.fontId = fontSelect.value;
    renderCanvas();
  });

  sizeSlider.addEventListener("input", () => {
    const r = state.redactions[state.activeRedaction];
    if (!r?.overrides) return;
    r.overrides.fontSize = parseInt(sizeSlider.value);
    sizeValue.textContent = sizeSlider.value;
    renderCanvas();
  });

  sizeDown.addEventListener("click", () => adjustSize(-1));
  sizeUp.addEventListener("click", () => adjustSize(1));

  posUp.addEventListener("click", () => nudge(0, -1));
  posDown.addEventListener("click", () => nudge(0, 1));
  posLeft.addEventListener("click", () => nudge(-1, 0));
  posRight.addEventListener("click", () => nudge(1, 0));

  posReset.addEventListener("click", () => {
    const r = state.redactions[state.activeRedaction];
    if (!r?.overrides || !r.analysis) return;
    r.overrides.offsetX = r.analysis.offset_x || 0;
    r.overrides.offsetY = r.analysis.offset_y || 0;
    updatePosDisplay();
    renderCanvas();
  });

  gapDown.addEventListener("click", () => adjustGap(-1));
  gapUp.addEventListener("click", () => adjustGap(1));

  leftTextInput.addEventListener("input", () => {
    const r = state.redactions[state.activeRedaction];
    if (!r?.overrides) return;
    r.overrides.leftText = leftTextInput.value;
    renderCanvas();
  });

  rightTextInput.addEventListener("input", () => {
    const r = state.redactions[state.activeRedaction];
    if (!r?.overrides) return;
    r.overrides.rightText = rightTextInput.value;
    renderCanvas();
  });

  textReset.addEventListener("click", () => {
    const r = state.redactions[state.activeRedaction];
    if (!r?.overrides || !r.analysis) return;
    const a = r.analysis;
    r.overrides.leftText = a.segments.length > 0 ? a.segments[0].text : "";
    r.overrides.rightText = a.segments.length > 1 ? a.segments[1].text : "";
    leftTextInput.value = r.overrides.leftText;
    rightTextInput.value = r.overrides.rightText;
    renderCanvas();
  });

  solveTolerance.addEventListener("input", () => {
    solveTolValue.textContent = solveTolerance.value;
  });
}
```

Note: `solveFilterPrefix` and `solveFilterSuffix` are imported in openPopover to reset them but they're also used in associates.js. Both import from dom.js — no conflict.

---

### Task 8: Create solver.js

**Files:** Create `unredact/static/solver.js`

**Step 1: Write solver.js**

```js
// @ts-check
/** Solve engine — SSE streaming constraint solver with associate matching. */

import { state } from './state.js';
import {
  solveCharset, solveTolerance, solveMode, solveFilter,
  solveFilterPrefix, solveFilterSuffix,
  solveResults, solveStatus, solveStart, solveStop,
  solveAccept, redactionMarker,
} from './dom.js';
import { escapeHtml } from './dom.js';
import { renderCanvas } from './canvas.js';
import { matchAssociates, tierBadgeClass, tierLabel, isVictimMatch, showAssocDetail } from './associates.js';

/** @type {AbortController|null} */
let activeEventSource = null;

export function startSolve() {
  const id = state.activeRedaction;
  if (!id) return;
  const r = state.redactions[id];
  if (!r || !r.analysis) return;

  const a = r.analysis;
  const o = r.overrides || {};
  const fontId = o.fontId ?? a.font.id;
  const fontSize = o.fontSize ?? a.font.size;
  const gapWidth = o.gapWidth ?? a.gap.w;

  const leftText = o.leftText ?? (a.segments.length > 0 ? a.segments[0].text : "");
  const rightText = o.rightText ?? (a.segments.length > 1 ? a.segments[1].text : "");
  const leftCtx = leftText.length > 0 ? leftText[leftText.length - 1] : "";
  const rightCtx = rightText.length > 0 ? rightText[0] : "";

  solveResults.innerHTML = "";
  solveStatus.textContent = "Starting...";
  solveStart.hidden = true;
  solveStop.hidden = false;
  solveAccept.hidden = true;

  const body = {
    font_id: fontId,
    font_size: fontSize,
    gap_width_px: gapWidth,
    tolerance_px: parseFloat(solveTolerance.value),
    left_context: leftCtx,
    right_context: rightCtx,
    hints: {
      charset: solveCharset.value,
    },
    mode: solveMode.value,
    word_filter: solveFilter.value,
    filter_prefix: solveFilterPrefix.value,
    filter_suffix: solveFilterSuffix.value,
  };

  const abortController = new AbortController();
  activeEventSource = abortController;

  fetch("/api/solve", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
    signal: abortController.signal,
  }).then(response => {
    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let buffer = "";

    function read() {
      reader.read().then(({ done, value }) => {
        if (done) {
          solveStart.hidden = false;
          solveStop.hidden = true;
          return;
        }
        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split("\n");
        buffer = lines.pop();

        for (const line of lines) {
          if (line.startsWith("data: ")) {
            try {
              const data = JSON.parse(line.slice(6));
              handleSolveEvent(data, id);
            } catch (e) { /* skip malformed */ }
          }
        }
        read();
      });
    }
    read();
  }).catch(err => {
    if (err.name !== "AbortError") {
      solveStatus.textContent = "Error: " + err.message;
    }
    solveStart.hidden = false;
    solveStop.hidden = true;
  });
}

/**
 * @param {Object} data
 * @param {string} redactionId
 */
function handleSolveEvent(data, redactionId) {
  if (data.status === "match") {
    const assocMatches = matchAssociates(data.text);
    const topMatch = assocMatches.length > 0 ? assocMatches[0] : null;

    const div = document.createElement("div");
    div.className = "solve-result";
    if (topMatch) {
      div.dataset.assocTier = String(topMatch.tier);
      div.dataset.assocScore = String(topMatch.score);
    }

    div.innerHTML = `
      <span class="result-text">${escapeHtml(data.text)}</span>
      <span class="result-error">${data.error_px.toFixed(1)}px ${data.source || ""}</span>
    `;

    const victim = isVictimMatch(data.text);

    if (victim) {
      const vBadge = document.createElement("span");
      vBadge.className = "assoc-badge victim";
      vBadge.textContent = "V";
      vBadge.title = "Matches a known victim name";
      div.prepend(vBadge);
    }

    if (assocMatches.length > 0) {
      const badge = document.createElement("button");
      badge.className = `assoc-badge ${tierBadgeClass(topMatch.tier)}`;
      badge.textContent = tierLabel(topMatch.tier);
      badge.title = "Click for details";
      badge.addEventListener("click", (e) => {
        e.stopPropagation();
        showAssocDetail(assocMatches, badge);
      });
      div.prepend(badge);
    }

    div.addEventListener("click", () => {
      const r = state.redactions[redactionId];
      if (!r) return;
      r.preview = data.text;
      redactionMarker.textContent = data.text;
      redactionMarker.className = "redaction-marker preview";
      renderCanvas();
      solveResults.querySelectorAll(".solve-result").forEach(el => el.classList.remove("active"));
      div.classList.add("active");
      solveAccept.hidden = false;
    });

    if (topMatch) {
      let inserted = false;
      for (const existing of solveResults.children) {
        const exTier = parseInt(/** @type {HTMLElement} */ (existing).dataset.assocTier || "99");
        const exScore = parseFloat(/** @type {HTMLElement} */ (existing).dataset.assocScore || "0");
        if (topMatch.tier < exTier || (topMatch.tier === exTier && topMatch.score > exScore)) {
          solveResults.insertBefore(div, existing);
          inserted = true;
          break;
        }
      }
      if (!inserted) solveResults.appendChild(div);
    } else {
      solveResults.appendChild(div);
    }

    solveStatus.textContent = `Found ${solveResults.children.length} matches`;
  } else if (data.status === "running") {
    solveStatus.textContent = `Checked ${data.checked}, found ${data.found}...`;
  } else if (data.status === "done") {
    solveStatus.textContent = `Done. ${data.total_found} total matches.`;
    solveStart.hidden = false;
    solveStop.hidden = true;
    activeEventSource = null;
  }
}

export function stopSolve() {
  if (activeEventSource) {
    activeEventSource.abort();
    activeEventSource = null;
  }
  solveStart.hidden = false;
  solveStop.hidden = true;
  solveStatus.textContent = "Stopped.";
}

/**
 * Accept the current preview as the solution. Only mutates state —
 * caller (main.js) is responsible for UI updates (renderRedactionList, closePopover).
 */
export function acceptSolution() {
  const id = state.activeRedaction;
  if (!id) return;
  const r = state.redactions[id];
  if (!r || !r.preview) return;

  r.status = "solved";
  const o = r.overrides || {};
  const solFontName = state.fonts.find(f => f.id === (o.fontId || r.analysis.font.id))?.name || r.analysis.font.name;
  r.solution = {
    text: r.preview,
    fontName: solFontName,
    fontSize: o.fontSize || r.analysis.font.size,
  };
  r.preview = null;
}

/** Set up solver button listeners. Call once from main.js. */
export function initSolver() {
  solveStart.addEventListener("click", startSolve);
  solveStop.addEventListener("click", stopSolve);
}
```

---

### Task 9: Create main.js

**Files:** Create `unredact/static/main.js`

**Step 1: Write main.js**

```js
// @ts-check
/** Entry point — upload, page navigation, redaction management, and event wiring. */

import { state } from './state.js';
import {
  dropZone, fileInput, uploadSection, viewerSection, docImage,
  canvas, pageInfo, prevBtn, nextBtn, redactionListEl,
  rightPanel, popoverEl, fontToolbar, textEditBar, fontSelect,
  solveAccept, gapValue, showToast,
} from './dom.js';
import { renderCanvas } from './canvas.js';
import { applyTransform, screenToDoc, hitTestRedaction, initViewport } from './viewport.js';
import { openPopover, closePopover, setOnPopoverClose, updatePosDisplay, initPopover } from './popover.js';
import { stopSolve, acceptSolution, initSolver } from './solver.js';

// ── Font loading ──

async function loadFonts() {
  const resp = await fetch("/api/fonts");
  const data = await resp.json();
  state.fonts = data.fonts;

  const promises = state.fonts
    .filter((f) => f.available)
    .map(async (f) => {
      const face = new FontFace(f.name, `url(/api/font/${f.id})`);
      try {
        const loaded = await face.load();
        document.fonts.add(loaded);
      } catch (e) {
        console.warn(`Failed to load font ${f.name}:`, e);
      }
    });

  await Promise.all(promises);
  state.fontsReady = true;

  fontSelect.innerHTML = "";
  for (const f of state.fonts.filter(f => f.available)) {
    const opt = document.createElement("option");
    opt.value = f.id;
    opt.textContent = f.name;
    fontSelect.appendChild(opt);
  }
}

async function loadAssociates() {
  try {
    const resp = await fetch("/api/associates");
    state.associates = await resp.json();
    state.associates.victim_set = new Set(state.associates.victim_names || []);
    console.log(`Loaded ${Object.keys(state.associates.names).length} associate lookups, ${state.associates.victim_set.size} victim names`);
  } catch (e) {
    console.warn("Failed to load associates data:", e);
    state.associates = { names: {}, persons: {}, victim_set: new Set() };
  }
}

// ── Upload & drag-drop ──

dropZone.addEventListener("click", () => fileInput.click());
dropZone.addEventListener("dragover", (e) => {
  e.preventDefault();
  dropZone.classList.add("dragover");
});
dropZone.addEventListener("dragleave", () => dropZone.classList.remove("dragover"));
dropZone.addEventListener("drop", (e) => {
  e.preventDefault();
  dropZone.classList.remove("dragover");
  if (e.dataTransfer.files.length) uploadFile(e.dataTransfer.files[0]);
});
fileInput.addEventListener("change", () => {
  if (fileInput.files.length) uploadFile(fileInput.files[0]);
});

async function uploadFile(file) {
  uploadSection.innerHTML = '<p class="loading">Analyzing document...</p>';

  const fontPromise = loadFonts();
  const assocPromise = loadAssociates();

  const form = new FormData();
  form.append("file", file);
  const resp = await fetch("/api/upload", { method: "POST", body: form });
  const data = await resp.json();

  state.docId = data.doc_id;
  state.pageCount = data.page_count;
  state.currentPage = 1;

  await Promise.all([fontPromise, assocPromise]);

  uploadSection.hidden = true;
  viewerSection.hidden = false;

  await loadPage(1);
}

// ── Page loading ──

async function loadPage(page) {
  state.currentPage = page;
  state.activeRedaction = null;
  closePopover();
  updatePageControls();

  docImage.src = `/api/doc/${state.docId}/page/${page}/original`;

  const resp = await fetch(`/api/doc/${state.docId}/page/${page}/data`);
  const data = await resp.json();

  for (const r of data.redactions) {
    if (!state.redactions[r.id]) {
      state.redactions[r.id] = {
        id: r.id,
        x: r.x,
        y: r.y,
        w: r.w,
        h: r.h,
        page: page,
        status: "unanalyzed",
        analysis: null,
        solution: null,
        preview: null,
      };
    }
  }

  renderRedactionList();
  renderCanvas();
}

function updatePageControls() {
  pageInfo.textContent = `Page ${state.currentPage} / ${state.pageCount}`;
  prevBtn.disabled = state.currentPage <= 1;
  nextBtn.disabled = state.currentPage >= state.pageCount;
}

prevBtn.addEventListener("click", () => {
  if (state.currentPage > 1) loadPage(state.currentPage - 1);
});
nextBtn.addEventListener("click", () => {
  if (state.currentPage < state.pageCount) loadPage(state.currentPage + 1);
});

// ── Redaction list (left panel) ──

function renderRedactionList() {
  const redactions = Object.values(state.redactions)
    .filter((r) => r.page === state.currentPage)
    .sort((a, b) => {
      if (Math.abs(a.y - b.y) > 5) return a.y - b.y;
      return a.x - b.x;
    });
  redactionListEl.innerHTML = "";

  redactions.forEach((r, idx) => {
    const div = document.createElement("div");
    div.className = "redaction-item";
    if (r.id === state.activeRedaction) div.classList.add("active");
    div.dataset.id = r.id;

    const numEl = document.createElement("span");
    numEl.className = "redaction-num";
    numEl.textContent = `#${idx + 1}`;

    const statusEl = document.createElement("span");
    statusEl.className = `redaction-status status-${r.status}`;
    statusEl.textContent = statusLabel(r.status);

    const infoEl = document.createElement("div");
    infoEl.className = "redaction-info";
    infoEl.textContent = redactionInfoText(r);

    const headerRow = document.createElement("div");
    headerRow.className = "redaction-header-row";
    headerRow.appendChild(numEl);
    headerRow.appendChild(statusEl);

    div.appendChild(headerRow);
    div.appendChild(infoEl);

    div.addEventListener("click", () => activateRedaction(r.id));
    redactionListEl.appendChild(div);
  });
}

function statusLabel(status) {
  switch (status) {
    case "unanalyzed": return "unanalyzed";
    case "analyzing": return "analyzing...";
    case "analyzed": return "analyzed";
    case "solved": return "solved";
    case "error": return "error";
    default: return status;
  }
}

function redactionInfoText(r) {
  if (r.status === "solved" && r.solution) {
    return r.solution.text;
  }
  if ((r.status === "analyzed" || r.status === "solved") && r.analysis) {
    const segs = r.analysis.segments;
    const left = segs.length > 0 ? segs[0].text : "";
    const right = segs.length > 1 ? segs[1].text : "";
    const leftTail = left.length > 15 ? "..." + left.slice(-15) : left;
    const rightHead = right.length > 15 ? right.slice(0, 15) + "..." : right;
    return `${leftTail} [___] ${rightHead}`;
  }
  return `${Math.round(r.w)} x ${Math.round(r.h)} px`;
}

// ── Activate & analyze redaction ──

function activateRedaction(id) {
  const r = state.redactions[id];
  if (!r) return;

  state.activeRedaction = id;

  state.panX = r.x + r.w / 2;
  state.panY = r.y + r.h / 2;
  applyTransform(true);

  renderRedactionList();
  renderCanvas();

  if (r.status === "unanalyzed") {
    analyzeRedaction(id);
  } else if (r.status === "analyzed" || r.status === "solved") {
    openPopover(id);
  }
}

async function analyzeRedaction(id) {
  const r = state.redactions[id];
  if (!r) return;

  r.status = "analyzing";
  renderRedactionList();

  try {
    const resp = await fetch("/api/redaction/analyze", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        doc_id: state.docId,
        page: r.page,
        redaction: { x: r.x, y: r.y, w: r.w, h: r.h },
      }),
    });

    if (!resp.ok) {
      const err = await resp.json();
      r.status = "error";
      console.error("Analysis failed:", err);
      renderRedactionList();
      return;
    }

    const data = await resp.json();
    r.status = "analyzed";
    r.analysis = data;
    r.overrides = {
      fontId: data.font.id,
      fontSize: data.font.size,
      offsetX: data.offset_x || 0,
      offsetY: data.offset_y || 0,
      gapWidth: data.gap.w,
      leftText: data.segments.length > 0 ? data.segments[0].text : "",
      rightText: data.segments.length > 1 ? data.segments[1].text : "",
    };
    renderRedactionList();
    renderCanvas();

    if (state.activeRedaction === id) {
      openPopover(id);
    }
  } catch (e) {
    r.status = "error";
    console.error("Analysis error:", e);
    renderRedactionList();
  }
}

// ── Canvas hit-testing ──

canvas.addEventListener("mousedown", (e) => {
  if (e.button !== 0) return;

  const rect = rightPanel.getBoundingClientRect();
  const sx = e.clientX - rect.left;
  const sy = e.clientY - rect.top;
  const doc = screenToDoc(sx, sy);

  const hit = hitTestRedaction(doc.x, doc.y);
  if (hit) {
    e.stopPropagation();
    activateRedaction(hit.id);
  }
});

canvas.addEventListener("mousemove", (e) => {
  const rect = rightPanel.getBoundingClientRect();
  const sx = e.clientX - rect.left;
  const sy = e.clientY - rect.top;
  const doc = screenToDoc(sx, sy);

  const hit = hitTestRedaction(doc.x, doc.y);
  canvas.style.cursor = hit ? "pointer" : "";
});

// ── Double-click spot detection ──

rightPanel.addEventListener("dblclick", async (e) => {
  if (popoverEl.contains(/** @type {Node} */ (e.target)) ||
      fontToolbar.contains(/** @type {Node} */ (e.target)) ||
      textEditBar.contains(/** @type {Node} */ (e.target))) return;
  if (!state.docId) return;

  const rect = rightPanel.getBoundingClientRect();
  const sx = e.clientX - rect.left;
  const sy = e.clientY - rect.top;
  const doc = screenToDoc(sx, sy);

  const hit = hitTestRedaction(doc.x, doc.y);
  if (hit) return;

  const clickX = Math.round(doc.x);
  const clickY = Math.round(doc.y);

  showToast("Detecting redaction...");

  try {
    const resp = await fetch("/api/redaction/spot", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        doc_id: state.docId,
        page: state.currentPage,
        click_x: clickX,
        click_y: clickY,
      }),
    });

    if (!resp.ok) {
      const err = await resp.json();
      showToast(err.error === "no_redaction_found" ? "No redaction found at click point" : "Detection failed", "error");
      return;
    }

    const data = await resp.json();
    const box = data.box;

    const existingDup = Object.values(state.redactions).find(r =>
      r.page === state.currentPage &&
      Math.abs(r.x - box.x) < 3 && Math.abs(r.y - box.y) < 3 &&
      Math.abs(r.w - box.w) < 3 && Math.abs(r.h - box.h) < 3
    );
    if (existingDup) {
      activateRedaction(existingDup.id);
      return;
    }

    const id = "m" + Date.now().toString(36);

    state.redactions[id] = {
      id,
      x: box.x,
      y: box.y,
      w: box.w,
      h: box.h,
      page: state.currentPage,
      status: data.segments ? "analyzed" : "unanalyzed",
      analysis: data.segments ? data : null,
      solution: null,
      preview: null,
    };

    if (data.segments) {
      state.redactions[id].overrides = {
        fontId: data.font.id,
        fontSize: data.font.size,
        offsetX: data.offset_x || 0,
        offsetY: data.offset_y || 0,
        gapWidth: data.gap.w,
        leftText: data.segments.length > 0 ? data.segments[0].text : "",
        rightText: data.segments.length > 1 ? data.segments[1].text : "",
      };
    }

    renderRedactionList();
    renderCanvas();
    activateRedaction(id);
  } catch (e) {
    console.error("Spot detection error:", e);
    showToast("Detection failed: " + e.message, "error");
  }
});

// ── Ctrl+drag offset / Shift+drag gap width ──

let modDrag = null;

canvas.addEventListener("mousedown", (e) => {
  if ((!e.ctrlKey && !e.shiftKey) || e.button !== 0) return;
  const r = state.redactions[state.activeRedaction];
  if (!r?.overrides) return;

  modDrag = {
    startX: e.clientX,
    startY: e.clientY,
    startOffsetX: r.overrides.offsetX,
    startOffsetY: r.overrides.offsetY,
    startGapWidth: r.overrides.gapWidth,
    widthMode: e.shiftKey && !e.ctrlKey,
  };
  e.stopPropagation();
  e.preventDefault();
}, { capture: true });

window.addEventListener("mousemove", (e) => {
  if (!modDrag) return;
  const r = state.redactions[state.activeRedaction];
  if (!r?.overrides) return;

  const dx = (e.clientX - modDrag.startX) / state.zoom;
  const dy = (e.clientY - modDrag.startY) / state.zoom;

  if (modDrag.widthMode) {
    r.overrides.gapWidth = Math.max(1, modDrag.startGapWidth + dx);
    gapValue.textContent = String(Math.round(r.overrides.gapWidth));
  } else {
    r.overrides.offsetX = modDrag.startOffsetX + dx;
    r.overrides.offsetY = modDrag.startOffsetY + dy;
    updatePosDisplay();
  }
  renderCanvas();
});

window.addEventListener("mouseup", () => {
  if (modDrag) modDrag = null;
});

// ── Accept solution (wired here to avoid circular dep solver↔main) ──

solveAccept.addEventListener("click", () => {
  acceptSolution();
  closePopover();
  renderRedactionList();
  renderCanvas();
});

// ── Initialize all modules ──

setOnPopoverClose(stopSolve);
initViewport();
initPopover();
initSolver();
```

---

### Task 10: Create jsconfig.json, update index.html, delete app.js

**Files:**
- Create: `unredact/static/jsconfig.json`
- Modify: `unredact/static/index.html:150`
- Delete: `unredact/static/app.js`

**Step 1: Write jsconfig.json**

```json
{
  "compilerOptions": {
    "checkJs": true,
    "target": "ES2022",
    "module": "ES2022"
  },
  "include": ["./**/*.js"]
}
```

**Step 2: Update index.html**

Change line 150 from:
```html
  <script src="/static/app.js"></script>
```
to:
```html
  <script type="module" src="/static/main.js"></script>
```

**Step 3: Delete app.js**

```bash
rm unredact/static/app.js
```

---

### Task 11: Verify and commit

**Step 1: Start the application**

```bash
make run
```

**Step 2: Manual smoke test**

Open http://localhost:8000 in a browser. Verify:
- [ ] Page loads without console errors
- [ ] PDF upload works (drag-drop and click)
- [ ] Page navigation works
- [ ] Redaction list populates
- [ ] Clicking a redaction activates it (pans to it, starts analysis)
- [ ] Popover opens with font/size/pos/gap controls
- [ ] Font/size/position/gap adjustments update the canvas
- [ ] Text editing works (left/right inputs)
- [ ] Solver starts, streams results, can be stopped
- [ ] Clicking a result shows preview on canvas
- [ ] Accept marks redaction as solved
- [ ] Double-click spot detection works
- [ ] Ctrl+drag adjusts offset, Shift+drag adjusts gap width
- [ ] Zoom (wheel, buttons, fit) works
- [ ] Touch pinch-zoom works (if testable)
- [ ] Toast notifications appear

**Step 3: Commit**

```bash
git add unredact/static/types.js unredact/static/state.js unredact/static/dom.js \
  unredact/static/canvas.js unredact/static/viewport.js unredact/static/associates.js \
  unredact/static/popover.js unredact/static/solver.js unredact/static/main.js \
  unredact/static/jsconfig.json unredact/static/index.html
git rm unredact/static/app.js
git commit -m "feat: split app.js into ES modules with JSDoc types

Reorganize the monolithic 1,436-line frontend into 9 focused modules:
types, state, dom, canvas, viewport, associates, popover, solver, main.

No build step — browsers load ES modules natively. JSDoc @typedef
annotations provide type safety via @ts-check + jsconfig.json."
```
