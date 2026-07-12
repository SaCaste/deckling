# Card Backs + Duplex PDF Export Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** One shared card back per deck, edited with the full existing editor via a Front/Back flip toggle, exported as interleaved duplex PDF cut sheets (fronts on odd pages, x-mirrored backs on even pages).

**Architecture:** Approach A from the spec — a card-shaped `state.back` object plus `state.side` toggle. `curCard()` routes to the back when flipped, so all editing tools work unchanged; a new `setCur()` helper centralizes writes to "whichever side is on canvas". Undo history snapshots become `{cards, back}`. Export gains `drawBackFor()` (cover-fit back at any card size) and a duplex mode in `exportPDF`.

**Tech Stack:** Single-file app `index.html` — React 18 UMD class component (`h = React.createElement`), jsPDF, no build step, **no test framework** (verification is via served browser checks, the established pattern for this project).

**Spec:** `docs/superpowers/specs/2026-07-12-card-backs-duplex-pdf-design.md`

---

## File Structure

Everything happens in one file (project convention — do not split it):

- Modify: `index.html`
  - Task 1: state block (~line 128), helpers after `curEl()` (~line 185), history (~lines 187–212), write paths (~lines 214–273, 392–398)
  - Task 2: `renderCanvas` top bar (~line 1194), `renderRightPanel` deck list (~lines 1076–1102), `openSeries` (~line 734)
  - Task 3: export section (~lines 575–627), export panel UI (~line 1088)
  - Task 4: `saveJSON`/`loadJSON` (~lines 629–644)

Line numbers are anchors from the current file — verify with the stated context before editing; content match wins over line number.

**Working conventions for every task:** Serve the app with `python -m http.server 8000` from the repo root and open `http://localhost:8000/`. Hard-reload (Ctrl+Shift+R) after each edit. Watch the DevTools console — any red error after load or during the verification steps is a task failure.

---

### Task 1: Side-aware core (state, routing, history)

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add `side`, `back`, `duplexBacks` to initial state**

At ~line 128, the state block currently reads:

```js
  state = {
    theme:'studio', accent:'#10b981',
    cards:[], selCard:null, selEl:null, editEl:null,
```

Change the third line to:

```js
    cards:[], selCard:null, selEl:null, editEl:null,
    side:'front', back:null, duplexBacks:false,
```

- [ ] **Step 2: Replace `curCard()` and add routing helpers**

At ~line 184 the current code is:

```js
  curCard(){ return this.state.cards.find(c=>c.id===this.state.selCard) || null; }
  curEl(){ const c=this.curCard(); if(!c) return null; return c.elements.find(e=>e.id===this.state.selEl) || null; }
```

Replace with:

```js
  isBack(){ return this.state.side==='back'; }
  frontCard(){ return this.state.cards.find(c=>c.id===this.state.selCard) || null; }
  curCard(){ return this.isBack() ? this.state.back : this.frontCard(); }
  curEl(){ const c=this.curCard(); if(!c) return null; return c.elements.find(e=>e.id===this.state.selEl) || null; }
  // Write the updated card object back to whichever side is on canvas.
  setCur(next, extra){
    if(this.isBack()) this.setState({ back:next, ...(extra||{}) });
    else this.setState({ cards:this.state.cards.map(c=>c.id===next.id?next:c), ...(extra||{}) });
  }
  // Default back: accent-colored solid with a centered white star (spec §1).
  defaultBack(ref){
    const w=ref?ref.w:this.TYPES.poker.w, h=ref?ref.h:this.TYPES.poker.h;
    return { name:'Card back', typeId:ref?ref.typeId:'poker', w, h,
      bg:{type:'solid', color:this.ac(), from:'#ffffff', to:'#dfe6ee', angle:135},
      border:{width:0, color:'#1c2230', radius:3},
      elements:[{ id:this.uid(), type:'icon', x:+(w/2-9).toFixed(1), y:+(h/2-9).toFixed(1), w:18, h:18, icon:'star', color:'#ffffff', rotation:0, opacity:1 }] };
  }
  flipSide(){
    const side = this.isBack() ? 'front' : 'back';
    this._hk=null; // break undo coalescing across sides
    const patch = { side, selEl:null, editEl:null };
    if(side==='back' && !this.state.back) patch.back=this.defaultBack(this.frontCard());
    this.setState(patch);
  }
```

Notes: `back` has no card `id`/`qty` (never enters `cards[]`), but it does carry `name` (PNG export filename) and `typeId` (left-panel Card Type section reads `c.typeId` at ~line 986). `setCur`'s front branch maps by `next.id`; the back branch ignores id.

- [ ] **Step 3: History snapshots include the back**

At ~lines 187–212, replace `pushHistory`, `undo`, `redo`, `fixSel` entirely:

```js
  pushHistory(key){
    const now=Date.now();
    if(key && key===this._hk && now-(this._ht||0)<700){ this._ht=now; return; }
    this._hk=key; this._ht=now;
    this.history.past.push(this.clone({ cards:this.state.cards, back:this.state.back }));
    if(this.history.past.length>60) this.history.past.shift();
    this.history.future=[];
  }
  undo(){
    if(!this.history.past.length) return;
    this.history.future.push(this.clone({ cards:this.state.cards, back:this.state.back }));
    const snap=this.history.past.pop();
    this.setState({ cards:snap.cards, back:snap.back }, ()=>this.fixSel());
  }
  redo(){
    if(!this.history.future.length) return;
    this.history.past.push(this.clone({ cards:this.state.cards, back:this.state.back }));
    const snap=this.history.future.pop();
    this.setState({ cards:snap.cards, back:snap.back }, ()=>this.fixSel());
  }
  fixSel(){
    let { selCard, selEl, cards } = this.state;
    let c=cards.find(x=>x.id===selCard); if(!c){ c=cards[0]; selCard=c?c.id:null; }
    // Undoing past the first flip can restore back:null — fall to front view.
    const side = (this.state.side==='back' && !this.state.back) ? 'front' : this.state.side;
    const cur = side==='back' ? this.state.back : c;
    if(!cur || !cur.elements.find(e=>e.id===selEl)) selEl=null;
    this.setState({ selCard, selEl, side });
  }
```

- [ ] **Step 4: Route all write paths through `setCur`**

Replace each function below (current versions at ~lines 215–273 and 392–398) with the side-aware version. Behavior on the front is byte-identical; the only change is where the write lands.

```js
  patchCard(patch, hist){ const c=this.curCard(); if(!c) return; if(hist) this.pushHistory(hist); this.setCur({...c, ...patch}); }
  updateSelEl(patch, opts){
    opts=opts||{};
    const c=this.curCard(); if(!c) return;
    if(opts.hist!==false) this.pushHistory(opts.key||'edit');
    this.setCur({ ...c, elements:c.elements.map(e=> e.id!==this.state.selEl ? e : {...e,...patch}) });
  }
```

```js
  addEl(type, extra){
    const c=this.curCard(); if(!c) return;
    this.pushHistory('add');
    const el=this.makeEl(type,c,extra);
    const editing = type==='text';
    if(editing){ this._focusEdit=true; this._selectNew=true; this._hk=null; }
    this.setCur({...c, elements:[...c.elements, el]}, { selEl:el.id, leftTab:'card', editEl: editing? el.id : null });
  }
  deleteEl(){
    const c=this.curCard(); if(!c||!this.state.selEl) return;
    this.pushHistory('del');
    this.setCur({...c, elements:c.elements.filter(e=>e.id!==this.state.selEl)}, { selEl:null });
  }
  dupEl(){
    const c=this.curCard(); const el=this.curEl(); if(!c||!el) return;
    this.pushHistory('dupel');
    const ne={...this.clone(el), id:this.uid(), x:el.x+4, y:el.y+4};
    const idx=c.elements.findIndex(e=>e.id===el.id);
    const els=[...c.elements]; els.splice(idx+1,0,ne);
    this.setCur({...c, elements:els}, { selEl:ne.id });
  }
  layer(dir){
    const c=this.curCard(); const el=this.curEl(); if(!c||!el) return;
    this.pushHistory('layer');
    let els=[...c.elements]; const i=els.findIndex(e=>e.id===el.id);
    els.splice(i,1);
    let j = dir==='front'?els.length : dir==='back'?0 : dir==='up'?Math.min(els.length,i+1) : Math.max(0,i-1);
    els.splice(j,0,el);
    this.setCur({...c, elements:els});
  }
```

And `applyTemplate` at ~line 392:

```js
  applyTemplate(tid){
    const c=this.curCard(); if(!c) return;
    this.pushHistory('tmpl');
    const els = tid==='blank' ? [] : this.templates()[tid](c.w,c.h);
    this.setCur({...c, elements:els}, { selEl:null });
  }
```

Leave untouched (they are deck-panel/front-only by design): `setCards`, `blankCard`, `addCard`, `dupCard`, `delCard`, `moveCard`, `setQty`, `selectCard`. `setType`/`setBg`/`setBorder` already call `patchCard` and need no edits.

- [ ] **Step 5: Verify no regression on the front side (browser)**

Serve and hard-reload. On the sample deck: drag an element, resize, rotate, double-click a text and edit, add an icon via the toolbar, delete an element, apply a template, Ctrl+Z several times, Ctrl+Shift+Z. Everything must behave exactly as before; console clean. (The back is not reachable yet — that's Task 2.)

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Card backs core: side-aware state, curCard routing, history snapshots {cards, back}"
```

---

### Task 2: Flip toggle UI + panel guards

**Files:**
- Modify: `index.html` (`renderCanvas` ~line 1182, `renderRightPanel` ~line 1076, `openSeries` ~line 734, Batch fill button ~line 1097)

- [ ] **Step 1: Add the Front/Back segmented toggle to the canvas top bar**

In `renderCanvas`, the floating top bar (~line 1194) currently starts:

```js
      h('div',{style:{position:'absolute',top:14,left:'50%',transform:'translateX(-50%)',zIndex:4,display:'flex',gap:8,alignItems:'center',background:t.panel,border:'1px solid '+t.border,borderRadius:10,padding:'6px 8px',boxShadow:t.shadow}},
        toggle('showBleed','Bleed','rgba(220,60,60,.8)'),
```

First, inside `renderCanvas` after the `zb` helper (~line 1191), add:

```js
    const A=this.ac(), side=this.state.side;
    const flipBtn=(v,label)=>h('button',{key:v,onClick:()=>{ if(side!==v) this.flipSide(); },
      style:{padding:'4px 12px',borderRadius:6,border:'none',cursor:'pointer',fontSize:11.5,fontWeight:700,
        background:side===v?A:'transparent',color:side===v?'#fff':t.sub,transition:'all .12s'}},label);
```

Then insert two children at the front of the top bar (before `toggle('showBleed',…)`):

```js
        h('div',{style:{display:'flex',gap:2,background:t.panel2,border:'1px solid '+t.border,borderRadius:8,padding:2}},
          flipBtn('front','Front'), flipBtn('back','Back')),
        h('div',{style:{width:1,height:18,background:t.border}}),
```

No other `renderCanvas` change is needed: it already renders `this.curCard()`, so flipping swaps the face, and the size readout shows the back's dimensions.

- [ ] **Step 2: Dim the deck list while on the back**

In `renderRightPanel` (~line 1076), add at the top of the function body, after `const total=…`:

```js
    const dim=this.isBack();
```

Change the header's add button (~line 1080) from:

```js
        this.iconBtn('plus',()=>this.addCard(),{title:'Add card',w:30})),
```

to:

```js
        h('div',{style:{opacity:dim?0.45:1,pointerEvents:dim?'none':'auto'}},
          this.iconBtn('plus',()=>this.addCard(),{title:'Add card',w:30}))),
```

And change the card-list container (~line 1081) style from:

```js
      h('div',{style:{flex:1,overflowY:'auto',padding:'10px 12px',display:'flex',flexDirection:'column',gap:8}},
```

to:

```js
      h('div',{style:{flex:1,overflowY:'auto',padding:'10px 12px',display:'flex',flexDirection:'column',gap:8,opacity:dim?0.45:1,pointerEvents:dim?'none':'auto'}},
```

The Export & Project section stays fully active on both sides.

- [ ] **Step 3: Force front side when opening Batch and Series**

Both generators must always use the front card as base (spec §2). At ~line 734, `openSeries` currently reads:

```js
  openSeries(){
    const c=this.curCard(); if(!c){ this.toast('Select a base card'); return; }
    this.setState({ modal:'series', series:{ mode:'product', vars:[] } });
  }
```

Replace with:

```js
  openSeries(){
    const c=this.frontCard(); if(!c){ this.toast('Select a base card'); return; }
    this.setState({ modal:'series', side:'front', selEl:null, editEl:null, series:{ mode:'product', vars:[] } });
  }
```

At ~line 1097 the Batch fill button currently reads:

```js
          eBtn('table','Batch fill',()=>this.setState({modal:'batch'})),
```

Replace with:

```js
          eBtn('table','Batch fill',()=>this.setState({modal:'batch', side:'front', selEl:null, editEl:null})),
```

After `side:'front'` is set, `curCard()` inside `batchGenerate`/`seriesTargets`/`seriesGenerate`/`renderModal` resolves to the front card — no further changes there.

- [ ] **Step 4: Verify the flip workflow (browser)**

1. Click **Back** → default back appears (accent solid + white star), deck list dims, selection cleared.
2. Edit the back: move the star, add a text ("PROTOTYPE"), double-click to edit it, Enter/F2 works, add a shape; all element inspector controls work.
3. Change back background color and border via the left panel; switch Card Type on the back — the back resizes, fronts untouched.
4. Click **Front** → original card untouched; click **Back** → your back edits are still there.
5. Undo repeatedly: back edits revert first, then (crossing sides) front edits; the canvas follows sensibly (undoing past first flip lands on Front view).
6. While on Back: click **Generate series** → modal opens showing the front card as base; close; same for **Batch fill**.
7. While on Back: **PNG card** downloads `Card back.png` with your design.
8. Console clean throughout.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Card backs UI: Front/Back flip toggle, deck panel dimming, front-side guards for batch/series"
```

---

### Task 3: Duplex PDF export

**Files:**
- Modify: `index.html` (export section ~lines 575–627, export panel ~line 1088)

- [ ] **Step 1: Add `drawBackFor` and the `cutMarks` helper**

Insert directly above `exportPDF` (~line 588):

```js
  // Back rendered at a given card's size: cover-fit over the full bleed
  // extents so mixed-size decks never show uncovered slivers (spec §3).
  drawBackFor(card, ppm){
    const B=this.B;
    const back=this.state.back || this.defaultBack(card);
    const s=Math.max((card.w+2*B)/(back.w+2*B), (card.h+2*B)/(back.h+2*B));
    const src=this.drawCard(back, ppm*s);
    const pw=Math.round((card.w+2*B)*ppm), ph=Math.round((card.h+2*B)*ppm);
    const cv=document.createElement('canvas'); cv.width=pw; cv.height=ph;
    cv.getContext('2d').drawImage(src, Math.round((pw-src.width)/2), Math.round((ph-src.height)/2));
    return cv;
  }
  cutMarks(pdf, x, y, card){
    const B=this.B, m=3.2;
    pdf.setDrawColor(40); pdf.setLineWidth(0.15);
    const tx=x+B, ty=y+B, tr=x+B+card.w, tb=y+B+card.h;
    pdf.line(tx-m,ty, tx-0.5,ty); pdf.line(tx,ty-m, tx,ty-0.5);
    pdf.line(tr+0.5,ty, tr+m,ty); pdf.line(tr,ty-m, tr,ty-0.5);
    pdf.line(tx-m,tb, tx-0.5,tb); pdf.line(tx,tb+0.5, tx,tb+m);
    pdf.line(tr+0.5,tb, tr+m,tb); pdf.line(tr,tb+0.5, tr,tb+m);
  }
```

Note on the cover-fit ratio: it must use bleed-inclusive extents `(w+2B)`, not trim sizes. With trim ratios, a tarot back (70×120) scaled down for a poker card (63×88) gives s=0.90 and the scaled bleed shrinks below 3 mm — total 68.4 mm < 69 mm needed → white sliver. Bleed-inclusive ratios guarantee full coverage.

- [ ] **Step 2: Rewrite `exportPDF` with duplex support**

Replace the entire `exportPDF` (~lines 588–627) with:

```js
  async exportPDF(pageId){
    if(!window.jspdf){ this.toast('PDF engine still loading…'); return; }
    const { jsPDF }=window.jspdf;
    const ppm=300/25.4, B=this.B;
    const pages={ letter:{w:215.9,h:279.4,fmt:'letter'}, a4:{w:210,h:297,fmt:'a4'} };
    const pg=pages[pageId]||pages.letter;
    const margin=8, gap=0;
    const duplex=this.state.duplexBacks;
    this.toast('Building PDF…');
    const imgCards=[...this.state.cards];
    if(duplex && this.state.back) imgCards.push(this.state.back);
    await this.ensureImages(imgCards);
    const list=[];
    this.state.cards.forEach(c=>{ for(let q=0;q<(c.qty||1);q++) list.push(c); });
    if(!list.length){ this.toast('No cards'); return; }
    const pdf=new jsPDF({ unit:'mm', format:pg.fmt, orientation:'portrait' });
    const backData={}; // rendered back PNG per distinct card size
    const backFor=(card)=>{
      const k=card.w+'x'+card.h;
      if(!backData[k]) backData[k]=this.drawBackFor(card,ppm).toDataURL('image/png');
      return backData[k];
    };
    let placed=0, first=true, idx=0;
    while(idx<list.length){
      if(!first) pdf.addPage(pg.fmt,'portrait');
      first=false;
      let x=margin, y=margin, rowH=0;
      const slots=[];
      while(idx<list.length){
        const card=list[idx];
        const cw=card.w+2*B, ch=card.h+2*B;
        if(x+cw > pg.w-margin){ x=margin; y+=rowH+gap; rowH=0; }
        if(y+ch > pg.h-margin){ break; }
        const cv=this.drawCard(card,ppm);
        pdf.addImage(cv.toDataURL('image/png'),'PNG', x, y, cw, ch, undefined, 'FAST');
        this.cutMarks(pdf, x, y, card);
        slots.push({ x, y, card });
        x+=cw+gap; rowH=Math.max(rowH,ch); idx++; placed++;
      }
      if(duplex && slots.length){
        pdf.addPage(pg.fmt,'portrait');
        slots.forEach(sl=>{
          const cw=sl.card.w+2*B, ch=sl.card.h+2*B;
          const bx=pg.w - sl.x - cw; // x-mirror for long-edge flip
          pdf.addImage(backFor(sl.card),'PNG', bx, sl.y, cw, ch, undefined, 'FAST');
          this.cutMarks(pdf, bx, sl.y, sl.card);
        });
      }
    }
    pdf.save('card-deck-'+pageId+'.pdf');
    this.toast('PDF: '+placed+' cards exported'+(duplex?' (double-sided)':''));
  }
```

With `duplexBacks` false this produces the identical document to the old code — same placement math, same marks, only refactored into `cutMarks` and a `slots` record (spec §4 regression guarantee).

- [ ] **Step 3: Add the "Double-sided (card backs)" checkbox to the export panel**

In `renderRightPanel`, above the PDF buttons row (~line 1088, right after the `'Export & Project'` heading div), insert:

```js
        h('label',{style:{display:'flex',alignItems:'center',gap:8,fontSize:12,fontWeight:600,color:t.text,cursor:'pointer',padding:'0 2px'}},
          h('input',{type:'checkbox',checked:this.state.duplexBacks,onChange:e=>this.setState({duplexBacks:e.target.checked}),style:{accentColor:A}}),
          'Double-sided (card backs)'),
```

- [ ] **Step 4: Verify duplex export (browser)**

1. Checkbox **off** → `PDF · Letter`: page count and layout identical to before (sample deck: compare against a pre-change export if one exists, otherwise sanity-check marks/positions).
2. Checkbox **on**, never having flipped to Back → PDF alternates fronts/backs pages; backs show the default design (defaultBack fallback in `drawBackFor`).
3. Design a back with an off-center text (e.g. top-left corner) → export duplex → in the PDF, on backs pages the design appears mirrored in position across the page (a card at page-left front has its back at page-right), each back internally un-mirrored.
4. Overlay check: in a PDF viewer, page 1 and page 2 card outlines must align when page 2 is flipped horizontally — equivalently, x-positions satisfy `bx = pageW − x − cardW` (spot-check two cards with a ruler tool or by eye against margins).
5. Mixed sizes: sample deck already includes poker + tarot + mini → every back fully covers its card, centered, no white slivers inside the cut marks.
6. Back containing an uploaded image exports correctly (ensureImages preloads it).
7. `PDF · A4` duplex also works. Console clean.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Duplex PDF export: mirrored backs pages, cover-fit back rendering, export checkbox"
```

---

### Task 4: Save/Load v2

**Files:**
- Modify: `index.html` (`saveJSON` ~line 629, `loadJSON` ~line 634)

- [ ] **Step 1: Bump save format and persist the back**

`saveJSON` currently:

```js
  saveJSON(){
    const data={ app:'card-maker', version:1, cards:this.state.cards, theme:this.state.theme, accent:this.state.accent, assets:this.state.assets };
    this.download('card-project.json', new Blob([JSON.stringify(data)],{type:'application/json'}));
    this.toast('Project saved');
  }
```

Replace the first line of the body with:

```js
    const data={ app:'card-maker', version:2, cards:this.state.cards, back:this.state.back, theme:this.state.theme, accent:this.state.accent, assets:this.state.assets };
```

- [ ] **Step 2: Load v1 and v2 files**

In `loadJSON`, the `setState` call currently reads:

```js
      this.setState({ cards:d.cards, selCard:d.cards[0]&&d.cards[0].id, selEl:null, theme:d.theme||this.state.theme, accent:d.accent||this.state.accent, assets:Array.isArray(d.assets)?d.assets:[] });
```

Replace with:

```js
      this.setState({ cards:d.cards, back:d.back||null, side:'front', selCard:d.cards[0]&&d.cards[0].id, selEl:null, editEl:null, theme:d.theme||this.state.theme, accent:d.accent||this.state.accent, assets:Array.isArray(d.assets)?d.assets:[] });
```

v1 files (no `back` field) load with `back:null`; the default back is generated lazily on first flip (`flipSide`) or at export (`drawBackFor` fallback) — user-visibly equivalent to spec §5 and keeps one source of truth for the default.

- [ ] **Step 3: Verify round-trip (browser)**

1. Design a back, save project → JSON contains `"version":2` and a `back` object (open the file and check).
2. Reload the page, load that JSON → flip to Back → design restored; duplex export uses it.
3. Load a **v1** file (create one: in DevTools console before saving, or use any previously saved project file) → loads without errors, toast "Project loaded", flip to Back shows the default back.
4. Console clean.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Project files v2: persist card back, accept v1 files"
```

---

### Task 5: Full smoke checklist (spec §7) 

**Files:** none (verification only)

- [ ] **Step 1: Run the complete spec §7 checklist end-to-end** on the served app, in order, fresh reload first:

1. Flip → default back; add/edit text, icon, image; flip front — intact; flip back — intact.
2. Undo/redo crosses sides in order (edit front → edit back → Ctrl+Z ×2 restores both).
3. Duplex PDF, uniform deck (delete non-poker cards first): pages 1–2 outlines align mirrored.
4. Duplex PDF, mixed sizes (fresh reload, sample deck): backs cover fully, centered.
5. Duplex off → export identical to pre-feature behavior.
6. v1 JSON loads clean, default back present.
7. PNG card while on Back exports the back.

- [ ] **Step 2: Report results to the user** — list each check with pass/fail. Any failure: stop, fix, re-run the failed check before proceeding.

---

## Self-review notes

- **Spec coverage:** §1 data model → Task 1; §2 editing/flip/guards → Tasks 1–2; §3 scaled rendering → Task 3 Step 1; §4 duplex export + checkbox + ensureImages + PNG bonus → Task 3 (PNG bonus verified Task 2 Step 4.7); §5 save/load → Task 4; §6 edge cases → covered by lazy creation (`flipSide`), `fixSel` side fallback, and front-only guards; §7 verification → Task 5.
- **Naming consistency:** `isBack`, `frontCard`, `curCard`, `setCur`, `defaultBack`, `flipSide`, `drawBackFor`, `cutMarks`, `duplexBacks`, `state.side`, `state.back` — used identically across all tasks.
- **Known intentional deviations from spec text:** v1 load creates the default back lazily rather than eagerly (§5) — user-visibly equivalent, noted in Task 4 Step 2.
