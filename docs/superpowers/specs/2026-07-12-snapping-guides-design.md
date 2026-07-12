# Snapping & Alignment Guides — Design Spec

**Date:** 2026-07-12
**Status:** Approved by user (design conversation, this date)
**App:** CardSmith — single-file browser card maker (`index.html`, React 18 UMD class component, `h = React.createElement`, no build step, no test framework)

## Goal

Smart guides while dragging elements — snap to card geometry and to other elements, with visible guide lines — plus a six-button "align to card" row in the inspector.

## Decisions (from brainstorming)

| Question | Decision |
|---|---|
| Snap targets | Card geometry (trim edges, center lines, safe-zone insets) **and** other elements' edges/centers |
| When | Move-drag only. Resize and rotate stay free (may extend later) |
| Alignment controls | Six align-to-card buttons for the selected element (L / center-H / R, T / middle-V / B). Distribute and multi-align wait for multi-select (explicitly out of scope) |
| Escape hatch | Hold **Alt** during drag to bypass; persistent **Snap** toggle chip in the canvas top bar |
| Architecture | **Approach A**: pure snap resolver + target lines cached at drag start, applied in `onPointerMove`'s move branch (not inside `updateSelEl` — inspector numeric input must never snap; not DOM-measured) |

## 1. Snap model

- **Target lines** are computed once per drag, at drag start (`onElPointerDown`), from `curCard()` — so the back side snaps against back elements automatically. All values in mm, trim space (same space as `el.x/y`):
  - Vertical: card `0`, `c.w/2`, `c.w`; safe insets `SAFE` and `c.w−SAFE`; for every *other* element: `left`, `centerX`, `right`.
  - Horizontal: card `0`, `c.h/2`, `c.h`; `SAFE` and `c.h−SAFE`; per other element: `top`, `centerY`, `bottom`.
- **Rotated elements** (rotation ≠ 0) contribute **center lines only** as targets — their layout-box edges don't match what's on screen. Likewise, a rotated *dragged* element snaps by its center only.
- **Candidate lines of the dragged element** (unrotated): left/centerX/right against vertical targets; top/centerY/bottom against horizontal targets. Each axis resolves independently; the smallest |delta| within tolerance wins; ties broken by target-list order (card lines first, then other elements in z-order).
- **Tolerance:** 6 *screen* pixels converted to mm (`6 / this.scale`) — identical feel at every zoom level.
- **Bypass:** no snapping when `!this.state.snapOn` or `e.altKey` is held during the move.
- The resolver is a **pure method** `snapMove(px, py, el, targets, tol)` → `{ x, y, guides }` (guides = array of `{axis:'v'|'h', pos}` in trim-space mm). No state reads/writes inside — unit-testable in isolation.

## 2. State & UI

- `state.snapOn: true` — editor preference; **not** persisted in project JSON, not in undo history.
- **Snap toggle chip** in the canvas floating top bar next to Bleed/Safe, reusing the existing `toggle()` helper pattern in `renderCanvas`.
- `state.guides: []` — active guide lines during a drag. Updated in the same `setState` as the element move: `updateSelEl` gains an optional `opts.extra` object merged into `setCur`'s extras (needed because the app's `window`-level pointermove handler runs outside React's event batching in UMD legacy mode; two setStates per move would double-render).
- Guides cleared (`guides: []`) in `onPointerUp` (only when non-empty, to avoid a redundant render on every click).
- **Rendering:** in `renderFace`, interactive mode only — for each guide, one absolutely positioned 1px line at `(pos + B) * s`, spanning the full face (vertical guides full height, horizontal full width), solid pink `#ff4d94`, `pointerEvents:'none'`, `zIndex` above elements. Distinct from the dashed red bleed and dashed blue safe outlines.

## 3. Drag integration

In `onPointerMove`'s `move` branch:

1. Compute proposed `x/y` from pointer delta (existing math).
2. If snapping active, call `snapMove` with the targets cached in `this.drag.targets`; use returned `x/y/guides`. Otherwise `guides = []`.
3. Apply via `updateSelEl({x, y}, { hist:false, extra:{ guides } })` — same single-write path as today (snapped values replace the raw `.toFixed(1)` values; snapped coordinates land exactly on the target line, not rounded away).

`onElPointerDown` additionally stores `this.drag.targets = buildSnapTargets()` (card + other elements of the current side). Resize (`onHandleDown`) and rotate paths are untouched.

## 4. Align-to-card buttons

- New inspector row **"Align to card"** with six icon buttons: left, center-H, right, top, middle-V, bottom.
- Pure layout-box math on the selected element: `x=0`, `x=(c.w−el.w)/2`, `x=c.w−el.w`; `y=0`, `y=(c.h−el.h)/2`, `y=c.h−el.h`. Values `.toFixed(1)`-rounded like all other writes.
- Applied via `updateSelEl(patch, { key:'align' })` → one undo step, works on both card sides, rotation-safe (rotation transforms around the layout-box center, which these buttons position).
- Placement: in `renderInspector`, above the existing layer/duplicate/delete action row.

## 5. Edge cases

- Elements partially or fully off-card still snap (target lines may lie outside `0..w` — allowed).
- Zero other elements → card-geometry targets only.
- Snap toggle state survives flips and project loads (it's session UI state; a fresh page load resets it to on).
- Guides never appear in exports (they live only in the interactive `renderFace` path, like bleed/safe outlines).
- Very small elements (4mm min) — center + both edges can all be within tolerance simultaneously; smallest-delta rule resolves this deterministically.

## 6. Verification (headless browser via `.claude/skills/verify/SKILL.md`)

1. Drag an element near the card's horizontal center → released `x` puts element center exactly at `c.w/2` (±0.05mm, read from inspector W/X or saved JSON); pink vertical guide visible in a mid-drag screenshot.
2. Drag near another element's left edge → snaps flush; guide shown.
3. Alt-drag through the same spot → no snap, no guide.
4. Toggle Snap off → no snap; toggle back on → snaps again.
5. Flip to Back → dragging the star snaps to the back card's center lines.
6. Each of the six align buttons → exact expected x/y for a known element; single Ctrl+Z reverts.
7. Resize a corner handle across a snap line → no snapping (scope check).
8. Console clean throughout.

## Out of scope

- Multi-select, align-selection, distribute (future feature).
- Resize/rotate snapping.
- Equal-spacing "distribute hints" between elements.
- Grid snapping.
