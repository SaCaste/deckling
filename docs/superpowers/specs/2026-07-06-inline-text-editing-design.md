# Inline text editing on the canvas — Design

_Date: 2026-07-06 · Project: CardSmith (`index.html`, single-file React 18 UMD app)_

## Goal

Let the user edit a text element directly on the canvas by double-clicking it,
instead of only through the Inspector's textarea. Editing is WYSIWYG: the text
is edited in place, with the element's real typography.

## Scope

**In scope**

- Double-click a text element on the interactive canvas to enter inline edit.
- Edit in place via a transparent `<textarea>` overlay matching the element's
  typography; changes flow through the existing `updateSelEl` path.
- Exit with Escape, blur, or clicking empty canvas. Exit keeps changes.
- One undo entry reverts the whole edit session.
- Keyboard/drag guards so typing never triggers delete/duplicate/move.

**Out of scope (YAGNI)**

- Entering edit via Enter key or auto-editing a newly added text element
  (explicitly declined: trigger is double-click only).
- Rich text (bold/italic runs, mixed colors) — text stays a plain string.
- Auto-resizing the element's bounding box to fit content.
- Inline editing of non-text element types.

## Decisions (confirmed with user)

- **Trigger:** double-click only.
- **Escape:** confirms and exits (changes kept). Undo (⌘Z) reverts the edit.
- **Mechanism:** transparent `<textarea>` overlay (approach A), chosen over
  `contentEditable` because the data model is a plain string with `pre-wrap`;
  a textarea handles newlines natively and reuses the controlled data flow,
  avoiding `contentEditable`'s plain-text/newline extraction pitfalls.

## Current architecture (relevant pieces)

- `renderElDOM(el, s, interactive)` (~line 992): draws each element. Text is a
  `<div>` with `pointerEvents:'none'` showing `el.text` (plain string,
  `whiteSpace:'pre-wrap'`).
- Inspector textarea (~line 807): the only current text editor →
  `updateSelEl({text}, {key:'text'})`.
- `onElPointerDown(e, el)` (~line 385): starts a move drag on pointer down.
- `onKey(e)` (~line 143): global shortcuts (undo/redo/dup/delete); already
  early-returns when `e.target.tagName` is `INPUT` or `TEXTAREA`.
- `fixSel()` (~line 181): reconciles selection when cards/elements change.
- `renderFace(...)` (~line 1032): the `onPointerDown` on empty face clears
  `selEl`.
- History: `pushHistory(key)` (~line 161) dedupes repeats of the same key
  within 700 ms.

## Design

### 1. State

Add `editEl` to component state: the id of the text element being edited, or
`null`. At most one element edits at a time. Entering edit ensures
`selCard`/`selEl` point at that element.

### 2. Enter edit

In `renderElDOM`, the text `<div>` gets `onDoubleClick → enterEdit(el)`:

```
enterEdit(el){
  if(el.type!=='text') return;
  this.pushHistory('text');   // single undo entry for the whole edit
  this.setState({ selCard:<its card>, selEl:el.id, editEl:el.id });
}
```

`pushHistory('text')` reuses the `'text'` key, so it does not create a
duplicate history entry if the user was already typing in the Inspector.

### 3. Edit mechanism (approach A)

In `renderElDOM`, when `interactive && el.type==='text' && editEl===el.id`, the
`inner` node becomes a `<textarea>` instead of the display `<div>`, with the
**same** typography styles the div uses today (`fontFamily`, `fontSize`
`el.size*s`, `fontWeight`, `fontStyle`, `color`, `textAlign`,
`lineHeight`, `letterSpacing`), plus:

```
background:transparent; border:0; padding:0; margin:0; resize:none;
width:100%; height:100%; overflow:hidden; pointerEvents:'auto'; display:'block';
```

- `value: el.text`
- `onChange: e => updateSelEl({ text:e.target.value }, { key:'text' })`
- On mount (ref callback): focus and place caret at end
  (`n.focus(); n.setSelectionRange(len, len);`).
- While editing, the resize/rotate handles are **not** rendered (so they don't
  block typing); the selection border is still drawn.
- The wrap `cursor` becomes `text` for the element being edited.

### 4. Exit edit

- **Escape** and **blur** on the textarea → `setState({ editEl:null })`. Changes
  are already saved (controlled value). Escape calls `e.stopPropagation()` so it
  doesn't collide with global handling.
- Clicking empty canvas (`renderFace` `onPointerDown`, which already clears
  `selEl`) also clears `editEl`.
- Enter does **not** exit: it inserts a newline (native textarea behavior,
  consistent with `pre-wrap`).

### 5. Keyboard & drag guards

- `onKey` already early-returns for `TEXTAREA`, so ⌘D / Delete / ⌘Z do not fire
  while focus is in the textarea. No change needed; verify focus really lands in
  the textarea.
- In `onElPointerDown`, if `this.state.editEl===el.id`, `return` early (without
  `stopPropagation`) so the pointer places the caret instead of starting a drag.
  Double-click still fires `onDoubleClick`.

### 6. Edge cases

- If the edited element is deleted or the card changes, `fixSel` must clear
  `editEl` alongside `selEl`.
- Export (PDF/PNG) and deck thumbnails render with `interactive=false`, so they
  never produce a textarea — no change there.
- Only one global `editEl`: double-clicking another text replaces the previous
  edit target.

## Verification

Single-file app with no test harness → verify in the browser:

1. Double-click a text element → typing edits in place, multi-line works.
2. Escape (and blur, and empty-canvas click) exit edit, keeping changes.
3. ⌘Z reverts the whole edit in one step.
4. Delete/Backspace and ⌘D do **not** affect the element while editing.
5. Pointer-down while editing places the caret; it does not start a drag.
6. Export/thumbnails unaffected.

Optionally capture screenshots of the before/edit/after states.
