# Mobile-Friendly Frontend Design

## Approach

CSS-only responsive using `@media (max-width: 768px)` media queries plus minimal JS for tab switching. No new HTML pages, no server-side UA detection. Desktop layout is untouched.

## Layout & Navigation

At ≤768px, a tab bar appears at the top of the workspace with two tabs: **Document** and **Redactions**.

- **Document tab (default):** Right panel takes full width/height. Left panel hidden.
- **Redactions tab:** Left panel takes full width/height. Right panel hidden.
- Tapping a redaction in the list auto-switches to the Document tab.
- Header shrinks: smaller font, tighter padding.
- Controls bar uses `flex-wrap: wrap` so page nav and zoom controls flow naturally.

## Solver Panel

The popover becomes a full-screen modal overlay:

- `position: fixed; inset: 0` covering the entire viewport.
- Full opacity background (no backdrop-filter needed).
- X button to close. Scrollable content.
- Internal structure unchanged — Mode, Charset, Tolerance, Known start/end, Solve button, results list all stay the same, just full-width.

Font toolbar becomes full-width at top of document view (remove `right: 340px` constraint, allow `flex-wrap`).

Text edit bar already stretches near full-width; no major changes.

## Touch & Interaction

- **Pinch-zoom & two-finger pan:** Already implemented in viewport.js. No changes.
- **Single-finger pan:** Add single-touch drag support in viewport.js (touchstart/touchmove with 1 finger).
- **Tap to select:** Works via synthesized mouse events. No changes.
- **Touch targets:** Bump `.tb-btn` from `1.4rem` to `2.2rem` on mobile.
- **No new gestures.** Tap tabs to switch. Keep it simple.

## Files Changed

| File | Change |
|------|--------|
| `style.css` | `@media (max-width: 768px)` block: header, controls, workspace, tabs, panels, popover, toolbar, touch targets |
| `index.html` | Add tab bar markup (2 buttons) inside `#workspace` before `#left-panel` |
| `main.js` | Tab switching logic (~15 lines): toggle `.active`, auto-switch on redaction select |
| `viewport.js` | Single-finger touch pan (~20 lines) |

No changes to canvas.js, solver.js, popover.js, state.js, dom.js, or backend code.
