# Starter Icon Library — Design

**Date:** 2026-07-07
**App:** CardSmith (`cardsmith/index.html`, single-file React 18 UMD, no build step)

## Problem

The icon set is limited to 16 game-themed single-path SVGs shown as one flat grid
in the properties panel. There is no way to browse icons by category, and inserting
an icon always drops a default `star` that must then be changed. Users want a richer,
organized starter library.

## Scope

Curated expansion to **~27 single-path SVG icons in 5 categories**, browsable and
insertable through a dedicated modal. No external dependencies (all paths inline,
`viewBox 0 0 24 24`, solid fill with `fill-rule: evenodd`).

### Icon set (27)

| Category | Icons |
|----------|-------|
| **Suits** | heart, spade, club, diamond |
| **Combat** | sword, shield, skull, bolt, axe*, arrow* |
| **Loot** | coin, gem, crown, key*, chest*, potion* |
| **Nature** | flame, drop, leaf, sun*, moon*, mountain* |
| **Symbols** | star, dice, target*, flag*, gear* |

`*` = 11 new paths added to `ICONS`. Existing icons keep their keys/paths unchanged.

## Data

- Add the 11 new path strings to the existing `ICONS` map.
- Add `ICON_CATS = { Suits:[...], Combat:[...], Loot:[...], Nature:[...], Symbols:[...] }`
  defining grouping and display order (keys reference `ICONS`).

## UI — Icon library modal (`modal:'iconlib'`)

- Category tabs across the top; grid of the active category's icons below.
- No search box (27 icons; categories suffice).
- **Two modes:**
  - **insert** — clicking an icon calls `addEl('icon',{icon})` (auto-selects the new
    element) and closes the modal.
  - **swap** — clicking an icon calls `updateSelEl({icon},{key:'icon'})` on the selected
    element and closes the modal.
- State: `iconLib:{cat}` for the active category; mode tracked via an opener
  (`openIconLib(mode)`), swap targets the currently selected element.

## Entry points (both)

- Toolbar **"Icon"** button → `openIconLib('insert')` (no longer inserts a default star).
- Properties panel of a selected icon element → replace the current flat icon grid
  (~line 920) with a **"Change from library"** button → `openIconLib('swap')`.

## Unchanged

- Export (PDF/PNG/JSON/batch), Series Generator (its icon variable reads all `ICONS`,
  now with more options), Assets (user-uploaded images) — untouched.

## Verification

- All 11 new paths pre-rendered to PNG (resvg) and visually confirmed recognizable
  before integration.
- After wiring: in-browser check that insert adds the chosen icon, swap changes the
  selected element, category tabs switch, and undo works (single `add` snapshot on insert).
