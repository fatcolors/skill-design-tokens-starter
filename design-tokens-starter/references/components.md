# Building token-bound components

Once the token tiers exist (Brand → Alias → Mapped → Typography), components should consume tokens for **every** value — never raw hex, **and never raw numbers** for layout:

- **Colors** → **Mapped** tokens, via `setBoundVariableForPaint` (fills, strokes).
- **Type** → bound **text styles** applied with `await textNode.setTextStyleIdAsync(style.id)` (the styles are themselves bound to **Typography** vars). **Mandatory — never leave a raw `fontSize`/`fontName` on a component's text.**
- **Layout dimensions** → **Alias** `spacing/*` (gap + padding), `radius/*` (corner radius), `stroke/*` (border width), via `node.setBoundVariable(...)`.

That's what makes a component inherit Light/Dark theming and stay in sync when a base color, *the spacing scale, or the type scale* changes: every property is a `VARIABLE_ALIAS` chain back to Brand/Typography. A hardcoded `itemSpacing = 8`, `cornerRadius = 8`, **or `fontSize = 14`** is just as broken as a hardcoded hex fill — it silently drops out of the system.

This is Phase 3 of the build (tokens are Phase 1–2). Build components **after** tokens, one component at a time, validating with a screenshot.

## The pattern

1. **Define variant axes.** A component is a Cartesian product of properties, e.g. Button = `Type` × `State`. Keep the matrix under ~30 variants; if `Size × Type × State` explodes, split or use component properties instead of variants.
2. **Build one `COMPONENT` per combination**, naming it `"Axis=Value, Axis=Value"` (Figma parses this into variant properties).
3. **Bind every visual property to a Mapped token** with `setBoundVariableForPaint` (fills, strokes). Transparent = `fills = []`.
3b. **Bind every layout dimension to an Alias `spacing/`, `radius/` or `stroke/` token** with `node.setBoundVariable(field, variable)` — `itemSpacing`/`counterAxisSpacing`, `paddingLeft|Right|Top|Bottom`, the four `*Radius` fields, and `strokeWeight`. Set a base number first (the variable overrides it), then bind. Never leave a raw layout number on a component.
3c. **Apply a bound text style to every TEXT node** with `await textNode.setTextStyleIdAsync(style.id)` — load the style's font first (`await figma.loadFontAsync(style.fontName)`). Pick the style matching the role (`Paragraph Small`/`Caption` for chips/badges, `Paragraph Medium` for body, `H4`–`H6` for headings). Never leave a raw `fontSize`/`fontName`. If the needed weight has no style yet (e.g. a Medium label), create the bound style first — don't hardcode. (`setTextStyleIdAsync` only sets the style's size/weight/line-height/family; the text's *color* is still a separate Mapped fill from step 3, so set both.)
4. **`combineAsVariants`** the components into a set, then **grid-layout** them (they stack at 0,0 otherwise).
5. **Add component properties** — a `TEXT` property for labels, `BOOLEAN` for optional elements, `INSTANCE_SWAP` for icons — and link them to child nodes.

## Binding helper

```js
const cols = await figma.variables.getLocalVariableCollectionsAsync();
const mapped = cols.find(c=>c.name==="Mapped");
const alias  = cols.find(c=>c.name==="Alias");
const allVars = await figma.variables.getLocalVariablesAsync();
const mvars = {}; allVars.filter(v=>v.variableCollectionId===mapped.id).forEach(v=>{mvars[v.name]=v;});
const dvars = {}; allVars.filter(v=>v.variableCollectionId===alias.id).forEach(v=>{dvars[v.name]=v;}); // spacing/ radius/ stroke/ live here

// COLOR: setBoundVariableForPaint returns a NEW paint — assign it, don't mutate in place
function bp(name){ return figma.variables.setBoundVariableForPaint({type:'SOLID',color:{r:0,g:0,b:0}}, 'color', mvars[name]); }
// usage: node.fills = [bp('surface/brand')]; node.strokes = [bp('border/focus')];

// DIMENSION: bind a layout field to a spacing/ radius/ stroke/ token
function bd(node, field, name){ node.setBoundVariable(field, dvars[name]); }
// gap + padding (one call each):
function bindPadding(node, token){ for(const f of ['paddingLeft','paddingRight','paddingTop','paddingBottom']) bd(node, f, token); }
function bindGap(node, token){ bd(node, 'itemSpacing', token); }
function bindRadius(node, token){ for(const f of ['topLeftRadius','topRightRadius','bottomLeftRadius','bottomRightRadius']) bd(node, f, token); }
function bindStroke(node, token){ bd(node, 'strokeWeight', token); }
// usage: bindPadding(card,'spacing/lg'); bindGap(card,'spacing/sm'); bindRadius(card,'radius/lg'); bindStroke(card,'stroke/thin');
```

> Padding/radius can differ per side — call `bd(node,'paddingTop','spacing/md')` etc. individually when a component needs asymmetric values. `strokeWeight` binds the whole stroke; per-side stroke fields (`strokeTopWeight`…) also exist if needed.

## Gotchas that bit during the Button build

- **`createComponent()` starts at 100×100.** Setting `layoutMode='HORIZONTAL'` hugs the *primary* (width) axis but the *counter* (height) axis can stay fixed at 100 — buttons come out tall. Fix: after layout, set **both** `primaryAxisSizingMode='AUTO'` and `counterAxisSizingMode='AUTO'` to hug. (Or set a fixed height deliberately with `counterAxisSizingMode='FIXED'`.)
- **Transparent fill / no border:** `fills = []` / `strokes = []`. Don't bind a token you don't want shown.
- **Focus ring:** give the Focused variant a `strokeWeight` of 2 with `strokes=[bp('border/focus')]` and `strokeAlign='INSIDE'` over the default fill.
- **`combineAsVariants` stacks variants at (0,0).** Reposition them in a grid afterward (`child.x`/`child.y`) and `resize()` the set to wrap them. Order of children = order you created the components, so build row-by-row (all of Type A's states, then Type B's…).
- **`setPluginData` is unavailable in `use_figma`** — use `setSharedPluginData` or just track returned node IDs.
- **Text/label property:** `const pid = set.addComponentProperty('Label','TEXT','Button')`, then for each variant link its text node: `textNode.componentPropertyReferences = { characters: pid }`.

## Worked example — Button (15 variants)

`Type` (Primary/Secondary/Tertiary) × `State` (Default/Hover/Pressed/Focused/Disabled), every color a Mapped token:

```js
// Resolve the bound text style for the label, and load its font before applying.
// (No Medium style in the default ramp — use Paragraph Small, or create a "Label" Medium style first.)
const styles = await figma.getLocalTextStylesAsync();
const labelStyle = styles.find(s=>s.name==='Paragraph Small');
await figma.loadFontAsync(labelStyle.fontName);
// ...build mvars + dvars + bp()/bd()/bindGap()/bindRadius()/bindStroke() as above...
const T = {
  Primary: {
    Default:  {fill:"surface/brand",         text:"text/on-brand", border:null,           bw:0},
    Hover:    {fill:"surface/brand-hover",   text:"text/on-brand", border:null,           bw:0},
    Pressed:  {fill:"surface/brand-pressed", text:"text/on-brand", border:null,           bw:0},
    Focused:  {fill:"surface/brand",         text:"text/on-brand", border:"border/focus", bw:2},
    Disabled: {fill:"surface/disabled",      text:"text/disabled", border:null,           bw:0}
  },
  Secondary: {
    Default:  {fill:"surface/primary",   text:"text/primary",  border:"border/default",  bw:1},
    Hover:    {fill:"surface/secondary", text:"text/primary",  border:"border/default",  bw:1},
    Pressed:  {fill:"surface/tertiary",  text:"text/primary",  border:"border/default",  bw:1},
    Focused:  {fill:"surface/primary",   text:"text/primary",  border:"border/focus",    bw:2},
    Disabled: {fill:"surface/primary",   text:"text/disabled", border:"border/disabled", bw:1}
  },
  Tertiary: {
    Default:  {fill:null,                   text:"text/brand",    border:null,           bw:0},
    Hover:    {fill:"surface/brand-subtle", text:"text/brand",    border:null,           bw:0},
    Pressed:  {fill:"surface/secondary",    text:"text/brand",    border:null,           bw:0},
    Focused:  {fill:null,                   text:"text/brand",    border:"border/focus", bw:2},
    Disabled: {fill:null,                   text:"text/disabled", border:null,           bw:0}
  }
};
const comps=[];
for(const type of ["Primary","Secondary","Tertiary"]){
  for(const state of ["Default","Hover","Pressed","Focused","Disabled"]){
    const s=T[type][state];
    const c=figma.createComponent();
    c.name=`Type=${type}, State=${state}`;
    c.layoutMode='HORIZONTAL';
    c.primaryAxisAlignItems='CENTER'; c.counterAxisAlignItems='CENTER';
    c.primaryAxisSizingMode='AUTO';   c.counterAxisSizingMode='AUTO';   // hug BOTH axes
    // base numbers (the bound variables override these), then bind every dimension to a token
    c.paddingTop=8; c.paddingBottom=8; c.paddingLeft=16; c.paddingRight=16;
    c.itemSpacing=8; c.cornerRadius=8;
    bd(c,'paddingTop','spacing/sm'); bd(c,'paddingBottom','spacing/sm');
    bd(c,'paddingLeft','spacing/lg'); bd(c,'paddingRight','spacing/lg');
    bindGap(c,'spacing/sm');
    bindRadius(c,'radius/md');
    c.fills = s.fill ? [bp(s.fill)] : [];
    if(s.border){ c.strokes=[bp(s.border)]; c.strokeWeight=s.bw; c.strokeAlign='INSIDE'; bindStroke(c, s.bw>=2?'stroke/thick':'stroke/thin'); } else { c.strokes=[]; }
    const t=figma.createText(); t.characters='Button';
    t.fills=[bp(s.text)];                       // color: Mapped token
    await t.setTextStyleIdAsync(labelStyle.id); // size/weight/line-height: bound text style (NOT a raw fontSize)
    c.appendChild(t);
    comps.push(c);
  }
}
const set=figma.combineAsVariants(comps, figma.currentPage);
set.name="Button";
let maxW=0,maxH=0; set.children.forEach(c=>{maxW=Math.max(maxW,c.width);maxH=Math.max(maxH,c.height);});
const PAD=32, GX=maxW+28, GY=maxH+24;
set.children.forEach((c,i)=>{ c.x=PAD+(i%5)*GX; c.y=PAD+Math.floor(i/5)*GY; });
set.resize(PAD*2+4*GX+maxW, PAD*2+2*GY+maxH);
const pid=set.addComponentProperty('Label','TEXT','Button');
set.children.forEach(v=>{ const tx=v.findOne(n=>n.type==='TEXT'); if(tx) tx.componentPropertyReferences={characters:pid}; });
return { setId:set.id };
```

## Typography in components

**Every** TEXT node in a component must carry a bound **text style** (Hero, H1–H6, Paragraph L/M/S, Caption) — not just text-heavy ones. The styles already reference the Typography vars, so applying one makes the text responsive (Desktop/Mobile) and lets font/size changes propagate; a hardcoded `fontSize`/`fontName` silently drops out of that system, exactly like a raw `itemSpacing`.

```js
const styles = await figma.getLocalTextStylesAsync();
const style = styles.find(s=>s.name==='Paragraph Small');   // role-matched
await figma.loadFontAsync(style.fontName);                  // load BEFORE applying
await textNode.setTextStyleIdAsync(style.id);               // size/weight/line-height/family — all bound
textNode.fills=[bp('text/primary')];                        // color is still a separate Mapped fill — set both
```

Role guide: `Caption` (12) / `Paragraph Small` (14) for chips, badges, helper text; `Paragraph Medium` (16) for body and inputs; `H4`–`H6` for card/section titles; `H1`–`Hero` for page headers. **Gotcha:** the default ramp ships only Regular and Semi Bold weights — if a component needs a Medium label, create a bound "Label" text style (Medium) first rather than hardcoding `fontName`. `setTextStyleIdAsync` is async and must be `await`ed, and the style's font must be loaded first or it throws `unloaded font`.

## Suggested component order (atoms → molecules)

Button → Input / Checkbox / Radio / Toggle → Badge / Tag → Card → Modal / Navbar. Build atoms first since molecules compose them. Every new component just calls `bp('<mapped-token>')` for its colors, so **everything stays connected to the token system automatically** — no detached hex values.
