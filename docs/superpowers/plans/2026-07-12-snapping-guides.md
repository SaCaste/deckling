# Snapping & Alignment Guides Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Smart guides while dragging elements — snap to card geometry and other elements with visible pink guide lines — plus a six-button align-to-card row in the inspector.

**Architecture:** Pure snap resolver (`snapMove`) + target lines cached at drag start (`buildSnapTargets`, stored on `this.drag`), applied only in `onPointerMove`'s move branch so inspector inputs and resize never snap. Guides ride in the same `setState` as the move via a new `opts.extra` passthrough on `updateSelEl` → `setCur`. Guide lines render in `renderFace` (interactive only), like the bleed/safe overlays.

**Tech Stack:** Single-file app `index.html` — React 18 UMD class component (`h = React.createElement`), no build step, **no test framework** (verification = headless-browser drive per `.claude/skills/verify/SKILL.md`).

**Spec:** `docs/superpowers/specs/2026-07-12-snapping-guides-design.md`

---

## File Structure

All changes in `index.html` (project convention — do not split):

- Task 1: state block (~line 131), `updateSelEl` (~line 242), new methods near the pointer handlers (~line 440), `onElPointerDown`/`onPointerMove`/`onPointerUp` (~lines 440–485)
- Task 2: `renderFace` interactive block (~line 1231), `renderCanvas` top bar (~line 1260)
- Task 3: `UICONS` map (~line 95), `renderInspector` (~line 958)
- Task 4: verification only (no product code)

Line numbers are anchors — verify against the stated context; content match wins.

**Working conventions for every task:** run on branch `snapping-guides` (create from `main` in Task 1 Step 1). Serve with `python -m http.server 8987` from the repo root for any browser checking. After each code edit, syntax-check the inline script:

```bash
node -e "const s=require('fs').readFileSync('index.html','utf8'); const m=s.match(/<script>([\s\S]*?)<\/script>/); require('fs').writeFileSync('_check.js', m[1]);" && node --check _check.js && rm _check.js
```

---

### Task 1: Snap engine + drag integration

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Create the branch**

```bash
git checkout -b snapping-guides main
```

- [ ] **Step 2: Add `snapOn` and `guides` to initial state**

At ~line 132 the state block contains:

```js
    showBleed:true, showSafe:true, zoom:1,
```

Change to:

```js
    showBleed:true, showSafe:true, zoom:1,
    snapOn:true, guides:[],
```

- [ ] **Step 3: `updateSelEl` passes `opts.extra` through to `setCur`**

At ~line 242, current:

```js
  updateSelEl(patch, opts){
    opts=opts||{};
    const c=this.curCard(); if(!c) return;
    if(opts.hist!==false) this.pushHistory(opts.key||'edit');
    this.setCur({ ...c, elements:c.elements.map(e=> e.id!==this.state.selEl ? e : {...e,...patch}) });
  }
```

Change ONLY the `setCur` line to:

```js
    this.setCur({ ...c, elements:c.elements.map(e=> e.id!==this.state.selEl ? e : {...e,...patch}) }, opts.extra);
  }
```

(`setCur(next, extra)` at ~line 190 already spreads `...(extra||{})` — undefined is safe.)

- [ ] **Step 4: Add `buildSnapTargets` and `snapMove`**

Insert directly above `onElPointerDown` (~line 440):

```js
  // Snap target lines (mm, trim space) for the element being dragged.
  // Order matters for tie-breaks: card lines first, then elements in z-order.
  buildSnapTargets(el){
    const c=this.curCard(); if(!c) return {v:[],h:[]};
    const v=[0, c.w/2, c.w, this.SAFE, c.w-this.SAFE];
    const hz=[0, c.h/2, c.h, this.SAFE, c.h-this.SAFE];
    c.elements.forEach(o=>{
      if(o.id===el.id) return;
      if(o.rotation){ v.push(o.x+o.w/2); hz.push(o.y+o.h/2); return; } // rotated: center only
      v.push(o.x, o.x+o.w/2, o.x+o.w);
      hz.push(o.y, o.y+o.h/2, o.y+o.h);
    });
    return { v, h:hz };
  }
  // Pure resolver: propose position (px,py) for a box {w,h,rotation}; snap each
  // axis to the nearest target line within tol (mm). Strict < keeps the first
  // (earliest-listed) line on ties. Returns snapped x/y + active guide lines.
  snapMove(px, py, box, targets, tol){
    const rot=!!box.rotation;
    const candX = rot ? [box.w/2] : [0, box.w/2, box.w];
    const candY = rot ? [box.h/2] : [0, box.h/2, box.h];
    const solve=(p, cands, lines)=>{
      let best=null;
      lines.forEach(L=>cands.forEach(off=>{
        const d=L-(p+off);
        if(Math.abs(d)<=tol && (!best || Math.abs(d)<Math.abs(best.d))) best={d, line:L};
      }));
      return best;
    };
    const bx=solve(px,candX,targets.v), by=solve(py,candY,targets.h);
    const guides=[];
    if(bx) guides.push({axis:'v', pos:bx.line});
    if(by) guides.push({axis:'h', pos:by.line});
    return { x: bx? +(px+bx.d).toFixed(2) : +px.toFixed(1),
             y: by? +(py+by.d).toFixed(2) : +py.toFixed(1), guides };
  }
```

Note: snapped coordinates keep 2 decimals so the element lands *exactly* on the line (e.g. safe inset 3.5 minus half-width can produce `.25` values); unsnapped axes keep the app's existing 1-decimal rounding.

- [ ] **Step 5: Cache targets at drag start**

`onElPointerDown` (~line 440) currently ends with:

```js
    this.drag={ mode:'move', sx:e.clientX, sy:e.clientY, ox:el.x, oy:el.y };
```

Replace with:

```js
    this.drag={ mode:'move', sx:e.clientX, sy:e.clientY, ox:el.x, oy:el.y,
      box:{w:el.w, h:el.h, rotation:el.rotation||0}, targets:this.buildSnapTargets(el) };
```

- [ ] **Step 6: Snap in the move branch, clear guides on pointer-up**

`onPointerMove`'s move branch (~line 465) currently:

```js
    if(d.mode==='move'){
      const dx=(e.clientX-d.sx)/s, dy=(e.clientY-d.sy)/s;
      this.updateSelEl({ x:+(d.ox+dx).toFixed(1), y:+(d.oy+dy).toFixed(1) }, {hist:false});
    } else if(d.mode==='resize'){
```

Replace the move branch with:

```js
    if(d.mode==='move'){
      const dx=(e.clientX-d.sx)/s, dy=(e.clientY-d.sy)/s;
      let r={ x:+(d.ox+dx).toFixed(1), y:+(d.oy+dy).toFixed(1), guides:[] };
      if(this.state.snapOn && !e.altKey && d.targets)
        r=this.snapMove(d.ox+dx, d.oy+dy, d.box, d.targets, 6/s);
      this.updateSelEl({ x:r.x, y:r.y }, {hist:false, extra:{guides:r.guides}});
    } else if(d.mode==='resize'){
```

(`6/s` = 6 screen px in mm; `extra` is always sent so guides clear the moment you leave tolerance.)

`onPointerUp` (~line 485) currently:

```js
  onPointerUp(){ this.drag=null; }
```

Replace with:

```js
  onPointerUp(){ this.drag=null; if(this.state.guides.length) this.setState({guides:[]}); }
```

- [ ] **Step 7: Verify (node)**

Syntax-check (command in the header). Then unit-check the pure resolver logic by pasting `buildSnapTargets`-shaped data through a standalone copy of `snapMove`:

```bash
node -e "
const snapMove=(px,py,box,targets,tol)=>{const rot=!!box.rotation;const candX=rot?[box.w/2]:[0,box.w/2,box.w];const candY=rot?[box.h/2]:[0,box.h/2,box.h];const solve=(p,cands,lines)=>{let best=null;lines.forEach(L=>cands.forEach(off=>{const d=L-(p+off);if(Math.abs(d)<=tol&&(!best||Math.abs(d)<Math.abs(best.d)))best={d,line:L};}));return best;};const bx=solve(px,candX,targets.v),by=solve(py,candY,targets.h);const guides=[];if(bx)guides.push({axis:'v',pos:bx.line});if(by)guides.push({axis:'h',pos:by.line});return{x:bx?+(px+bx.d).toFixed(2):+px.toFixed(1),y:by?+(py+by.d).toFixed(2):+py.toFixed(1),guides};};
const T={v:[0,31.5,63,3.5,59.5,10,19,28],h:[0,44,88,3.5,84.5]};
// 1: center snap — box 18 wide proposed at 22.2 => center 31.2, within 0.5 of 31.5 => x=22.5
let r=snapMove(22.2,50,{w:18,h:18},T,0.5); console.log('center:',r.x===22.5&&r.guides.some(g=>g.axis==='v'&&g.pos===31.5)?'OK':'FAIL '+JSON.stringify(r));
// 2: edge snap to another element's left (10) — proposed x 9.7 => left within 0.5 => x=10
r=snapMove(9.7,50,{w:18,h:18},T,0.5); console.log('edge:',r.x===10?'OK':'FAIL '+JSON.stringify(r));
// 3: out of tolerance — x unchanged (1-decimal rounded)
r=snapMove(24.83,50,{w:18,h:18},T,0.5); console.log('free:',r.x===24.8&&r.guides.length===0?'OK':'FAIL '+JSON.stringify(r));
// 4a: rotated box — edge candidates suppressed. Use a target set WITHOUT line 19 (T's line 19 sits 0.3 from the rotated center 18.7 and legitimately center-snaps — bad test data in v1 of this plan):
const T2={v:[0,31.5,63,3.5,59.5,10,28],h:[0,44,88,3.5,84.5]};
r=snapMove(9.7,50,{w:18,h:18,rotation:30},T2,0.5); console.log('rot-no-edge-snap:',r.x===9.7&&r.guides.length===0?'OK':'FAIL '+JSON.stringify(r));
// 4b: rotated box still snaps BY CENTER: center 31.2 -> card line 31.5 -> x=22.5
r=snapMove(22.2,50,{w:18,h:18,rotation:30},T2,0.5); console.log('rot-center-snap:',r.x===22.5&&r.guides[0].pos===31.5?'OK':'FAIL '+JSON.stringify(r));
"
```

Expected: `center: OK`, `edge: OK`, `free: OK`, `rot-no-edge-snap: OK`, `rot-center-snap: OK`.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Snap engine: pure snapMove resolver, drag-start target cache, move-branch integration"
```

---

### Task 2: Guide rendering + Snap toggle

**Files:**
- Modify: `index.html` (`renderFace` ~line 1231, `renderCanvas` top bar ~line 1260)

- [ ] **Step 1: Render guide lines in `renderFace`**

The interactive block (~line 1231) currently:

```js
    if(interactive){
      if(this.state.showBleed) children.push(h('div',{key:'trim',style:{position:'absolute',left:B*s,top:B*s,width:card.w*s,height:card.h*s,outline:'1px dashed rgba(220,60,60,.6)',pointerEvents:'none'}}));
      if(this.state.showSafe) children.push(h('div',{key:'safe',style:{position:'absolute',left:(B+this.SAFE)*s,top:(B+this.SAFE)*s,width:(card.w-2*this.SAFE)*s,height:(card.h-2*this.SAFE)*s,outline:'1px dashed rgba(60,130,250,.5)',pointerEvents:'none'}}));
    }
```

Add guide rendering inside the same `if(interactive){` block, after the safe line:

```js
      this.state.guides.forEach((g,i)=>{
        children.push(h('div',{key:'gd'+i,style: g.axis==='v'
          ? {position:'absolute',left:(g.pos+B)*s-0.5,top:0,width:1,height:ph,background:'#ff4d94',pointerEvents:'none',zIndex:5}
          : {position:'absolute',top:(g.pos+B)*s-0.5,left:0,height:1,width:pw,background:'#ff4d94',pointerEvents:'none',zIndex:5}}));
      });
```

(`pw`/`ph`/`B`/`s` are already in scope in `renderFace`. Guides live only in interactive mode — never in thumbnails or exports.)

- [ ] **Step 2: Add the Snap toggle chip to the canvas top bar**

In `renderCanvas`'s top bar (~line 1260):

```js
        toggle('showBleed','Bleed','rgba(220,60,60,.8)'),
        toggle('showSafe','Safe','rgba(60,130,250,.8)'),
```

Change to:

```js
        toggle('showBleed','Bleed','rgba(220,60,60,.8)'),
        toggle('showSafe','Safe','rgba(60,130,250,.8)'),
        toggle('snapOn','Snap','rgba(255,77,148,.8)'),
```

(The existing `toggle(key,label,col)` helper flips `state[key]` — no new code needed. `snapOn` is not saved in project JSON and not in undo history, per spec §2 — nothing further to do, since `saveJSON` and `pushHistory` list fields explicitly.)

- [ ] **Step 3: Verify in browser (quick manual/headless)**

Serve, open, drag an element slowly across the card center: a vertical pink line must appear when it snaps and vanish when past tolerance; Alt-drag shows no line and no stickiness; clicking the Snap chip off disables snapping (chip dims like Bleed/Safe when off). Full checklist is Task 4 — this is a sanity pass. Console must be clean.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Snap guides UI: pink guide lines in renderFace, Snap toggle chip in canvas top bar"
```

---

### Task 3: Align-to-card buttons

**Files:**
- Modify: `index.html` (`UICONS` ~line 95, `renderInspector` ~line 958)

- [ ] **Step 1: Add six align icons to `UICONS`**

`UICONS` (~line 95) is a map of 24-unit stroke paths. Add these entries after the existing `star:` entry (keep the trailing-comma style consistent — add a comma to the current last entry):

```js
    alCL:'M4 3v18M8 8h10v8H8z',
    alCH:'M12 3v18M6 8h12v8H6z',
    alCR:'M20 3v18M6 8h10v8H6z',
    alCT:'M3 4h18M8 8h8v10H8z',
    alCM:'M3 12h18M8 6h8v12H8z',
    alCB:'M3 20h18M8 6h8v10H8z'
```

- [ ] **Step 2: Add the "Align to card" row to `renderInspector`**

In `renderInspector` (~line 958), locate the final action row push (search for `key:'act'`):

```js
    rows.push(h('div',{key:'act',style:{display:'flex',gap:6,marginTop:4,paddingTop:12,borderTop:'1px solid '+t.line}},
```

Insert IMMEDIATELY BEFORE that `rows.push(...)` statement:

```js
    const cc=this.curCard();
    if(cc){
      const alignBtn=(icon,title,patch)=>this.iconBtn(icon,()=>this.updateSelEl(patch(),{key:'align'}),{title});
      rows.push(this.field('Align to card', h('div',{key:'alg',style:{display:'flex',gap:6}},
        alignBtn('alCL','Align left',()=>({x:0})),
        alignBtn('alCH','Center horizontally',()=>({x:+((cc.w-el.w)/2).toFixed(1)})),
        alignBtn('alCR','Align right',()=>({x:+(cc.w-el.w).toFixed(1)})),
        h('div',{style:{width:1,background:t.line}}),
        alignBtn('alCT','Align top',()=>({y:0})),
        alignBtn('alCM','Center vertically',()=>({y:+((cc.h-el.h)/2).toFixed(1)})),
        alignBtn('alCB','Align bottom',()=>({y:+(cc.h-el.h).toFixed(1)})))));
    }
```

Notes: `el` and `t` are already in scope; `this.field(label, control)` and `this.iconBtn(name, onClick, opts)` are existing helpers (see their other uses in the same function). Patches are lazy closures so they read `el.w/el.h` at click time. History key `'align'` gives one undo step per click (coalesced within 700ms by `pushHistory` — same behavior as other inspector edits). Works on both sides because `updateSelEl` → `setCur` is side-aware.

- [ ] **Step 3: Verify (browser sanity)**

Serve, select an element, click each of the six buttons: element jumps to exact edge/center; one Ctrl+Z reverts the last click; buttons behave the same on the Back side. Console clean. Syntax check passes.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Align-to-card: six-button inspector row (L/C/R, T/M/B) with align icons"
```

---

### Task 4: Headless-browser verification (spec §6)

**Files:** none (verification only). Uses `.claude/skills/verify/SKILL.md` recipe: `python -m http.server 8987` + `playwright-core` with the cached Chromium (`C:/Users/Sanctuary/AppData/Local/ms-playwright/chromium-*/chrome-win64/chrome.exe`).

- [ ] **Step 1: Write and run the drive script**

Create `drive-snap.js` in a scratch dir with `npm i playwright-core`, then run it. Full script:

```js
const { chromium } = require('playwright-core');
const fs = require('fs'), path = require('path');
const EXE = 'C:/Users/Sanctuary/AppData/Local/ms-playwright/chromium-1228/chrome-win64/chrome.exe'; // adjust to current cache
const OUT = path.join(__dirname, 'evidence-snap'); fs.mkdirSync(OUT, { recursive: true });

(async () => {
  const browser = await chromium.launch({ executablePath: EXE });
  const ctx = await browser.newContext({ acceptDownloads: true, viewport: { width: 1600, height: 1000 } });
  const page = await ctx.newPage();
  const errors = [];
  page.on('pageerror', e => errors.push(e.message));
  page.on('console', m => { if (m.type() === 'error' && !m.text().includes('favicon')) errors.push(m.text()); });
  await page.goto('http://localhost:8987/index.html', { waitUntil: 'networkidle' });
  await page.waitForSelector('text=Deck');

  // helper: save project JSON and return parsed object
  const saveProj = async () => {
    const [dl] = await Promise.all([page.waitForEvent('download'), page.locator('[title="Save project (JSON)"]').click()]);
    const p = path.join(OUT, 'p.json'); await dl.saveAs(p);
    return JSON.parse(fs.readFileSync(p, 'utf8'));
  };
  // helper: drag from a locator's center by (dx,dy) screen px, screenshot mid-drag
  const dragBy = async (loc, dx, dy, shotName, alt) => {
    const b = await loc.boundingBox();
    const sx = b.x + b.width / 2, sy = b.y + b.height / 2;
    if (alt) await page.keyboard.down('Alt');
    await page.mouse.move(sx, sy); await page.mouse.down();
    await page.mouse.move(sx + dx, sy + dy, { steps: 8 });
    if (shotName) await page.screenshot({ path: path.join(OUT, shotName + '.png') });
    await page.mouse.up();
    if (alt) await page.keyboard.up('Alt');
  };

  // Setup: add a text element (auto-centered, auto-edit), rename to SNAPME
  await page.getByRole('button', { name: 'Text', exact: true }).click();
  await page.waitForTimeout(300); await page.keyboard.type('SNAPME'); await page.keyboard.press('Escape');
  const el = page.getByText('SNAPME').first();

  // 1: drag near horizontal center -> snaps to exact center + guide visible mid-drag
  await dragBy(el, 40, 25, null);           // move it clearly off-center first
  await dragBy(el, -37, 0, '01-snap-guide'); // come back to ~3px left of center: inside 6px tolerance
  let proj = await saveProj();
  const card = proj.cards[0];
  const sn = () => card.elements.find(x => x.text === 'SNAPME') || proj.cards[0].elements.find(x => x.text === 'SNAPME');
  let e1 = sn();
  const centered = Math.abs((e1.x + e1.w / 2) - card.w / 2) <= 0.05;
  console.log('CHECK snap-to-center:', centered ? 'OK' : 'FAIL x=' + e1.x + ' w=' + e1.w + ' cw=' + card.w);

  // 2: Alt-drag through the same spot -> NOT centered
  await dragBy(el, 40, 0, null);
  await dragBy(el, -37, 0, '02-alt-drag', true);
  proj = await saveProj(); e1 = proj.cards[0].elements.find(x => x.text === 'SNAPME');
  const altFree = Math.abs((e1.x + e1.w / 2) - proj.cards[0].w / 2) > 0.05;
  console.log('CHECK alt bypasses snap:', altFree ? 'OK' : 'FAIL');

  // 3: Snap toggle off -> free; on -> snaps again
  await page.getByRole('button', { name: 'Snap' }).click();
  await dragBy(el, 37, 0, null); await dragBy(el, -37, 0, null);
  proj = await saveProj(); e1 = proj.cards[0].elements.find(x => x.text === 'SNAPME');
  const toggleFree = Math.abs((e1.x + e1.w / 2) - proj.cards[0].w / 2) > 0.05;
  console.log('CHECK toggle-off disables snap:', toggleFree ? 'OK' : 'FAIL');
  await page.getByRole('button', { name: 'Snap' }).click();
  await dragBy(el, 3, 0, null);
  proj = await saveProj(); e1 = proj.cards[0].elements.find(x => x.text === 'SNAPME');
  console.log('CHECK toggle-on restores snap:', Math.abs((e1.x + e1.w / 2) - proj.cards[0].w / 2) <= 0.05 ? 'OK' : 'FAIL');

  // 4: align buttons -> exact positions, single undo reverts
  const clickAlign = t => page.locator('[title="' + t + '"]').click();
  await el.click();
  await clickAlign('Align left');  proj = await saveProj();
  e1 = proj.cards[0].elements.find(x => x.text === 'SNAPME');
  console.log('CHECK align-left x=0:', e1.x === 0 ? 'OK' : 'FAIL x=' + e1.x);
  await clickAlign('Align bottom'); proj = await saveProj();
  e1 = proj.cards[0].elements.find(x => x.text === 'SNAPME');
  console.log('CHECK align-bottom:', Math.abs(e1.y - (proj.cards[0].h - e1.h)) <= 0.05 ? 'OK' : 'FAIL');
  await page.keyboard.press('Control+z'); proj = await saveProj();
  e1 = proj.cards[0].elements.find(x => x.text === 'SNAPME');
  console.log('CHECK undo reverts align:', Math.abs(e1.y - (proj.cards[0].h - e1.h)) > 0.05 ? 'OK' : 'FAIL');

  // 5: resize does NOT snap (drag SE handle across center line)
  await el.click();
  const handles = page.locator('div[style*="nwse-resize"]');
  const hb = await handles.last().boundingBox();
  await page.mouse.move(hb.x + 5, hb.y + 5); await page.mouse.down();
  await page.mouse.move(hb.x + 33, hb.y + 5, { steps: 6 }); // sweep across a line region
  await page.screenshot({ path: path.join(OUT, '03-resize-no-guide.png') });
  await page.mouse.up();
  console.log('CHECK resize screenshot captured (verify no pink guide): see 03-resize-no-guide.png');

  // 6: back side snapping
  await page.getByRole('button', { name: 'Back', exact: true }).click();
  await page.waitForTimeout(300);
  // drag the star (icon element) off and back to center
  const face = page.locator('div').filter({ has: page.locator('svg') });
  // the back's star: drag from the card face center (star sits centered by default)
  const stage = await page.locator('canvas, div').first(); // fallback not needed; use keyboard nudge instead:
  // simpler assertion path: back star is already centered; drag it off-center then near-center and check via saved JSON
  // (back is saved in project v2)
  proj = await saveProj();
  console.log('INFO back exists in save:', !!proj.back ? 'OK' : 'FAIL');
  console.log('ERRORS:', errors.length ? errors.join(' | ') : 'none');
  await browser.close();
})().catch(e => { console.error('DRIVER FAILURE:', e); process.exit(1); });
```

For check 6 (back-side snap), if driving the star by mouse proves fiddly, an equivalent assertion: flip to Back, add a text element, drag it near the back's center exactly as in check 1, save, and assert against `proj.back.elements` — same `dragBy` + `saveProj` helpers work unchanged.

Expected output: every CHECK line `OK`, `ERRORS: none`. Inspect `01-snap-guide.png` — a 1px pink vertical line must be visible at the card's center; `03-resize-no-guide.png` — no pink line.

- [ ] **Step 2: Fix anything that fails, re-run until clean**

Any FAIL: diagnose in the app (not the driver first — the driver mimics a user), fix, `node --check`, re-run. Commit fixes with descriptive messages.

- [ ] **Step 3: Report results**

List each spec §6 item with pass/fail and reference the screenshots.

---

## Self-review notes

- **Spec coverage:** §1 snap model (targets incl. rotated-center-only, tolerance 6px/scale, tie-break card-first, pure `snapMove`) → Task 1; §2 state/UI (`snapOn`, `guides`, `opts.extra`, toggle chip, pink 1px lines interactive-only) → Tasks 1–2; §3 drag integration (targets cached at drag start, resize/rotate untouched, exact-landing coordinates) → Task 1; §4 align buttons (six, lazy patches, `key:'align'`, above action row) → Task 3; §5 edge cases (off-card lines allowed, no persistence of `snapOn` — explicit field lists in `saveJSON`/`pushHistory` mean no code needed) → noted in Task 2 Step 2; §6 verification → Task 4.
- **Naming consistency:** `buildSnapTargets`, `snapMove`, `state.snapOn`, `state.guides`, `opts.extra`, drag fields `box`/`targets`, icons `alCL/alCH/alCR/alCT/alCM/alCB` — consistent across tasks.
- **No placeholders:** every code step carries complete code; Task 4's check-6 fallback is fully specified in terms of existing helpers.
