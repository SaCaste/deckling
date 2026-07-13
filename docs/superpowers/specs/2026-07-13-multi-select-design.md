# Multi-Select — Design Spec

**Date:** 2026-07-13
**Status:** Approved by user (design conversation, this date)
**App:** CardSmith — single-file browser card maker (`index.html`, React 18 UMD class component, `h = React.createElement`, no build step, no test framework)

## Goal

Select multiple elements (shift-click and marquee), move them together with bounding-box snapping, and act on the group: align selection, distribute, duplicate, delete.

## Decisions (from brainstorming)

| Question | Decision |
|---|---|
| Selection UX | **Shift-click add/remove + marquee** (rubber-band drag on empty canvas) |
| Group operations | **Move, delete, duplicate, align-selection, distribute.** Style editing (color, font, opacity…) stays single-select |
| Group snapping | **Selection bounding box** snaps via the existing `snapMove` resolver; selected members are excluded as snap targets |
| Architecture | **Approach A — additive `selEls`**: keep `selEl` untouched for single selection; `selEls` holds ids only when 2+ are selected. Invariant: never both active (multi ⇒ `selEl:null`; single ⇒ `selEls:[]`) |

Conventional defaults (fixed, not open questions): marquee selects anything its rectangle **touches** (intersects); **distribute = equal gaps**, outer two elements fixed; align-selection aligns to the **selection's** bounding box; `Ctrl+A` selects all on the current side; `Escape` clears the selection; resize and rotate handles remain single-select only.

## 1. Selection state & mechanics

- New state: `selEls: []` (element ids, current side only), `marquee: null | {x0,y0,x1,y1}` (mm, trim space).
- **Invariant:** `selEls.length` is 0 or ≥ 2. Multi-selection sets `selEl:null, editEl:null`. Any code path that would leave exactly one id demotes it to `selEl` and empties `selEls`.
- **Shift-click** on an element:
  - nothing selected → behaves as plain click (`selEl:clicked`);
  - single `selEl` active and the **same** element shift-clicked → deselect (clear);
  - single `selEl` active and a different element shift-clicked → promote: `selEls=[selEl, clicked]`, `selEl:null`;
  - element already in `selEls` → remove it (if 1 remains → demote to `selEl`);
  - element not in `selEls` → add it.
- **Plain click** on an element: always demote to that single element (`selEl:id, selEls:[]`) — current behavior for single, escape hatch for multi.
- **Marquee:** pointerdown on the empty face starts drag mode `'marquee'` anchored at the pointer (mm). While moving, `state.marquee` updates. On release:
  - if the pointer never moved past a small threshold (< 3 screen px), treat as the existing click-to-deselect;
  - otherwise select every element whose layout box (x/y/w/h, ignoring rotation — documented approximation) intersects the rectangle: 0 hits → clear; 1 hit → `selEl`; 2+ → `selEls`.
- **Ctrl+A** (in `onKey`, outside inputs): select all elements of the current side (2+ → `selEls`; 1 → `selEl`; 0 → no-op). **Escape** (when not editing text): clear selection.
- Flipping sides clears the selection (extend `flipSide`'s existing `selEl:null` patch with `selEls:[]`).

## 2. Rendering

- Each member of `selEls` renders the existing thin accent outline (the `sel` border in `renderElDOM`) but **no resize/rotate handles** — handles remain exclusive to single selection. Guard: handles render only when `selEl===el.id`, which the invariant already guarantees; members are marked selected when `selEls.includes(el.id)`.
- A **dashed accent bounding box** around the union of member boxes, rendered in `renderFace` (interactive only), `pointerEvents:'none'`.
- **Marquee rectangle:** 1px dashed accent rect from `state.marquee`, interactive only, `pointerEvents:'none'`.

## 3. Group move + snapping

- Pointerdown on an element that is **in `selEls`** starts drag mode `'gmove'` (instead of `'move'`): caches per-member start positions `[{id,x,y}]`, the group bbox `{x,y,w,h}`, and snap targets built with **all members excluded** (a one-arg extension of `buildSnapTargets` to skip a set of ids).
- On pointer move: proposed bbox origin = start bbox + pointer delta; call the existing `snapMove(px, py, {w:bbox.w, h:bbox.h}, targets, 6/s)` (group box never rotated); the snapped delta applies to every member (`.toFixed(2)` on snapped axes, `.toFixed(1)` free — same convention as single). Guides flow through the same `extra:{guides}` channel.
- History: one `pushHistory('gmove')` at pointerdown; `hist:false` during moves; guides cleared on pointerup (existing).
- Pointerdown with **Shift** held on a member must NOT start a drag — it's a selection toggle (process toggle, no drag object).

## 4. Group operations

New methods, each one undo step, all operating on `curCard()` so they work on both sides:

- `selBBox()` — union of member layout boxes (helper).
- `moveSels(dx,dy)` — internal to gmove.
- `deleteSels()` — remove all members; clear selection. Key: `'gdel'`.
- `dupSels()` — clone members (+4/+4 mm offset, new ids), append in z-order, selection becomes the clones. Key: `'gdup'`.
- `alignSels(edge)` — `edge ∈ {l,ch,r,t,cm,b}`: set each member's x (or y) so the chosen edge/center aligns to the selection bbox's same line. Key: `'galign'`.
- `distributeSels(axis)` — axis `h`/`v`, requires ≥3 members: sort by box start on that axis, keep first and last fixed, equalize the gaps between consecutive boxes (gap may be negative when overlapping — allowed). Key: `'gdist'`.

**Keyboard routing in `onKey`:** when `selEls.length ≥ 2`, Delete/Backspace → `deleteSels()`, Ctrl+D → `dupSels()`. Single-select paths unchanged. Enter/F2 (inline edit) does nothing for multi (needs `curEl()`, which is null under the invariant).

## 5. Inspector multi-panel

When `selEls.length ≥ 2`, `renderInspector` returns a compact panel instead of the element inspector:

- Header: **"N elements selected"**.
- **Align selection** row: 6 icon buttons reusing the `alCL/alCH/alCR/alCT/alCM/alCB` icons → `alignSels(...)`.
- **Distribute** row: 2 buttons "Distribute H" / "Distribute V" (new `distH`/`distV` UICONS glyphs), disabled below 3 members.
- Action row: duplicate (`dupSels`) and delete (`deleteSels`) icon buttons.
- The single-select "Align to card" row and element properties are untouched.

## 6. Integration & edge cases

- `fixSel` (runs after undo/redo): prune `selEls` ids missing from the current side's card; if < 2 remain, demote/clear per the invariant.
- `updateSelEl`, `curEl`, inline editing, single-drag snapping, align-to-card: untouched (they key off `selEl`, null during multi).
- Batch/Series/deck panel: unaffected (`selEls` never consulted; front-only guards already in place). `loadJSON` resets `selEls:[]` and `marquee:null` alongside its existing selection reset.
- Marquee on a card with zero elements: allowed, selects nothing.
- `dupSels` of elements near the card edge may push clones past the edge — allowed (consistent with single `dupEl`).
- Distribute with members of equal positions: sort is stable by current z-order; result deterministic.
- Saved project JSON is unchanged (selection is session state; v2 format untouched).

## 7. Verification (headless browser via `.claude/skills/verify/SKILL.md` — mind the locator trap and px/mm calibration notes)

1. Shift-click two elements → both outlined + dashed group bbox; shift-click one again → back to single selection.
2. Marquee over 2 of 3 elements → exactly those selected; marquee over empty area → selection cleared; plain click on empty face still deselects.
3. Group drag: both members move by the same delta (saved-JSON assertion); with snap on, group bbox left edge snaps to card center line exactly; guides visible; Alt bypasses.
4. Align selection L: all members' left edges equal the bbox left (exact); single Ctrl+Z reverts all.
5. Distribute H with 3 elements: middle element's gaps equal (±0.05mm); outer two unmoved.
6. Group duplicate: N clones offset +4/+4, selection on clones; group delete removes all; one undo restores.
7. Ctrl+A selects all; Escape clears.
8. Repeat checks 1 and 3 on the card Back.
9. Single-select regression: inline edit, resize handles, align-to-card row all behave as before.
10. Console clean.

## Out of scope

- Group styling (shared color/font/opacity editing).
- Group resize/rotate.
- Persistent groups (grouping as a document concept).
- Marquee hit-testing against rotated true bounds (layout-box approximation documented above).
