# Inline Text Editing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let the user double-click a text element on the canvas and edit it in place via a transparent textarea overlay.

**Architecture:** All changes live in the single-file React app `index.html`. A new `editEl` state field marks the text element being edited. When set, `renderElDOM` swaps that element's display `<div>` for a transparent `<textarea>` matching its typography, wired to the existing `updateSelEl` data flow. A `componentDidUpdate` hook auto-clears `editEl` whenever selection moves away, centralizing exit for card switches, deletes, undo/redo, and empty-canvas clicks.

**Tech Stack:** React 18 (UMD, class component, `h = React.createElement`), single HTML file, no build step, no test framework.

**Testing note:** This project has no automated test harness and no build step. Per-task verification is a **manual browser check**: open `index.html` directly in a browser and exercise the described interaction. Do not add a test framework — that is out of scope (YAGNI). Reference spec: `docs/superpowers/specs/2026-07-06-inline-text-editing-design.md`.

---

## File Structure

- **Modify only:** `index.html`
  - `state` init (~line 110-117): add `editEl:null`.
  - `componentDidUpdate` (new, after `componentWillUnmount` ~line 138): auto-clear `editEl`.
  - `enterEdit` (new method, near `updateSelEl` ~line 196): enter edit mode.
  - `onElPointerDown` (~line 385): early-return guard while editing (place caret, don't drag).
  - `renderElDOM` (~line 992-1029): textarea overlay when editing, `onDoubleClick` wiring, editing cursor, hide handles while editing.

---

## Task 1: State field + auto-clear on selection change

**Files:**
- Modify: `index.html:112` (state init)
- Modify: `index.html` (add `componentDidUpdate` after `componentWillUnmount`, ~line 138)

- [ ] **Step 1: Add `editEl` to state**

Find (line 112):

```js
    cards:[], selCard:null, selEl:null,
```

Replace with:

```js
    cards:[], selCard:null, selEl:null, editEl:null,
```

- [ ] **Step 2: Add `componentDidUpdate` to auto-clear `editEl`**

Locate `componentWillUnmount(){ ... }` (ends ~line 138). Immediately after its closing brace, add this method:

```js
  componentDidUpdate(){
    if(this.state.editEl && this.state.editEl!==this.state.selEl){
      this.setState({ editEl:null });
    }
  }
```

This is the single exit point for edit mode when selection moves: card switch (`selectCard`), delete (`deleteEl` sets `selEl:null`), template apply, import, and undo/redo (`fixSel` reconciles `selEl`) all change `selEl` away from `editEl`, so `editEl` clears automatically. The guard (`editEl` truthy AND differs from `selEl`) prevents an update loop — once cleared, the condition is false.

- [ ] **Step 3: Verify no regression in browser**

Open `index.html` in a browser.
Expected: app loads normally, sample deck renders, selecting/moving/resizing elements still works exactly as before. No visible change yet (this task is infrastructure).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add editEl state and auto-clear on selection change"
```

---

## Task 2: Inline edit — enter, textarea overlay, drag guard, exit

**Files:**
- Modify: `index.html` (add `enterEdit` near `updateSelEl`, ~line 196)
- Modify: `index.html:385-390` (`onElPointerDown`)
- Modify: `index.html:994-1000` (`renderElDOM` wrap cursor + text branch)
- Modify: `index.html:1016-1028` (`renderElDOM` selection/handles block)
- Modify: `index.html:1029` (`renderElDOM` wrap `onDoubleClick`)

- [ ] **Step 1: Add the `enterEdit` method**

Find `updateSelEl(patch, opts){ ... }` (ends ~line 196). Immediately after its closing brace, add:

```js
  enterEdit(el){
    if(!el || el.type!=='text') return;
    if(!this.curCard()) return;
    this.pushHistory('text');   // single undo entry for the whole edit session
    this._focusEdit=true;       // one-shot flag consumed by the textarea ref
    this.setState({ selEl:el.id, editEl:el.id });
  }
```

- [ ] **Step 2: Guard `onElPointerDown` so editing places the caret instead of dragging**

Find (lines 385-390):

```js
  onElPointerDown(e, el){
    if(e.button!==0) return;
    e.stopPropagation();
    this.setState({ selEl:el.id });
    this.pushHistory('move_'+el.id);
    this.drag={ mode:'move', sx:e.clientX, sy:e.clientY, ox:el.x, oy:el.y };
  }
```

Replace with:

```js
  onElPointerDown(e, el){
    if(e.button!==0) return;
    if(this.state.editEl===el.id) return;   // editing: let the pointer place the caret
    e.stopPropagation();
    this.setState({ selEl:el.id });
    this.pushHistory('move_'+el.id);
    this.drag={ mode:'move', sx:e.clientX, sy:e.clientY, ox:el.x, oy:el.y };
  }
```

The guard returns before `stopPropagation`, so the native textarea (which has `pointerEvents:auto`) receives the pointer and positions the caret.

- [ ] **Step 3: Make the wrap cursor `text` while editing**

Find (lines 994-996):

```js
    const wrap={ position:'absolute', left:(el.x+B)*s, top:(el.y+B)*s, width:el.w*s, height:el.h*s,
      transform:'rotate('+(el.rotation||0)+'deg)', transformOrigin:'center', opacity:el.opacity==null?1:el.opacity,
      cursor:interactive?'move':'default' };
```

Replace the `cursor` line (last line of that object) so it reads:

```js
    const wrap={ position:'absolute', left:(el.x+B)*s, top:(el.y+B)*s, width:el.w*s, height:el.h*s,
      transform:'rotate('+(el.rotation||0)+'deg)', transformOrigin:'center', opacity:el.opacity==null?1:el.opacity,
      cursor:interactive?(this.state.editEl===el.id?'text':'move'):'default' };
```

- [ ] **Step 4: Render the textarea overlay when editing**

Find the text branch (lines 998-1000):

```js
    if(el.type==='text'){
      inner=h('div',{style:{width:'100%',height:'100%',fontFamily:el.font,fontSize:el.size*s,fontWeight:el.weight,fontStyle:el.italic?'italic':'normal',
        color:el.color,textAlign:el.align,lineHeight:el.lineH||1.2,letterSpacing:(el.letter||0)*s/3,whiteSpace:'pre-wrap',wordBreak:'break-word',overflow:'hidden',pointerEvents:'none',display:'block'}},el.text);
    } else if(el.type==='image'){
```

Replace with:

```js
    if(el.type==='text'){
      if(interactive && this.state.editEl===el.id){
        inner=h('textarea',{
          value:el.text,
          onChange:e=>this.updateSelEl({text:e.target.value},{key:'text'}),
          onKeyDown:e=>{ if(e.key==='Escape'){ e.stopPropagation(); e.preventDefault(); this.setState({editEl:null}); } },
          onBlur:()=>this.setState({editEl:null}),
          ref:(n)=>{ if(n && this._focusEdit){ this._focusEdit=false; n.focus(); const L=n.value.length; try{ n.setSelectionRange(L,L); }catch(_){} } },
          style:{width:'100%',height:'100%',fontFamily:el.font,fontSize:el.size*s,fontWeight:el.weight,fontStyle:el.italic?'italic':'normal',
            color:el.color,textAlign:el.align,lineHeight:el.lineH||1.2,letterSpacing:(el.letter||0)*s/3,whiteSpace:'pre-wrap',wordBreak:'break-word',
            overflow:'hidden',background:'transparent',border:0,padding:0,margin:0,resize:'none',display:'block',boxSizing:'border-box',pointerEvents:'auto'}});
      } else {
        inner=h('div',{style:{width:'100%',height:'100%',fontFamily:el.font,fontSize:el.size*s,fontWeight:el.weight,fontStyle:el.italic?'italic':'normal',
          color:el.color,textAlign:el.align,lineHeight:el.lineH||1.2,letterSpacing:(el.letter||0)*s/3,whiteSpace:'pre-wrap',wordBreak:'break-word',overflow:'hidden',pointerEvents:'none',display:'block'}},el.text);
      }
    } else if(el.type==='image'){
```

The textarea copies the display div's typography exactly (font, size `el.size*s`, weight, italic, color, align, lineHeight, letterSpacing) so the text stays WYSIWYG, and zeroes out chrome (transparent background, no border/padding/margin, no resize). The inline `ref` is called with `null` then the node on every re-render; the `this._focusEdit` flag makes focus + caret-to-end happen exactly once (on enter), so mid-text typing does not reset the caret.

- [ ] **Step 5: Hide resize/rotate handles while editing (keep the selection border)**

Find the selection block (lines 1016-1028):

```js
    if(sel){
      children.push(h('div',{key:'sel',style:{position:'absolute',inset:-1,border:'1.5px solid '+A,borderRadius:el.type==='shape'&&el.shape==='circle'?'50%':3,pointerEvents:'none'}}));
      ['nw','ne','sw','se','n','s','e','w'].forEach(cn=>{
        const pos={nw:[0,0],n:[50,0],ne:[100,0],e:[100,50],se:[100,100],s:[50,100],sw:[0,100],w:[0,50]}[cn];
        const curs={nw:'nwse-resize',se:'nwse-resize',ne:'nesw-resize',sw:'nesw-resize',n:'ns-resize',s:'ns-resize',e:'ew-resize',w:'ew-resize'}[cn];
        children.push(h('div',{key:cn,onPointerDown:e=>this.onHandleDown(e,el,cn),
          style:{position:'absolute',left:pos[0]+'%',top:pos[1]+'%',width:10,height:10,marginLeft:-5,marginTop:-5,background:'#fff',border:'1.5px solid '+A,borderRadius:3,cursor:curs,zIndex:3}}));
      });
      children.push(h('div',{key:'rot',onPointerDown:e=>this.onRotateDown(e,el),
        style:{position:'absolute',left:'50%',top:-22,marginLeft:-7,width:14,height:14,background:'#fff',border:'1.5px solid '+A,borderRadius:'50%',cursor:'grab',zIndex:3,display:'flex',alignItems:'center',justifyContent:'center'}},
        this.uIcon('rotate',9,A,2)));
      children.push(h('div',{key:'rl',style:{position:'absolute',left:'50%',top:-11,width:1,height:11,background:A,marginLeft:-0.5,pointerEvents:'none'}}));
    }
```

Replace with (wrap the handles + rotate in an `editing` guard; the border stays unconditional):

```js
    if(sel){
      children.push(h('div',{key:'sel',style:{position:'absolute',inset:-1,border:'1.5px solid '+A,borderRadius:el.type==='shape'&&el.shape==='circle'?'50%':3,pointerEvents:'none'}}));
      if(this.state.editEl!==el.id){
        ['nw','ne','sw','se','n','s','e','w'].forEach(cn=>{
          const pos={nw:[0,0],n:[50,0],ne:[100,0],e:[100,50],se:[100,100],s:[50,100],sw:[0,100],w:[0,50]}[cn];
          const curs={nw:'nwse-resize',se:'nwse-resize',ne:'nesw-resize',sw:'nesw-resize',n:'ns-resize',s:'ns-resize',e:'ew-resize',w:'ew-resize'}[cn];
          children.push(h('div',{key:cn,onPointerDown:e=>this.onHandleDown(e,el,cn),
            style:{position:'absolute',left:pos[0]+'%',top:pos[1]+'%',width:10,height:10,marginLeft:-5,marginTop:-5,background:'#fff',border:'1.5px solid '+A,borderRadius:3,cursor:curs,zIndex:3}}));
        });
        children.push(h('div',{key:'rot',onPointerDown:e=>this.onRotateDown(e,el),
          style:{position:'absolute',left:'50%',top:-22,marginLeft:-7,width:14,height:14,background:'#fff',border:'1.5px solid '+A,borderRadius:'50%',cursor:'grab',zIndex:3,display:'flex',alignItems:'center',justifyContent:'center'}},
          this.uIcon('rotate',9,A,2)));
        children.push(h('div',{key:'rl',style:{position:'absolute',left:'50%',top:-11,width:1,height:11,background:A,marginLeft:-0.5,pointerEvents:'none'}}));
      }
    }
```

- [ ] **Step 6: Wire `onDoubleClick` on the element wrap**

Find (line 1029):

```js
    return h('div',{key:el.id, onPointerDown: interactive?(e=>this.onElPointerDown(e,el)):undefined, style:wrap}, children);
```

Replace with:

```js
    return h('div',{key:el.id, onPointerDown: interactive?(e=>this.onElPointerDown(e,el)):undefined,
      onDoubleClick: interactive?(e=>this.enterEdit(el)):undefined, style:wrap}, children);
```

`enterEdit` guards `el.type!=='text'`, so double-clicking non-text elements is a no-op.

- [ ] **Step 7: Verify inline editing in browser**

Open `index.html`. On a card with a text element:
- Double-click a text element → a caret appears in place; the text is editable inline with the same font/size/color/alignment as displayed (WYSIWYG).
- Type mid-text → the caret stays where you clicked (does not jump to end on each keystroke).
- Press Enter → inserts a newline (does not exit).
- Press Escape → edit mode exits, changes kept.
- Double-click a non-text element (icon/shape/image) → nothing happens.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Add inline text editing via double-click on canvas"
```

---

## Task 3: Full verification pass

**Files:** none (manual verification of the completed feature against the spec).

- [ ] **Step 1: Run the full browser checklist**

Open `index.html` and confirm every item from the spec's Verification section:

1. Double-click a text element → typing edits in place; multi-line (Enter) works.
2. Escape exits edit, keeping changes.
3. Blur (click another element) exits edit, keeping changes.
4. Clicking empty canvas exits edit (selection clears → `componentDidUpdate` clears `editEl`).
5. ⌘Z / Ctrl+Z reverts the **whole** edit session in one step (thanks to the single `pushHistory('text')` on enter).
6. While editing, Delete/Backspace and ⌘D do **not** delete or duplicate the element (focus is in the textarea, so `onKey` early-returns on `TEXTAREA`).
7. While editing, pressing pointer down on the text places the caret; it does **not** start a drag.
8. Resize/rotate handles are hidden while editing; the selection border remains.
9. Export (PDF/PNG) and deck thumbnails render normally with no textarea (they use `interactive=false`).
10. Switching cards while editing exits edit mode.

- [ ] **Step 2: If any check fails**

Use the `systematic-debugging` skill; fix in `index.html`; re-run the checklist. Commit fixes with a descriptive message. If all pass, the feature is complete — proceed to the `finishing-a-development-branch` skill to decide on merge/PR.

---

## Self-Review (author)

- **Spec coverage:** state `editEl` (T1), enter via double-click (T2 S6), textarea overlay approach A (T2 S4), exit on Escape/blur/empty-click (T2 S4 + T1 componentDidUpdate), single-undo (T2 S1 `pushHistory('text')`), keyboard/drag guards (T2 S2 + existing `onKey` TEXTAREA guard), hide handles (T2 S5), edge cases: delete/card-change clear via componentDidUpdate (T1), export/thumbnails unaffected via `interactive` guard (T2 S4 condition). All spec sections mapped.
- **Placeholder scan:** none — every code step shows full code.
- **Type/name consistency:** `editEl`, `enterEdit`, `_focusEdit`, `updateSelEl(patch,{key:'text'})`, `pushHistory('text')` used identically across tasks.
