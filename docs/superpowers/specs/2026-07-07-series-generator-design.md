# Series Generator — Design

_Date: 2026-07-07 · Project: CardSmith (`index.html`, single-file React 18 UMD app)_

## Goal

Speed up prototyping: design one base card, then generate N cards in one shot,
varying chosen properties (a number/text, an icon, a color, the background)
across a range or list of values — without hand-authoring a CSV table.

## Motivation / relationship to existing "Batch generate"

CardSmith already has **Batch generate** (`batchGenerate`, `modal:'batch'`):
put `{{field}}` placeholders in text/image elements, paste a CSV/TSV table, one
row → one card. It has two gaps for this use case:

1. Substitution only reaches **text** and **image `src`** — it cannot vary an
   element's **icon** or **color**, nor the card background.
2. Every row must be typed by hand — there is no "numbers 1..10 in one shot".

The Series Generator is a **separate, guided** flow that fixes both: you pick an
element and give it a range or list of values, with proper pickers per type.
`batchGenerate` and the `{{field}}` flow are left untouched (they still serve the
paste-a-table workflow).

## Decisions (confirmed with user)

- **Guided, no placeholders.** Target elements are chosen from the base card,
  not marked with `{{field}}`.
- **Combination modes: both `zip` and `product`**, chosen with a toggle.
- **Values: range + list, with type-appropriate pickers** (number range/list,
  color swatches/hex, icon grid).

## Data model

New component state:

```js
series: {
  mode: 'zip' | 'product',
  vars: [
    { id, elId, prop, valueMode, from, to, step, list:[...] }
  ]
}
```

A **variable** binds a target on the base card to a set of values:

- `elId`: the id of a base-card element, or the literal `'bg'` (card background).
- `prop`: the concrete property key to write —
  `'text'` | `'icon'` | `'color'` | `'fill'`. For `elId==='bg'`, the generator
  writes `bg.color` (and forces `bg.type='solid'`); `prop` is unused/`'color'`.
- `valueMode`: `'range'` (numbers only) or `'list'`.
- `from`, `to`, `step`: numeric range bounds (used when `valueMode==='range'`).
- `list`: array of explicit values — numbers/strings (text), icon-name keys
  (icon), or hex color strings (color/fill/bg).

Initial state: `series:{ mode:'zip', vars:[] }`.

### Which props a base-card element offers

- text element → **Number/Text** (`prop:'text'`) and **Color** (`prop:'color'`)
- icon element → **Icon** (`prop:'icon'`) and **Color** (`prop:'color'`)
- shape element → **Fill color** (`prop:'fill'`)
- image element → (none in v1)
- card background → **Background color** (`elId:'bg'`)

### Element labels (for the picker)

Derived, since elements have no user names:
`Text · "Card Title"` (text snippet, truncated), `Icon · star` (icon name),
`Shape · rect` (shape kind), plus `Background`.

## Generation semantics

**`resolveVar(v)` → array of values:**

- text + range → `[from, from+step, … ≤ to]` as strings (guard: `step>0`,
  cap element count to avoid runaway).
- any prop + list → `v.list` as-is.
- icon/color/fill/bg are always `valueMode:'list'`.

**`seriesCount()` → N (live):**

- No vars → 0 (Generate disabled).
- `zip`: `N = max(len(resolveVar(v)))` over vars.
- `product`: `N = ∏ len(resolveVar(v))`.

**`seriesGenerate()`:**

- Guardrail: if `N === 0` toast and abort; if `N > 150` toast
  ("Too many cards (N) — narrow the ranges") and abort.
- One `pushHistory('series')` so a single undo reverts the whole run.
- For each card index `i` in `[0, N)`:
  - Determine each variable's value at `i`:
    - `zip`: `values[i % values.length]`.
    - `product`: standard mixed-radix decomposition of `i` across the vars'
      value-array lengths.
  - `nc = clone(base)`, `nc.id = uid()`, every element gets a fresh `uid()`,
    `nc.qty = 1`. **Name:** use the value at `i` of the first variable whose
    `prop==='text'` (converted to a string); if there is no text variable, fall
    back to `base.name + ' ' + (i+1)`.
  - Apply each variable: if `elId==='bg'`, set
    `nc.bg = {...nc.bg, type:'solid', color:value}`; else find the element by
    `elId` in `nc.elements` and set `el[prop] = value` (skip if the element is
    missing).
- Append all N cards to the deck, close the modal, toast
  `"Generated N cards"`.

## UI

New method `renderSeriesModal()`, shown when `state.modal==='series'` (mirrors
`renderModal`'s structure/styling for consistency). Opened via `openSeries()`
from a **"Generate series"** button placed next to the existing Batch button.

Layout:

```
┌─ Generate series ──────────────────────────────  [×] ┐
│ Base: "Number Card"          Mode: (•)Zip ( )Product │
│  Variables                                 [+ Add ▾]  │
│  ┌───────────────────────────────────────────────┐   │
│  │ ● Text "1"   ▸ Number  [range] 1 → 10  step 1 │[×]│
│  │   values: 1,2,3,4,5,6,7,8,9,10                 │   │
│  │ ● Icon       ▸ Icon    [grid picker]          │[×]│
│  │   values: star, heart, shield                  │   │
│  │ ● Background ▸ Color   [swatches]             │[×]│
│  │   values: #eee, #fce, #cef                     │   │
│  └───────────────────────────────────────────────┘   │
│  → 10 cards will be generated       [Cancel] [Generate]│
└───────────────────────────────────────────────────────┘
```

- **[+ Add ▾]** lists the base card's targetable elements (derived labels) plus
  "Background". Selecting one calls `addSeriesVar`. When an element offers more
  than one prop (text→Number/Color, icon→Icon/Color), a small inline selector
  chooses which; default to the primary prop (text/icon) so one click is enough.
- **Per-type value editor:**
  - number (`prop:'text'`): range inputs (from/to/step via `numInput`) with a
    range/list toggle; list is a comma-separated `textInput`.
  - icon (`prop:'icon'`): a grid of `this.ICONS` (rendered with `gIcon`);
    clicking toggles membership in `list` (ordered).
  - color (`prop:'color'|'fill'|'bg'`): swatch chips; add via `colorInput`,
    remove per chip.
- **Live count** from `seriesCount()`, plus resolved-value chips per variable.
- Reuses existing helpers: `numInput`, `textInput`, `colorInput`, `gIcon`,
  `uIcon`, `iconBtn`, and the modal chrome from `renderModal`.

## Scope (YAGNI)

**In:** targets text/icon/color/fill + background; range and list; zip and
product; live count; N guardrail; single-undo generation.

**Out:** shape **stroke** (fill only); per-field name templates (name comes from
the first text variable, else `base + index`); CSV import here (Batch already
covers it); per-card visual preview (value chips only); drag-reordering of
values; image `src` variation.

## Verification

Single-file app, no test harness. Verify in the browser:

1. Design a base card with a number text, an icon, and a background.
2. Add a Number variable range 1→10 step 1 → count shows 10.
3. Add an Icon variable with 3 icons and a Background color variable with 3
   colors; keep **Zip** → count stays 10; Generate → 10 cards appended,
   numbers 1–10, icons and colors cycling every 3.
4. Switch to **Product** with numbers 1→4 and 4 icons → count shows 16; Generate
   → 16 cards covering all combinations; each card is named after its number
   (the text variable's value).
5. Push a range that would exceed 150 (e.g. product 13×13 = 169) → blocked with a
   toast, nothing generated.
6. Undo once → the whole generated run disappears in one step.
7. Batch generate (`{{field}}` + CSV) still works unchanged; export/PNG/PDF and
   thumbnails unaffected.
