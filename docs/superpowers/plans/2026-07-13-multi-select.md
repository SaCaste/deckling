# Multi-Select Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Select multiple elements (shift-click + marquee), drag them as a group with bounding-box snapping, and act on the group: align selection, distribute, duplicate, delete.

**Architecture:** Additive `state.selEls` (ids, populated only at 2+; invariant: multi ⇒ `selEl:null`, single ⇒ `selEls:[]`) so every shipped single-select path stays untouched. New drag modes `'gmove'` and `'marquee'` beside the existing `'move'/'resize'/'rotate'`. Group snapping reuses the shipped `snapMove` resolver on the selection bbox with members excluded from `buildSnapTargets`. Group ops are new isolated methods writing through `setCur` (side-aware, works on card backs).

**Tech Stack:** Single-file `index.html` — React 18 UMD class component (`h = React.createElement`), no build, **no test framework** (verification = headless-browser drive per `.claude/skills/verify/SKILL.md`; heed its locator-trap and px/mm-calibration notes).

**Spec:** `docs/superpowers/specs/2026-07-13-multi-select-design.md`

---

## File Structure

All changes in `index.html` (project convention — do not split):

- Task 1: state (~134), selection helpers (after `curEl` ~195), `onElPointerDown` (~483), `buildSnapTargets` (~449), `fixSel` (~237), `flipSide` (~209), `loadJSON` (~734), member outlines in `renderElDOM` (~1222), group bbox in `renderFace` (~1277)
- Task 2: marquee — face pointerdown in `renderFace`, new `faceMM`/`onFaceDown`, `onPointerMove` branch (~506), `onPointerUp` (~532), marquee rect render
- Task 3: `'gmove'` branch in `onPointerMove`
- Task 4: group ops methods, `onKey` (~176), `UICONS` (~131), `renderInspector` multi-panel (~1005)
- Task 5: verification only

Line numbers are anchors from commit `198ab7c` — content match wins. Work on branch `multi-select` (created in Task 1). After every code edit run the syntax check:

```bash
node -e "const s=require('fs').readFileSync('index.html','utf8'); const m=s.match(/<script>([\s\S]*?)<\/script>/); require('fs').writeFileSync('_check.js', m[1]);" && node --check _check.js && rm _check.js
```

---

### Task 1: Selection core (state, shift-click, invariant plumbing, outlines)

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Branch**

```bash
git checkout -b multi-select main
```

- [ ] **Step 2: State**

At ~line 136 the state block contains:

```js
    cards:[], selCard:null, selEl:null, editEl:null,
```

Change to:

```js
    cards:[], selCard:null, selEl:null, editEl:null,
    selEls:[], marquee:null,
```

- [ ] **Step 3: Selection helpers**

Insert directly after `curEl()` (~line 195):

```js
  // Normalize selection to the invariant: 0 ids -> none, 1 -> single selEl, 2+ -> multi selEls.
  setSelection(ids, extra){
    const u=[...new Set(ids)];
    if(u.length>=2) this.setState({ selEls:u, selEl:null, editEl:null, ...(extra||{}) });
    else this.setState({ selEls:[], selEl:u[0]||null, editEl:null, ...(extra||{}) });
  }
  selList(){ const c=this.curCard(); if(!c) return []; return c.elements.filter(e=>this.state.selEls.includes(e.id)); }
  selBBox(){
    const els=this.selList(); if(!els.length) return null;
    const x0=Math.min(...els.map(e=>e.x)), y0=Math.min(...els.map(e=>e.y));
    const x1=Math.max(...els.map(e=>e.x+e.w)), y1=Math.max(...els.map(e=>e.y+e.h));
    return { x:x0, y:y0, w:x1-x0, h:y1-y0 };
  }
```

- [ ] **Step 4: `buildSnapTargets` gains an exclusion list**

Current (~line 449):

```js
  buildSnapTargets(el){
    const c=this.curCard(); if(!c) return {v:[],h:[]};
    const v=[0, c.w/2, c.w, this.SAFE, c.w-this.SAFE];
    const hz=[0, c.h/2, c.h, this.SAFE, c.h-this.SAFE];
    c.elements.forEach(o=>{
      if(o.id===el.id) return;
```

Replace the signature and the skip line:

```js
  buildSnapTargets(el, excludeIds){
    const c=this.curCard(); if(!c) return {v:[],h:[]};
    const ex=excludeIds||[];
    const v=[0, c.w/2, c.w, this.SAFE, c.w-this.SAFE];
    const hz=[0, c.h/2, c.h, this.SAFE, c.h-this.SAFE];
    c.elements.forEach(o=>{
      if((el && o.id===el.id) || ex.includes(o.id)) return;
```

(Rest of the function unchanged. Existing single-drag call `this.buildSnapTargets(el)` keeps working.)

- [ ] **Step 5: Replace `onElPointerDown` (shift-click + gmove start + demote)**

Current (~line 483):

```js
  onElPointerDown(e, el){
    if(e.button!==0) return;
    if(this.state.editEl===el.id) return;
    e.stopPropagation();
    this.setState({ selEl:el.id });
    this.pushHistory('move_'+el.id);
    this.drag={ mode:'move', sx:e.clientX, sy:e.clientY, ox:el.x, oy:el.y,
      box:{w:el.w, h:el.h, rotation:el.rotation||0}, targets:this.buildSnapTargets(el) };
  }
```

Replace with:

```js
  onElPointerDown(e, el){
    if(e.button!==0) return;
    if(this.state.editEl===el.id) return;
    e.stopPropagation();
    const S=this.state.selEls;
    if(e.shiftKey){
      // selection toggle only — never starts a drag (spec §3)
      if(S.length) this.setSelection(S.includes(el.id) ? S.filter(i=>i!==el.id) : [...S, el.id]);
      else if(this.state.selEl && this.state.selEl!==el.id) this.setSelection([this.state.selEl, el.id]);
      else if(this.state.selEl===el.id) this.setSelection([]);
      else this.setState({ selEl:el.id });
      return;
    }
    if(S.includes(el.id)){
      // drag the whole selection (move handler lands in a later task)
      const members=this.selList().map(m=>({id:m.id, x:m.x, y:m.y}));
      const bb=this.selBBox();
      this.pushHistory('gmove');
      this.drag={ mode:'gmove', sx:e.clientX, sy:e.clientY, members, bb,
        targets:this.buildSnapTargets(null, S) };
      return;
    }
    this.setState({ selEl:el.id, selEls:[] });
    this.pushHistory('move_'+el.id);
    this.drag={ mode:'move', sx:e.clientX, sy:e.clientY, ox:el.x, oy:el.y,
      box:{w:el.w, h:el.h, rotation:el.rotation||0}, targets:this.buildSnapTargets(el) };
  }
```

- [ ] **Step 6: Invariant plumbing — `fixSel`, `flipSide`, `loadJSON`, empty-face click**

`fixSel` (~line 237) — replace entirely:

```js
  fixSel(){
    let { selCard, selEl, cards } = this.state;
    let c=cards.find(x=>x.id===selCard); if(!c){ c=cards[0]; selCard=c?c.id:null; }
    // Undoing past the first flip can restore back:null — fall to front view.
    const side = (this.state.side==='back' && !this.state.back) ? 'front' : this.state.side;
    const cur = side==='back' ? this.state.back : c;
    if(!cur || !cur.elements.find(e=>e.id===selEl)) selEl=null;
    let selEls=this.state.selEls.filter(id=>cur && cur.elements.find(e=>e.id===id));
    if(selEls.length===1){ selEl=selEls[0]; selEls=[]; }
    this.setState({ selCard, selEl, selEls, side });
  }
```

`flipSide` (~line 209): change the patch line

```js
    const patch = { side, selEl:null, editEl:null };
```

to

```js
    const patch = { side, selEl:null, selEls:[], editEl:null, marquee:null };
```

`loadJSON` (~line 743): in its `setState`, after `selEl:null,` insert `selEls:[], marquee:null,`.

Empty-face click in `renderFace` (~line 1295) — the interim behavior until Task 2's marquee replaces it: change

```js
      onPointerDown:interactive?(e=>{ if(e.target===e.currentTarget) this.setState({selEl:null}); }):undefined,
```

to

```js
      onPointerDown:interactive?(e=>{ if(e.target===e.currentTarget) this.setState({selEl:null, selEls:[]}); }):undefined,
```

- [ ] **Step 7: Member outlines + group bbox**

`renderElDOM` first line (~1223):

```js
    const h=this.h, B=this.B, sel=interactive && el.id===this.state.selEl, A=this.ac();
```

becomes

```js
    const h=this.h, B=this.B, sel=interactive && el.id===this.state.selEl, inSel=interactive && this.state.selEls.includes(el.id), A=this.ac();
```

Further down, the selection block starts (~1261):

```js
    const children=[inner];
    if(sel){
      children.push(h('div',{key:'sel',style:{position:'absolute',inset:-1,border:'1.5px solid '+A,borderRadius:el.type==='shape'&&el.shape==='circle'?'50%':3,pointerEvents:'none'}}));
      if(this.state.editEl!==el.id){
```

Change ONLY the two condition lines (outline shows for members; handles/rotator stay single-select):

```js
    const children=[inner];
    if(sel || inSel){
      children.push(h('div',{key:'sel',style:{position:'absolute',inset:-1,border:'1.5px solid '+A,borderRadius:el.type==='shape'&&el.shape==='circle'?'50%':3,pointerEvents:'none'}}));
      if(sel && this.state.editEl!==el.id){
```

In `renderFace`'s `if(interactive){` block, after the guides `forEach` added by the snapping feature, insert:

```js
      if(this.state.selEls.length>=2){
        const gb=this.selBBox();
        if(gb) children.push(h('div',{key:'gbb',style:{position:'absolute',left:(gb.x+B)*s,top:(gb.y+B)*s,width:gb.w*s,height:gb.h*s,outline:'1.5px dashed '+this.ac(),pointerEvents:'none',zIndex:4}}));
      }
```

- [ ] **Step 8: Verify + commit**

Syntax check (header command). Browser sanity (serve `python -m http.server 8987`): shift-click two elements → both outlined + dashed bbox; shift-click one again → back to single with handles; plain click on a non-member → single-select; empty-face click clears; dragging a member does nothing yet (expected — Task 3); single-element drag/snap unchanged. Console clean.

```bash
git add index.html
git commit -m "Multi-select core: selEls state + invariant, shift-click, member outlines, group bbox"
```

---

### Task 2: Marquee selection

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Face-relative mm helper + marquee start**

Insert directly above `onElPointerDown`:

```js
  faceMM(e){
    const r=this.faceEl.getBoundingClientRect(), s=this.scale, B=this.B;
    return { x:(e.clientX-r.left)/s-B, y:(e.clientY-r.top)/s-B };
  }
  onFaceDown(e){
    if(e.button!==0) return;
    const p=this.faceMM(e);
    this.drag={ mode:'marquee', sx:e.clientX, sy:e.clientY, x0:p.x, y0:p.y, moved:false };
  }
```

Change the empty-face pointerdown in `renderFace` (as left by Task 1):

```js
      onPointerDown:interactive?(e=>{ if(e.target===e.currentTarget) this.setState({selEl:null, selEls:[]}); }):undefined,
```

to

```js
      onPointerDown:interactive?(e=>{ if(e.target===e.currentTarget) this.onFaceDown(e); }):undefined,
```

- [ ] **Step 2: Marquee move branch**

In `onPointerMove`, after the `rotate` branch's closing brace (before the function's closing `}`), append:

```js
    else if(d.mode==='marquee'){
      if(Math.abs(e.clientX-d.sx)+Math.abs(e.clientY-d.sy)>3) d.moved=true;
      const p=this.faceMM(e);
      this.setState({ marquee:{ x0:d.x0, y0:d.y0, x1:p.x, y1:p.y } });
    }
```

- [ ] **Step 3: Marquee release (select or click-clear)**

Replace `onPointerUp` (~line 532):

```js
  onPointerUp(){ this.drag=null; if(this.state.guides.length) this.setState({guides:[]}); }
```

with

```js
  onPointerUp(){
    const d=this.drag; this.drag=null;
    if(d && d.mode==='marquee'){
      const m=this.state.marquee;
      if(!d.moved || !m){ this.setState({ selEl:null, selEls:[], editEl:null, marquee:null }); return; }
      const x0=Math.min(m.x0,m.x1), x1=Math.max(m.x0,m.x1), y0=Math.min(m.y0,m.y1), y1=Math.max(m.y0,m.y1);
      const c=this.curCard();
      const ids=(c?c.elements:[]).filter(el=> el.x<x1 && el.x+el.w>x0 && el.y<y1 && el.y+el.h>y0).map(el=>el.id);
      this.setSelection(ids, { marquee:null });
      return;
    }
    if(this.state.guides.length) this.setState({guides:[]});
  }
```

(Hit test = layout-box intersection; rotation ignored per spec §1. Zero hits → `setSelection([])` clears.)

- [ ] **Step 4: Marquee rectangle render**

In `renderFace`'s `if(interactive){` block, after the group-bbox block from Task 1, insert:

```js
      if(this.state.marquee){
        const m=this.state.marquee;
        const mx=Math.min(m.x0,m.x1), my=Math.min(m.y0,m.y1), mw=Math.abs(m.x1-m.x0), mh=Math.abs(m.y1-m.y0);
        children.push(h('div',{key:'mq',style:{position:'absolute',left:(mx+B)*s,top:(my+B)*s,width:mw*s,height:mh*s,border:'1px dashed '+this.ac(),background:this.ac()+'10',pointerEvents:'none',zIndex:6}}));
      }
```

- [ ] **Step 5: Verify + commit**

Syntax check. Browser: drag on empty canvas → dashed accent rectangle; release over 2+ elements → multi-selection; over 1 → single; over none → cleared; a plain click (no movement) still just deselects. Console clean.

```bash
git add index.html
git commit -m "Multi-select: marquee selection with dashed rubber-band rect"
```

---

### Task 3: Group move with bbox snapping

**Files:**
- Modify: `index.html` (`onPointerMove`)

- [ ] **Step 1: `gmove` branch**

In `onPointerMove`, insert as the FIRST branch (before `if(d.mode==='move'){`):

```js
    if(d.mode==='gmove'){
      const dx=(e.clientX-d.sx)/s, dy=(e.clientY-d.sy)/s;
      let nx=d.bb.x+dx, ny=d.bb.y+dy, guides=[];
      if(this.state.snapOn && !e.altKey){
        const r=this.snapMove(nx, ny, {w:d.bb.w, h:d.bb.h}, d.targets, 6/s);
        nx=r.x; ny=r.y; guides=r.guides;
      } else { nx=+nx.toFixed(1); ny=+ny.toFixed(1); }
      const ddx=nx-d.bb.x, ddy=ny-d.bb.y;
      const pos={}; d.members.forEach(m=>{ pos[m.id]={ x:+(m.x+ddx).toFixed(2), y:+(m.y+ddy).toFixed(2) }; });
      const c=this.curCard(); if(!c) return;
      this.setCur({ ...c, elements:c.elements.map(el=> pos[el.id] ? {...el, ...pos[el.id]} : el) }, { guides });
      return;
    }
```

(`snapMove({w,h})` — no `rotation` field, so the group box always uses full edge/center candidates. `d.targets` was built in Task 1 with all members excluded. History was pushed once at pointerdown; per-move writes bypass `updateSelEl` deliberately — they write all members in one `setCur` with guides riding as the extra.)

Change the existing `if(d.mode==='move'){` to `else if(d.mode==='move'){` so the chain stays exclusive.

- [ ] **Step 2: Verify + commit**

Syntax check. Browser: marquee-select two elements, drag one → both move together; with Snap on, group's outer edge snaps and pink guide shows; Alt bypasses; single Ctrl+Z after the drag restores both to pre-drag positions; single-element drag still snaps as before. Console clean.

```bash
git add index.html
git commit -m "Multi-select: group move with selection-bbox snapping"
```

---

### Task 4: Group operations, keyboard, inspector multi-panel

**Files:**
- Modify: `index.html` (`UICONS` ~131, group ops near `dupEl`, `onKey` ~176, `renderInspector` ~1005)

- [ ] **Step 1: Distribute icons**

In `UICONS`, after the `alCB:` entry (add trailing comma to it):

```js
    distH:'M3 4v16M21 4v16M9 8h6v8H9z',
    distV:'M4 3h16M4 21h16M8 9h8v6H8z'
```

- [ ] **Step 2: Group op methods**

Insert directly after `dupEl()` (~line 285):

```js
  deleteSels(){
    const c=this.curCard(); if(!c || this.state.selEls.length<2) return;
    this.pushHistory('gdel');
    const S=this.state.selEls;
    this.setCur({ ...c, elements:c.elements.filter(e=>!S.includes(e.id)) }, { selEls:[], selEl:null });
  }
  dupSels(){
    const c=this.curCard(); if(!c || this.state.selEls.length<2) return;
    this.pushHistory('gdup');
    const S=this.state.selEls;
    const clones=c.elements.filter(e=>S.includes(e.id)).map(e=>({ ...this.clone(e), id:this.uid(), x:+(e.x+4).toFixed(1), y:+(e.y+4).toFixed(1) }));
    this.setCur({ ...c, elements:[...c.elements, ...clones] }, { selEls:clones.map(e=>e.id), selEl:null });
  }
  alignSels(edge){
    const c=this.curCard(); const bb=this.selBBox(); if(!c || !bb || this.state.selEls.length<2) return;
    this.pushHistory('galign');
    const S=this.state.selEls;
    this.setCur({ ...c, elements:c.elements.map(e=>{
      if(!S.includes(e.id)) return e;
      if(edge==='l')  return {...e, x:+bb.x.toFixed(1)};
      if(edge==='ch') return {...e, x:+(bb.x+(bb.w-e.w)/2).toFixed(1)};
      if(edge==='r')  return {...e, x:+(bb.x+bb.w-e.w).toFixed(1)};
      if(edge==='t')  return {...e, y:+bb.y.toFixed(1)};
      if(edge==='cm') return {...e, y:+(bb.y+(bb.h-e.h)/2).toFixed(1)};
      return {...e, y:+(bb.y+bb.h-e.h).toFixed(1)}; // 'b'
    }) });
  }
  distributeSels(axis){
    const c=this.curCard(); if(!c || this.state.selEls.length<3) return;
    this.pushHistory('gdist');
    const S=this.state.selEls;
    const mem=c.elements.filter(e=>S.includes(e.id));
    const p=axis==='h'?'x':'y', dim=axis==='h'?'w':'h';
    const sorted=[...mem].sort((a,b)=>a[p]-b[p]);
    const first=sorted[0], last=sorted[sorted.length-1];
    const span=(last[p]+last[dim])-first[p];
    const total=sorted.reduce((sum,e)=>sum+e[dim],0);
    const gap=(span-total)/(sorted.length-1);
    const pos={}; let cur=first[p];
    sorted.forEach(e=>{ pos[e.id]=+cur.toFixed(1); cur+=e[dim]+gap; });
    pos[last.id]=+last[p].toFixed(1); // outer two stay fixed exactly (no rounding drift)
    this.setCur({ ...c, elements:c.elements.map(e=> S.includes(e.id) ? {...e, [p]:pos[e.id]} : e) });
  }
```

- [ ] **Step 3: Keyboard routing**

Replace the body of `onKey` (~line 176) with:

```js
  onKey(e){
    const tag = (e.target && e.target.tagName) || '';
    if(tag==='INPUT' || tag==='TEXTAREA') return;
    const mod = e.metaKey || e.ctrlKey;
    const multi = this.state.selEls.length>=2;
    if(mod && e.key.toLowerCase()==='z'){ e.preventDefault(); if(e.shiftKey) this.redo(); else this.undo(); }
    else if(mod && e.key.toLowerCase()==='y'){ e.preventDefault(); this.redo(); }
    else if(mod && e.key.toLowerCase()==='d'){ e.preventDefault(); if(multi) this.dupSels(); else this.dupEl(); }
    else if(mod && e.key.toLowerCase()==='a'){ e.preventDefault(); const c=this.curCard(); if(c) this.setSelection(c.elements.map(x=>x.id)); }
    else if((e.key==='Enter'||e.key==='F2') && this.state.selEl && !this.state.editEl){ const el=this.curEl(); if(el && el.type==='text'){ e.preventDefault(); this.enterEdit(el); } }
    else if((e.key==='Delete'||e.key==='Backspace') && (this.state.selEl||multi)){ e.preventDefault(); if(multi) this.deleteSels(); else this.deleteEl(); }
    else if(e.key==='Escape' && !this.state.editEl && (this.state.selEl||this.state.selEls.length)){ this.setState({ selEl:null, selEls:[] }); }
  }
```

(Existing lines are preserved verbatim; the additions are `multi` routing on Ctrl+D/Delete, Ctrl+A, and Escape. The inline-edit textarea's own Escape handler stops propagation, so editing is unaffected.)

- [ ] **Step 4: Inspector multi-panel**

`renderInspector` (~line 1005) currently begins:

```js
  renderInspector(){
    const t=this.th(), h=this.h, el=this.curEl();
    if(!el) return null;
```

Replace those lines with:

```js
  renderInspector(){
    const t=this.th(), h=this.h;
    if(this.state.selEls.length>=2) return this.renderMultiPanel();
    const el=this.curEl();
    if(!el) return null;
```

Then insert this new method directly above `renderInspector`:

```js
  renderMultiPanel(){
    const t=this.th(), h=this.h, n=this.state.selEls.length;
    const rows=[];
    rows.push(this.field('Align selection', h('div',{key:'galg',style:{display:'flex',gap:6}},
      this.iconBtn('alCL',()=>this.alignSels('l'),{title:'Align lefts'}),
      this.iconBtn('alCH',()=>this.alignSels('ch'),{title:'Align centers (H)'}),
      this.iconBtn('alCR',()=>this.alignSels('r'),{title:'Align rights'}),
      h('div',{style:{width:1,background:t.line}}),
      this.iconBtn('alCT',()=>this.alignSels('t'),{title:'Align tops'}),
      this.iconBtn('alCM',()=>this.alignSels('cm'),{title:'Align middles (V)'}),
      this.iconBtn('alCB',()=>this.alignSels('b'),{title:'Align bottoms'}))));
    rows.push(this.field('Distribute', h('div',{key:'gdst',style:{display:'flex',gap:6,opacity:n>=3?1:0.4,pointerEvents:n>=3?'auto':'none'}},
      this.iconBtn('distH',()=>this.distributeSels('h'),{title:'Distribute horizontally'}),
      this.iconBtn('distV',()=>this.distributeSels('v'),{title:'Distribute vertically'}))));
    rows.push(h('div',{key:'gact',style:{display:'flex',gap:6,marginTop:4,paddingTop:12,borderTop:'1px solid '+t.line}},
      this.iconBtn('copy',()=>this.dupSels(),{title:'Duplicate selection (⌘D)'}),
      h('div',{style:{flex:1}}),
      this.iconBtn(this.uIcon('trash',16,'#d4584a'),()=>this.deleteSels(),{title:'Delete selection'})));
    return this.section('Inspector · '+n+' selected', 'layers',
      h('div',{style:{background:t.panel2,border:'1px solid '+t.border,borderRadius:10,padding:'12px 12px 4px'}},rows));
  }
```

(`section`, `field`, `iconBtn`, `uIcon` are existing helpers; the trash-button pattern mirrors the single-select action row.)

- [ ] **Step 5: Verify + commit**

Syntax check. Browser: select 3 elements → panel shows "3 selected", align lefts lines them up, distribute-H equalizes gaps (outer two unmoved), Ctrl+D duplicates all (+4/+4, selection on clones), Delete removes all, each op = one Ctrl+Z; with 2 selected the distribute row is dimmed; Ctrl+A selects all; Escape clears. Console clean.

```bash
git add index.html
git commit -m "Multi-select: group ops (align/distribute/dup/delete), keyboard routing, inspector multi-panel"
```

---

### Task 5: Headless verification (spec §7)

**Files:** none. Serve `python -m http.server 8987`; scratch dir with `npm i playwright-core`; cached Chromium exe per `.claude/skills/verify/SKILL.md`. **Mind the skill's locator trap:** always pick the `getByText` match with `boundingBox().x` inside ~320–1280, and guard drags with a "moved" assertion; calibrate px/mm via a snap-off drag.

- [ ] **Step 1: Write and run the drive script**

```js
// drive-multi.js
const { chromium } = require('playwright-core');
const fs = require('fs'), path = require('path');
const EXE = 'C:/Users/Sanctuary/AppData/Local/ms-playwright/chromium-1228/chrome-win64/chrome.exe'; // adjust to cache
const OUT = path.join(__dirname, 'evidence-multi'); fs.mkdirSync(OUT, { recursive: true });

(async () => {
  const browser = await chromium.launch({ executablePath: EXE });
  const page = await (await browser.newContext({ acceptDownloads: true, viewport: { width: 1600, height: 1000 } })).newPage();
  const errors = [];
  page.on('pageerror', e => errors.push(e.message));
  await page.goto('http://localhost:8987/index.html', { waitUntil: 'networkidle' });
  await page.waitForSelector('text=Deck');

  const canvasBB = async (txt) => {
    const m = page.getByText(txt); const n = await m.count();
    for (let i = 0; i < n; i++) { const bb = await m.nth(i).boundingBox(); if (bb && bb.x > 320 && bb.x < 1280 && bb.width > 30) return bb; }
    throw new Error('canvas instance of ' + txt + ' not found');
  };
  const saveProj = async () => {
    const [dl] = await Promise.all([page.waitForEvent('download'), page.locator('[title="Save project (JSON)"]').click()]);
    const p = path.join(OUT, 'p.json'); await dl.saveAs(p);
    return JSON.parse(fs.readFileSync(p, 'utf8'));
  };
  const el = (proj, txt) => proj.cards[0].elements.find(x => x.text === txt);
  const clickCenter = async (txt, mods) => {
    const bb = await canvasBB(txt);
    await page.mouse.click(bb.x + bb.width / 2, bb.y + bb.height / 2, { modifiers: mods || [] });
    await page.waitForTimeout(150);
  };
  const dragFrom = async (txt, dx, dy, shot) => {
    const bb = await canvasBB(txt);
    const sx = bb.x + bb.width / 2, sy = bb.y + bb.height / 2;
    await page.mouse.move(sx, sy); await page.mouse.down();
    await page.mouse.move(sx + dx, sy + dy, { steps: 8 });
    if (shot) await page.screenshot({ path: path.join(OUT, shot + '.png') });
    await page.mouse.up(); await page.waitForTimeout(150);
  };

  // Setup: blank card, three text elements A/BB/CC, spread them out (snap OFF for setup)
  await page.getByRole('button', { name: 'Blank', exact: true }).click(); await page.waitForTimeout(200);
  const addText = async (txt) => {
    await page.getByRole('button', { name: 'Text', exact: true }).click();
    await page.waitForTimeout(300); await page.keyboard.type(txt); await page.keyboard.press('Escape'); await page.waitForTimeout(200);
  };
  await addText('ONE'); await addText('TWO'); await addText('THREE');
  await page.getByRole('button', { name: 'Snap', exact: true }).click(); // off for deterministic setup
  await dragFrom('ONE', -60, -180); await dragFrom('TWO', 0, -60); await dragFrom('THREE', 40, 90);
  // calibrate px/mm
  let p0 = await saveProj(); const x0 = el(p0, 'ONE').x;
  await dragFrom('ONE', 50, 0);
  let p1 = await saveProj(); const pxmm = 50 / (el(p1, 'ONE').x - x0);
  console.log('INFO pxPerMm=' + pxmm.toFixed(2));

  // 1: shift-click multi-select; toggle off again
  await clickCenter('ONE'); await clickCenter('TWO', ['Shift']);
  await page.screenshot({ path: path.join(OUT, '01-two-selected.png') });
  const twoSel = await page.getByText('2 selected').count();
  console.log('CHECK shift-click multi + panel:', twoSel > 0 ? 'OK' : 'FAIL');
  await clickCenter('TWO', ['Shift']);
  const backSingle = await page.getByText('2 selected').count();
  console.log('CHECK shift-click removes member:', backSingle === 0 ? 'OK' : 'FAIL');

  // 2: marquee over all three (drag from an empty corner across the card)
  const face = await canvasBB('TWO'); // any canvas element to locate the face region roughly
  await page.mouse.move(face.x - 150, face.y - 250); await page.mouse.down();
  await page.mouse.move(face.x + 250, face.y + 250, { steps: 8 });
  await page.screenshot({ path: path.join(OUT, '02-marquee.png') });
  await page.mouse.up(); await page.waitForTimeout(150);
  const threeSel = await page.getByText('3 selected').count();
  console.log('CHECK marquee selects 3:', threeSel > 0 ? 'OK' : 'FAIL');

  // 3: group drag moves all by same delta (snap still off)
  let pa = await saveProj();
  const before = { ONE: el(pa, 'ONE'), TWO: el(pa, 'TWO'), THREE: el(pa, 'THREE') };
  await dragFrom('TWO', 30, 20);
  let pb = await saveProj();
  const d1 = +(el(pb, 'ONE').x - before.ONE.x).toFixed(1), d2 = +(el(pb, 'TWO').x - before.TWO.x).toFixed(1), d3 = +(el(pb, 'THREE').x - before.THREE.x).toFixed(1);
  console.log('CHECK group drag same delta:', (d1 === d2 && d2 === d3 && d1 !== 0) ? 'OK (' + d1 + 'mm)' : 'FAIL ' + [d1, d2, d3]);
  await page.keyboard.press('Control+z'); await page.waitForTimeout(200);
  let pc = await saveProj();
  console.log('CHECK one undo reverts group drag:', el(pc, 'ONE').x === before.ONE.x && el(pc, 'THREE').x === before.THREE.x ? 'OK' : 'FAIL');

  // 4: group snap — snap ON, drag group so bbox left edge lands ~0.3mm from card center line
  await page.getByRole('button', { name: 'Snap', exact: true }).click(); // on
  let pd = await saveProj();
  const bbL = Math.min(el(pd, 'ONE').x, el(pd, 'TWO').x, el(pd, 'THREE').x);
  const need = (pd.cards[0].w / 2 + 0.3 - bbL) * pxmm;
  await dragFrom('TWO', need, 0, '03-group-snap');
  let pe = await saveProj();
  const bbL2 = Math.min(el(pe, 'ONE').x, el(pe, 'TWO').x, el(pe, 'THREE').x);
  console.log('CHECK group bbox snaps to card center:', Math.abs(bbL2 - pe.cards[0].w / 2) <= 0.05 ? 'OK' : 'FAIL bbL=' + bbL2);

  // 5: align lefts + distribute V exactness, one undo each
  await page.locator('[title="Align lefts"]').click(); await page.waitForTimeout(150);
  let pf = await saveProj();
  const xs = ['ONE', 'TWO', 'THREE'].map(t => el(pf, t).x);
  console.log('CHECK align lefts:', (xs[0] === xs[1] && xs[1] === xs[2]) ? 'OK (x=' + xs[0] + ')' : 'FAIL ' + xs);
  await page.waitForTimeout(800);
  await page.locator('[title="Distribute vertically"]').click(); await page.waitForTimeout(150);
  let pg = await saveProj();
  const items = ['ONE', 'TWO', 'THREE'].map(t => el(pg, t)).sort((a, b) => a.y - b.y);
  const gap1 = +(items[1].y - (items[0].y + items[0].h)).toFixed(2), gap2 = +(items[2].y - (items[1].y + items[1].h)).toFixed(2);
  console.log('CHECK distribute V equal gaps:', Math.abs(gap1 - gap2) <= 0.1 ? 'OK (' + gap1 + '/' + gap2 + ')' : 'FAIL ' + gap1 + ' vs ' + gap2);

  // 6: group duplicate & delete
  await page.waitForTimeout(800);
  await page.keyboard.press('Control+d'); await page.waitForTimeout(200);
  let ph = await saveProj();
  console.log('CHECK group duplicate:', ph.cards[0].elements.length === 6 ? 'OK' : 'FAIL n=' + ph.cards[0].elements.length);
  await page.keyboard.press('Delete'); await page.waitForTimeout(200);
  let pi = await saveProj();
  console.log('CHECK group delete removes clones:', pi.cards[0].elements.length === 3 ? 'OK' : 'FAIL n=' + pi.cards[0].elements.length);

  // 7: Ctrl+A and Escape
  await page.keyboard.press('Control+a'); await page.waitForTimeout(150);
  console.log('CHECK Ctrl+A:', (await page.getByText('3 selected').count()) > 0 ? 'OK' : 'FAIL');
  await page.keyboard.press('Escape'); await page.waitForTimeout(150);
  console.log('CHECK Escape clears:', (await page.getByText('3 selected').count()) === 0 ? 'OK' : 'FAIL');

  // 8: back side — shift-click two elements and group-drag
  await page.getByRole('button', { name: 'Back', exact: true }).click(); await page.waitForTimeout(400);
  await addText('BONE'); await addText('BTWO');
  await dragFrom('BONE', -40, -80);
  await clickCenter('BONE'); await clickCenter('BTWO', ['Shift']);
  let pj = await saveProj();
  const bb1 = { BONE: pj.back.elements.find(x => x.text === 'BONE'), BTWO: pj.back.elements.find(x => x.text === 'BTWO') };
  await dragFrom('BTWO', 25, 15);
  let pk = await saveProj();
  const bd1 = +(pk.back.elements.find(x => x.text === 'BONE').x - bb1.BONE.x).toFixed(1);
  const bd2 = +(pk.back.elements.find(x => x.text === 'BTWO').x - bb1.BTWO.x).toFixed(1);
  console.log('CHECK back-side group drag:', (bd1 === bd2 && bd1 !== 0) ? 'OK' : 'FAIL ' + [bd1, bd2]);

  // 9: single-select regression — front, click ONE, handles present, Enter edits
  await page.getByRole('button', { name: 'Front', exact: true }).click(); await page.waitForTimeout(300);
  await clickCenter('ONE');
  const handles = await page.locator('div[style*="nwse-resize"]').count();
  console.log('CHECK single-select handles intact:', handles >= 2 ? 'OK' : 'FAIL');

  console.log('ERRORS:', errors.length ? errors.join(' | ') : 'none');
  await browser.close();
})().catch(e => { console.error('DRIVER FAILURE:', e); process.exit(1); });
```

Expected: every CHECK `OK`, `ERRORS: none`. Inspect `01-two-selected.png` (outlines + dashed bbox), `02-marquee.png` (rubber-band rect), `03-group-snap.png` (pink guide at card center).

- [ ] **Step 2: Fix failures, re-run to clean** — diagnose app vs driver honestly (see the verify skill's false-positive lessons); commit fixes.

- [ ] **Step 3: Report per-check results with screenshots.**

---

## Self-review notes

- **Spec coverage:** §1 mechanics (shift-click incl. empty/same-element cases via `setSelection` + explicit branches, marquee w/ 3px click threshold, Ctrl+A, Escape, flip clears) → Tasks 1, 2, 4; §2 rendering (member outlines sans handles, dashed group bbox, marquee rect) → Tasks 1–2; §3 group move + bbox snap + member exclusion + shift-no-drag → Tasks 1, 3; §4 ops with history keys `gdel/gdup/galign/gdist` (+`gmove` at drag start) → Tasks 1, 4; §5 multi-panel → Task 4; §6 (`fixSel` prune/demote, `loadJSON` reset, JSON format untouched — selection never serialized: `saveJSON` field list is explicit) → Task 1; §7 → Task 5.
- **Type consistency:** `setSelection(ids, extra)`, `selList()`, `selBBox()`, `buildSnapTargets(el, excludeIds)`, drag fields `members/bb/targets`, ops `deleteSels/dupSels/alignSels(edge)/distributeSels(axis)`, icons `distH/distV` — names match across all tasks.
- **Ordering note:** Task 1 introduces the `'gmove'` drag object before Task 3 adds its move handler — between commits, dragging a multi-selection is a deliberate no-op (documented in Task 1 Step 8's expectations).
- **No placeholders:** all code steps carry complete code; the verification script is fully written.

---

## Addendum — Task 1 code review (2026-07-13)

Spec §1's "plain click on a member demotes to that single element" needs explicit delivery (pointerdown commits to `'gmove'`, so the demote must happen on a zero-movement release):

- Task 1 (done, this addendum's commit): the `'gmove'` drag object now carries `startId:el.id, moved:false`.
- Task 3: the `gmove` branch of `onPointerMove` must set `d.moved=true` once movement exceeds the 3-screen-px threshold (`Math.abs(e.clientX-d.sx)+Math.abs(e.clientY-d.sy)>3`), and `onPointerUp` gains a `gmove` case: if `!d.moved`, demote via `this.setState({ selEl:d.startId, selEls:[] })` (guides-clear behavior unchanged).
- Task 5: add a check — click (no drag) one member of a multi-selection → single-select with resize handles.
