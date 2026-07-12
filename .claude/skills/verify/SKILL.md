---
name: verify
description: Drive CardSmith in a headless browser to verify changes end-to-end (serve, click, download, inspect PDFs)
---

# Verifying CardSmith

Single-file app (`index.html`, React 18 UMD + jsPDF via CDN — network required). No build, no tests; verification = drive the real UI.

## Launch

```bash
# serve (from repo root, background)
python -m http.server 8987
```

Browser: Playwright's cached Chromium works with `playwright-core` (no browser download):

```bash
SCRATCH=$(mktemp -d) && cd "$SCRATCH" && npm init -y && npm i playwright-core
```

```js
const { chromium } = require('playwright-core');
const browser = await chromium.launch({
  executablePath: 'C:/Users/Sanctuary/AppData/Local/ms-playwright/chromium-1228/chrome-win64/chrome.exe' // adjust to current cache dir
});
const ctx = await browser.newContext({ acceptDownloads: true, viewport: { width: 1600, height: 1000 } });
```

## Driving patterns that work

- Wait for app ready: `page.waitForSelector('text=Deck')`.
- Buttons by visible text: `page.getByRole('button', { name: 'PDF · Letter' })`, toolbar add-element buttons by `name: 'Text' | 'Icon' | ...`, Front/Back toggle by `name: 'Back', exact: true`.
- Save project: `page.locator('[title="Save project (JSON)"]')`. Load: `page.locator('input[type=file][accept*="json"]').setInputFiles(path)` — then expect toast `Project loaded`.
- Downloads (PDF/PNG/JSON): `Promise.all([page.waitForEvent('download'), button.click()])`, then `download.saveAs(...)`; check `suggestedFilename()`.
- New text elements auto-enter inline edit with placeholder selected — just `keyboard.type(...)` then `Escape`.
- Undo/redo: `Control+z` / `Control+y` on `page.keyboard`.
- Collect `page.on('pageerror')` + console errors; the only expected 404 is `/favicon.ico`.

## Inspecting exported PDFs (no viewer needed)

jsPDF output here is uncompressed unless a stream says FlateDecode (then `zlib.inflateSync`):

- Page count: count `/\/Type\s*\/Page[^s]/g` matches.
- Card placements: image ops look like `q W 0 0 H X Y cm /I0 Do Q` (pt units, y from bottom). Letter width = 215.9·72/25.4 pt.
- Duplex mirror check: for each front op `(w,x,y)` on page N, expect back op on page N+1 with same `w`,`y` and `x' ≈ pageW − x − w` (±0.6 pt).

## Flows worth driving

Sample deck is mixed-size (poker/tarot/mini, 13 printed) — good for export tests. Core smoke: flip Front/Back, edit both sides, undo across sides, PDF Letter with/without "Double-sided (card backs)", JSON save/reload round-trip, v1 file load (delete `back` + set `version:1` in a saved v2 file).
