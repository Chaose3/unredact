# Frontend Modularization Design

## Problem

The frontend is a single 1,436-line `app.js` file handling state, canvas rendering, zoom/pan, SSE streaming, solver control, associate matching, and UI interactions. As the project grows and attracts contributors, this monolith becomes hard to navigate, refactor, and reason about — both for humans and code agents.

## Approach

Split `app.js` into ES modules with JSDoc type annotations. No build step — browsers natively support `<script type="module">`. VS Code enforces types via `// @ts-check` + `jsconfig.json`.

### Why not TypeScript?

- Introduces a build step (`tsc`) and `node_modules` dependency
- The zero-build simplicity of FastAPI serving static files directly is valuable
- JSDoc + `@ts-check` gives ~90% of TypeScript's benefits (autocomplete, type errors, hover docs) with zero tooling
- If TS is desired later, the migration from JSDoc-typed ES modules to `.ts` files is mechanical

## Module Structure

| File | ~Lines | Responsibility |
|------|--------|----------------|
| `types.js` | 30 | JSDoc `@typedef` definitions — the data contract layer |
| `state.js` | 30 | Global `state` object + `getPageRedactions()` |
| `dom.js` | 65 | All `getElementById` refs + `showToast()` + `escapeHtml()` |
| `canvas.js` | 250 | `renderCanvas()` + all `drawRedaction*()` functions |
| `viewport.js` | 280 | Transforms, zoom, pan, touch, resize, hit-testing, ctrl/shift drag |
| `popover.js` | 140 | `openPopover()`, `closePopover()`, font/size/pos/gap/text controls |
| `solver.js` | 200 | SSE solve engine: `startSolve()`, `stopSolve()`, `handleSolveEvent()`, `acceptSolution()` |
| `associates.js` | 130 | `matchAssociates()`, tier badges, `showAssocDetail()`, `isVictimMatch()` |
| `main.js` | 170 | Entry point: upload, drag-drop, page nav, font/associate loading, init wiring |

### Agent Visibility Optimizations

- **Small, focused files** — each module fits comfortably in a single agent context read
- **`types.js` as source of truth** — an agent reads this first to understand all data shapes
- **Explicit imports/exports** — every module declares what it provides and consumes
- **One-line module header** — e.g. `/** Canvas rendering — draws redaction overlays on the document image. */`
- **Named DOM imports** — each module imports only the elements it touches, making dependencies obvious

## Dependency Graph

```
types.js          (no imports)
state.js          ← types.js
dom.js            (no imports)
canvas.js         ← state.js, dom.js
viewport.js       ← state.js, dom.js, canvas.js
associates.js     ← state.js, dom.js
popover.js        ← state.js, dom.js, canvas.js
solver.js         ← state.js, dom.js, canvas.js, associates.js, popover.js
main.js           ← state.js, dom.js, canvas.js, viewport.js, popover.js, solver.js
```

No circular dependencies. The one tricky spot (`closePopover` calls `stopSolve`, `acceptSolution` calls `closePopover`) is resolved via a callback registration pattern:

```js
// popover.js
let _onClose = null;
export function setOnPopoverClose(fn) { _onClose = fn; }
export function closePopover() {
  // ... hide elements ...
  if (_onClose) _onClose();
}

// main.js
import { setOnPopoverClose } from './popover.js';
import { stopSolve } from './solver.js';
setOnPopoverClose(stopSolve);
```

## Type Safety

Every file starts with `// @ts-check`. Key type definitions in `types.js`:

```js
/** @typedef {Object} Redaction
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

/** @typedef {Object} Analysis ... */
/** @typedef {Object} Overrides ... */
/** @typedef {Object} Solution ... */
/** @typedef {Object} AppState ... */
/** @typedef {Object} Font ... */
/** @typedef {Object} AssociatesData ... */
```

## Editor Configuration

A `jsconfig.json` in `unredact/static/`:

```json
{
  "compilerOptions": {
    "checkJs": true,
    "strict": true,
    "target": "ES2022",
    "module": "ES2022"
  },
  "include": ["./**/*.js"]
}
```

## HTML Change

```html
<!-- before -->
<script src="/static/app.js"></script>

<!-- after -->
<script type="module" src="/static/main.js"></script>
```

## Backend Changes

None. FastAPI's `StaticFiles` mount serves all `.js` files. ES module imports resolve via HTTP requests automatically.

## Verification

- All existing functionality works identically after the split
- No new dependencies, no `package.json`, no `node_modules`
- VS Code shows type errors via `@ts-check` without any build command
