# Series Generator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a guided "Generate series" tool: design one base card, then produce N cards varying a number/icon/color/background across a range or list, with Zip and Product combination modes.

**Architecture:** All changes are in the single-file React app `index.html`. A new `series` state object holds the combination `mode` and a list of `vars`, each binding a base-card target (`elId`+`prop`, or `'bg'`) to values (numeric range or list). A pure-ish engine (`resolveVar`, `seriesCount`, `seriesGenerate`) clones the base card N times applying each variable's value per index (Zip = cycle; Product = mixed-radix). A dedicated `renderSeriesModal` provides the picker/editors, opened from a new sidebar button. The existing CSV `batchGenerate` flow is untouched.

**Tech Stack:** React 18 (UMD, class component, `h = React.createElement`), single HTML file, no build step, no test framework.

**Testing note:** This project has no automated test harness and no build step. Per-task verification is a **manual browser check** plus a `node --check` syntax gate on the inline script. Do not add a test framework (YAGNI). Reference spec: `docs/superpowers/specs/2026-07-07-series-generator-design.md`.

**Syntax-check helper (used in several tasks):** from the repo root run:
```bash
node -e 'const fs=require("fs");const h=fs.readFileSync("index.html","utf8");const m=h.match(/<script(?![^>]*\bsrc=)[^>]*>([\s\S]*?)<\/script>/i);const f=require("os").tmpdir()+"/cs_check.js";fs.writeFileSync(f,m[1]);require("child_process").execSync("node --check "+f,{stdio:"inherit"});console.log("PARSE_OK");'
```
Expected output: `PARSE_OK`.

---

## File Structure

- **Modify only:** `index.html`
  - `state` init (~line 116): add `series:{ mode:'zip', vars:[] }`.
  - New methods after `batchGenerate` (~line 710): `openSeries`, `seriesTargets`, `addSeriesVar`, `updateSeriesVar`, `removeSeriesVar`, `resolveVar`, `seriesCount`, `seriesGenerate`.
  - New method `renderSeriesModal` after `renderModal` (~line 1152).
  - New "Generate series" button in the Export panel (~line 996).
  - Wire `this.renderSeriesModal()` into `render()` (~line 1163).

---

## Task 1: Series state + generation engine

**Files:**
- Modify: `index.html:116` (state init)
- Modify: `index.html` (add 8 methods after `batchGenerate`, ~line 710)

- [ ] **Step 1: Add `series` to state**

Find (line 116):

```js
    leftTab:'card', modal:null, batchText:'', toast:null
```

Replace with:

```js
    leftTab:'card', modal:null, batchText:'', toast:null,
    series:{ mode:'zip', vars:[] }
```

- [ ] **Step 2: Add the engine methods**

Locate the `batchGenerate(){ ... }` method. It ends (~line 710) with:

```js
    this.setState({ cards:[...this.state.cards, ...newCards], modal:null });
    this.toast('Generated '+newCards.length+' cards');
  }
```

Immediately after that method's closing brace, insert these eight methods:

```js
  openSeries(){
    const c=this.curCard(); if(!c){ this.toast('Select a base card'); return; }
    this.setState({ modal:'series', series:{ mode:'zip', vars:[] } });
  }
  seriesTargets(){
    const c=this.curCard(); if(!c) return [];
    const out=[];
    c.elements.forEach(e=>{
      if(e.type==='text'){ const snip=(e.text||'').replace(/\s+/g,' ').slice(0,16); out.push({elId:e.id,label:'Text · "'+snip+'"',props:[{prop:'text',label:'Number/Text'},{prop:'color',label:'Color'}]}); }
      else if(e.type==='icon'){ out.push({elId:e.id,label:'Icon · '+e.icon,props:[{prop:'icon',label:'Icon'},{prop:'color',label:'Color'}]}); }
      else if(e.type==='shape'){ out.push({elId:e.id,label:'Shape · '+e.shape,props:[{prop:'fill',label:'Fill color'}]}); }
    });
    out.push({elId:'bg',label:'Background',props:[{prop:'color',label:'Background color'}]});
    return out;
  }
  addSeriesVar(elId, prop){
    const c=this.curCard(); if(!c) return;
    let list=[];
    if(elId==='bg'){ list=[(c.bg&&c.bg.color)||'#ffffff']; }
    else {
      const el=c.elements.find(e=>e.id===elId); if(!el) return;
      if(prop==='icon') list=[el.icon].filter(Boolean);
      else if(prop==='color') list=[el.color].filter(Boolean);
      else if(prop==='fill') list=[el.fill].filter(Boolean);
    }
    const v={ id:this.uid(), elId, prop, valueMode: prop==='text'?'range':'list', from:1, to:5, step:1, list };
    this.setState({ series:{...this.state.series, vars:[...this.state.series.vars, v]} });
  }
  updateSeriesVar(id, patch){
    const vars=this.state.series.vars.map(v=>v.id!==id?v:{...v,...patch});
    this.setState({ series:{...this.state.series, vars} });
  }
  removeSeriesVar(id){
    const vars=this.state.series.vars.filter(v=>v.id!==id);
    this.setState({ series:{...this.state.series, vars} });
  }
  resolveVar(v){
    if(v.prop==='text' && v.valueMode==='range'){
      const step=Math.abs(v.step||1); if(!(step>0)) return [];
      const out=[]; const from=+v.from, to=+v.to;
      if(to>=from){ for(let n=from;n<=to+1e-9;n+=step){ out.push(String(+n.toFixed(4))); if(out.length>=500) break; } }
      else { for(let n=from;n>=to-1e-9;n-=step){ out.push(String(+n.toFixed(4))); if(out.length>=500) break; } }
      return out;
    }
    return (v.list||[]).map(x=>String(x)).filter(x=>x.length>0);
  }
  seriesCount(){
    const arrs=this.state.series.vars.map(v=>this.resolveVar(v)).filter(a=>a.length);
    if(!arrs.length) return 0;
    if(this.state.series.mode==='product') return arrs.reduce((n,a)=>n*a.length,1);
    return arrs.reduce((m,a)=>Math.max(m,a.length),0);
  }
  seriesGenerate(){
    const c=this.curCard(); if(!c){ this.toast('Select a base card'); return; }
    const vars=this.state.series.vars;
    const arrs=vars.map(v=>({v,vals:this.resolveVar(v)})).filter(o=>o.vals.length);
    if(!arrs.length){ this.toast('Add at least one variable with values'); return; }
    const mode=this.state.series.mode;
    const N = mode==='product' ? arrs.reduce((n,o)=>n*o.vals.length,1) : arrs.reduce((m,o)=>Math.max(m,o.vals.length),0);
    if(N>150){ this.toast('Too many cards ('+N+') — narrow the ranges'); return; }
    this.pushHistory('series');
    const textArr=arrs.find(o=>o.v.prop==='text');
    const newCards=[];
    for(let i=0;i<N;i++){
      const valById={}; let acc=i;
      arrs.forEach(o=>{ let vi; if(mode==='product'){ vi=acc % o.vals.length; acc=Math.floor(acc/o.vals.length); } else { vi=i % o.vals.length; } valById[o.v.id]=o.vals[vi]; });
      const nc=this.clone(c); nc.id=this.uid(); nc.qty=1;
      nc.name = textArr ? String(valById[textArr.v.id]) : (c.name+' '+(i+1));
      nc.elements = c.elements.map(e=>{ const ne={...this.clone(e), id:this.uid()}; vars.forEach(v=>{ if(v.elId===e.id && valById[v.id]!==undefined) ne[v.prop]=valById[v.id]; }); return ne; });
      vars.forEach(v=>{ if(v.elId==='bg' && valById[v.id]!==undefined) nc.bg={...nc.bg, type:'solid', color:valById[v.id]}; });
      newCards.push(nc);
    }
    this.setState({ cards:[...this.state.cards, ...newCards], modal:null });
    this.toast('Generated '+N+' cards');
  }
```

- [ ] **Step 3: Syntax-check**

Run the syntax-check helper from the plan header.
Expected: `PARSE_OK`.

- [ ] **Step 4: Self-review the engine**

Confirm by reading:
- `resolveVar` returns string arrays; range handles ascending and descending, caps at 500, rejects `step<=0`.
- `seriesCount` matches `seriesGenerate`'s N (both use `product`=∏, `zip`=max).
- In `seriesGenerate`, Product uses mixed-radix (`acc % len`, `acc = floor(acc/len)` per var, in `arrs` order); Zip uses `i % len`.
- Elements are rebuilt from the ORIGINAL `c.elements` with fresh ids, and variable values are applied by matching the original `elId` (not the new id).
- `nc.name` uses the first text variable's value, else `base + index`.

No UI yet — behavior is exercised in Task 2/3.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add series generator state and generation engine"
```

---

## Task 2: Series modal UI + entry button

**Files:**
- Modify: `index.html:996-997` (add sidebar button)
- Modify: `index.html` (add `renderSeriesModal` after `renderModal`, ~line 1152)
- Modify: `index.html:1163` (wire modal into `render`)

- [ ] **Step 1: Add the "Generate series" button**

Find (lines 996-997):

```js
        h('div',{style:{display:'flex',gap:8}},
          eBtn('table','Batch fill',()=>this.setState({modal:'batch'})),
```

Replace with (adds a full-width series button row above the Batch row):

```js
        h('div',{style:{display:'flex',gap:8}},
          eBtn('grid','Generate series',()=>this.openSeries())),
        h('div',{style:{display:'flex',gap:8}},
          eBtn('table','Batch fill',()=>this.setState({modal:'batch'})),
```

- [ ] **Step 2: Add `renderSeriesModal`**

Locate the `renderModal(){ ... }` method. It ends (~line 1152) with a line like:

```js
            h('button',{onClick:()=>this.batchGenerate(),style:{padding:'9px 18px',borderRadius:9,border:'none',background:A,color:'#fff',cursor:'pointer',fontSize:13,fontWeight:700}},'Generate deck')))));
  }
```

Immediately after that method's closing brace, insert this method verbatim:

```js
  renderSeriesModal(){
    if(this.state.modal!=='series') return null;
    const t=this.th(), h=this.h, A=this.ac(), c=this.curCard();
    const S=this.state.series;
    const targets=this.seriesTargets();
    const N=this.seriesCount();
    const over=N>150;
    const varLabel=(v)=>{ const tg=targets.find(x=>x.elId===v.elId); const p=tg&&tg.props.find(pp=>pp.prop===v.prop); return (tg?tg.label:v.elId)+' → '+(p?p.label:v.prop); };
    const modeBtn=(m,label)=>h('button',{key:m,onClick:()=>this.setState({series:{...S,mode:m}}),
      style:{padding:'5px 12px',borderRadius:7,border:'1px solid '+(S.mode===m?A:t.border),background:S.mode===m?A:t.panel,color:S.mode===m?'#fff':t.text,cursor:'pointer',fontSize:12,fontWeight:700}},label);
    const addOptions=[]; targets.forEach(tg=>tg.props.forEach(p=>addOptions.push({elId:tg.elId,prop:p.prop,label:tg.label+' → '+p.label})));
    const addSelect=h('select',{value:'',onChange:e=>{ const val=e.target.value; if(!val) return; const parts=val.split('|'); this.addSeriesVar(parts[0],parts[1]); },
      style:{background:t.input,border:'1px solid '+t.border,borderRadius:8,padding:'7px 10px',fontSize:12.5,color:t.text,cursor:'pointer',fontWeight:600}},
      [h('option',{key:'_',value:''},'+ Add variable'), ...addOptions.map((o,k)=>h('option',{key:k,value:o.elId+'|'+o.prop},o.label))]);
    const varEditor=(v)=>{
      let editor;
      if(v.prop==='text'){
        const seg=h('div',{style:{display:'flex',gap:6,marginBottom:8}},
          ['range','list'].map(mm=>h('button',{key:mm,onClick:()=>this.updateSeriesVar(v.id,{valueMode:mm}),
            style:{padding:'4px 10px',borderRadius:6,border:'1px solid '+(v.valueMode===mm?A:t.border),background:v.valueMode===mm?A+'18':t.panel,color:v.valueMode===mm?A:t.sub,cursor:'pointer',fontSize:11.5,fontWeight:700}},mm==='range'?'Range':'List')));
        const body = v.valueMode==='range'
          ? h('div',{style:{display:'flex',gap:8,alignItems:'center'}},
              h('div',{style:{flex:1}},this.field('From',this.numInput(v.from,x=>this.updateSeriesVar(v.id,{from:x}),{step:1}))),
              h('div',{style:{flex:1}},this.field('To',this.numInput(v.to,x=>this.updateSeriesVar(v.id,{to:x}),{step:1}))),
              h('div',{style:{flex:1}},this.field('Step',this.numInput(v.step,x=>this.updateSeriesVar(v.id,{step:x}),{step:1,min:0}))))
          : this.textInput(v.list.join(', '),x=>this.updateSeriesVar(v.id,{list:x.split(',').map(s=>s.trim()).filter(Boolean)}),{ph:'e.g. A, K, Q, J'});
        editor=h('div',{},seg,body);
      } else if(v.prop==='icon'){
        editor=h('div',{style:{display:'flex',flexWrap:'wrap',gap:6}},
          Object.keys(this.ICONS).map(k=>{ const on=v.list.indexOf(k)>=0;
            return h('button',{key:k,title:k,onClick:()=>this.updateSeriesVar(v.id,{list: on? v.list.filter(x=>x!==k) : [...v.list,k]}),
              style:{width:34,height:34,borderRadius:8,border:'1.5px solid '+(on?A:t.border),background:on?A+'18':t.input,cursor:'pointer',display:'flex',alignItems:'center',justifyContent:'center',padding:0}},
              this.gIcon(k,20,on?A:t.sub)); }));
      } else {
        editor=h('div',{style:{display:'flex',flexWrap:'wrap',gap:6,alignItems:'center'}},
          v.list.map((col,ci)=>h('div',{key:ci,style:{display:'flex',alignItems:'center',gap:5,background:t.input,border:'1px solid '+t.border,borderRadius:7,padding:'3px 7px'}},
            h('span',{style:{width:14,height:14,borderRadius:4,background:col,border:'1px solid '+t.border}}),
            h('span',{style:{fontSize:11,color:t.text}},col),
            h('button',{onClick:()=>this.updateSeriesVar(v.id,{list:v.list.filter((_,x)=>x!==ci)}),style:{border:'none',background:'transparent',color:t.faint,cursor:'pointer',padding:0,fontSize:14,lineHeight:1}},'×'))),
          h('label',{style:{display:'inline-flex',alignItems:'center',gap:4,cursor:'pointer',fontSize:11.5,color:A,fontWeight:700,border:'1px dashed '+A,borderRadius:7,padding:'4px 8px',position:'relative'}},
            '+ color',
            h('input',{type:'color',onChange:e=>this.updateSeriesVar(v.id,{list:[...v.list,e.target.value]}),style:{width:0,height:0,opacity:0,position:'absolute',pointerEvents:'none'}})));
      }
      const vals=this.resolveVar(v);
      return h('div',{key:v.id,style:{border:'1px solid '+t.border,borderRadius:12,padding:'12px',marginBottom:10,background:t.panel2}},
        h('div',{style:{display:'flex',alignItems:'center',gap:8,marginBottom:10}},
          h('div',{style:{fontSize:12.5,fontWeight:700,color:t.text,flex:1}},varLabel(v)),
          h('button',{onClick:()=>this.removeSeriesVar(v.id),title:'Remove',style:{border:'none',background:'transparent',color:t.faint,cursor:'pointer',padding:2,display:'flex'}},this.uIcon('trash',15,'currentColor'))),
        editor,
        h('div',{style:{fontSize:11,color:t.faint,marginTop:9}},'values: '+(vals.length? vals.slice(0,24).join(', ')+(vals.length>24?' …':'') : '—')));
    };
    return h('div',{style:{position:'fixed',inset:0,background:'rgba(15,20,18,.5)',zIndex:50,display:'flex',alignItems:'center',justifyContent:'center',padding:24},onClick:()=>this.setState({modal:null})},
      h('div',{onClick:e=>e.stopPropagation(),style:{width:600,maxWidth:'100%',maxHeight:'90vh',display:'flex',flexDirection:'column',background:t.panel,borderRadius:16,boxShadow:'0 30px 80px rgba(0,0,0,.4)',overflow:'hidden',border:'1px solid '+t.border}},
        h('div',{style:{padding:'18px 22px',borderBottom:'1px solid '+t.border,display:'flex',alignItems:'center',gap:10}},
          h('div',{style:{width:34,height:34,borderRadius:9,background:A+'18',color:A,display:'flex',alignItems:'center',justifyContent:'center'}},this.uIcon('grid',18,'currentColor')),
          h('div',{style:{flex:1}},
            h('div',{style:{fontWeight:800,fontSize:16,color:t.text}},'Generate series'),
            h('div',{style:{fontSize:12,color:t.sub,marginTop:1}},'Vary properties of "'+(c?c.name:'this card')+'" across a range or list')),
          h('div',{style:{display:'flex',gap:6,marginRight:8}},modeBtn('zip','Zip'),modeBtn('product','Product')),
          h('button',{onClick:()=>this.setState({modal:null}),style:{border:'none',background:'transparent',color:t.sub,cursor:'pointer',padding:4}},this.uIcon('close',20,'currentColor'))),
        h('div',{style:{padding:'16px 22px',overflowY:'auto',flex:1}},
          h('div',{style:{display:'flex',alignItems:'center',gap:10,marginBottom:12}},
            h('div',{style:{fontSize:11,fontWeight:700,letterSpacing:'.9px',textTransform:'uppercase',color:t.sub,flex:1}},'Variables'),
            addSelect),
          S.vars.length? S.vars.map(varEditor)
            : h('div',{style:{fontSize:12.5,color:t.sub,background:t.input,borderRadius:10,padding:'14px',textAlign:'center'}},'Add a variable to vary a number, icon, or color across the deck.')),
        h('div',{style:{padding:'14px 22px',borderTop:'1px solid '+t.border,display:'flex',alignItems:'center',gap:12}},
          h('div',{style:{flex:1,fontSize:13,fontWeight:700,color:over?'#d0432f':t.text}}, over? ('Too many cards ('+N+') — max 150') : ('→ '+N+' card'+(N===1?'':'s')+' will be generated')),
          h('button',{onClick:()=>this.setState({modal:null}),style:{padding:'9px 14px',borderRadius:9,border:'1px solid '+t.border,background:t.panel,color:t.text,cursor:'pointer',fontSize:13,fontWeight:600}},'Cancel'),
          h('button',{onClick:()=>this.seriesGenerate(),disabled:N===0||over,style:{padding:'9px 18px',borderRadius:9,border:'none',background:(N===0||over)?t.border:A,color:'#fff',cursor:(N===0||over)?'default':'pointer',fontSize:13,fontWeight:700,opacity:(N===0||over)?.6:1}},'Generate')));
  }
```

- [ ] **Step 3: Wire the modal into `render`**

Find (line 1163):

```js
      this.renderModal(),
```

Replace with:

```js
      this.renderModal(),
      this.renderSeriesModal(),
```

- [ ] **Step 4: Syntax-check**

Run the syntax-check helper. Expected: `PARSE_OK`.

- [ ] **Step 5: Verify in browser**

Open `index.html`. On the default sample deck:
- The Export panel shows a **Generate series** button; click it → the modal opens (base card name shown).
- Click **+ Add variable** → pick a `Text · "…" → Number/Text` target → a Range editor appears (From 1, To 5, Step 1); footer shows `→ 5 cards will be generated`.
- Set To=10 → footer shows `→ 10 cards`. Add an `Icon → Icon` variable, toggle 3 icons in the grid; keep **Zip** → footer still `→ 10 cards`. Add a `Background → Background color` variable, add 2–3 colors.
- Click **Generate** → modal closes, N new cards appended to the deck; numbers run 1–10, icons/background cycle; each card is named after its number.
- Re-open, switch to **Product**, set number 1→4 and 4 icons → footer `→ 16 cards`; Generate → 16 cards.
- Ctrl+Z once → the whole generated run disappears in one step.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add Generate series modal UI and entry button"
```

---

## Task 3: Full verification pass

**Files:** none (manual verification against the spec).

- [ ] **Step 1: Run the full browser checklist**

Open `index.html` and confirm every item from the spec's Verification section:

1. Base card with a number text, an icon, and a background — all appear as targets in **+ Add variable**.
2. Number variable range 1→10 step 1 → count shows 10.
3. Icon variable (3 icons) + Background color (3 colors), **Zip** → count stays 10; Generate → 10 cards, numbers 1–10, icons/colors cycling every 3.
4. **Product** with numbers 1→4 and 4 icons → count shows 16; Generate → 16 cards covering all combinations; each named after its number.
5. Product that exceeds 150 (e.g. 13×13 = 169) → **Generate** is disabled and the footer shows the "Too many cards" warning; forcing it does nothing.
6. Undo once → the whole generated run disappears in one step.
7. Batch generate (`{{field}}` + CSV) still works unchanged; export PDF/PNG and deck thumbnails render normally.
8. Remove-variable (trash) and switching a text variable between Range/List both work; List mode accepts comma-separated values.

- [ ] **Step 2: If any check fails**

Use the `systematic-debugging` skill; fix in `index.html`; re-run the checklist and the syntax-check. Commit fixes with a descriptive message. If all pass, the feature is complete — proceed to `finishing-a-development-branch`.

---

## Self-Review (author)

- **Spec coverage:** state model `series{mode,vars}` (T1 S1); `elId/prop/valueMode/from/to/step/list` variable shape (T1 S2 `addSeriesVar`); target props per element type incl. `bg` (T1 `seriesTargets`); `resolveVar` range+list (T1); `seriesCount` zip/product (T1); `seriesGenerate` with N cap 150, single `pushHistory`, name-from-text-var, apply-by-original-elId, bg solid (T1); guided picker + per-type editors (range/list, icon grid, color swatches) + live count + value chips + zip/product toggle (T2 `renderSeriesModal`); entry button (T2 S1); render wiring (T2 S3). All spec sections mapped.
- **Placeholder scan:** none — every code step is complete.
- **Type/name consistency:** `series`, `mode`, `vars`, `elId`, `prop`, `valueMode`, `from/to/step/list`, and methods `openSeries/seriesTargets/addSeriesVar/updateSeriesVar/removeSeriesVar/resolveVar/seriesCount/seriesGenerate/renderSeriesModal` are used identically across Task 1 and Task 2. Reused existing helpers `curCard/uid/clone/pushHistory/toast/th/ac/h/field/numInput/textInput/gIcon/uIcon/ICONS` all exist in `index.html`.
