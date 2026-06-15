# Building token-bound components

Once the token tiers exist (Brand → Alias → Mapped → Responsive), components should consume **only Mapped** color tokens and **Responsive**/text-style typography — never raw hex. That's what makes a component inherit Light/Dark theming and stay in sync when a base color changes: every visual property is a `VARIABLE_ALIAS` chain back to Brand.

This is Phase 3 of the build (tokens are Phase 1–2). Build components **after** tokens, one component at a time, validating with a screenshot.

## The pattern

1. **Define variant axes.** A component is a Cartesian product of properties, e.g. Button = `Type` × `State`. Keep the matrix under ~30 variants; if `Size × Type × State` explodes, split or use component properties instead of variants.
2. **Build one `COMPONENT` per combination**, naming it `"Axis=Value, Axis=Value"` (Figma parses this into variant properties).
3. **Bind every visual property to a Mapped token** with `setBoundVariableForPaint` (fills, strokes). Transparent = `fills = []`.
4. **`combineAsVariants`** the components into a set, then **grid-layout** them (they stack at 0,0 otherwise).
5. **Add component properties** — a `TEXT` property for labels, `BOOLEAN` for optional elements, `INSTANCE_SWAP` for icons — and link them to child nodes.

## Binding helper

```js
const mapped = (await figma.variables.getLocalVariableCollectionsAsync()).find(c=>c.name==="Mapped");
const mvars = {}; (await figma.variables.getLocalVariablesAsync())
  .filter(v=>v.variableCollectionId===mapped.id).forEach(v=>{mvars[v.name]=v;});
// setBoundVariableForPaint returns a NEW paint — assign it, don't mutate in place
function bp(name){ return figma.variables.setBoundVariableForPaint({type:'SOLID',color:{r:0,g:0,b:0}}, 'color', mvars[name]); }
// usage: node.fills = [bp('surface/brand')]; node.strokes = [bp('border/focus')];
```

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
await figma.loadFontAsync({family:'Inter',style:'Medium'});
// ...build mvars + bp() as above...
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
    c.paddingTop=10; c.paddingBottom=10; c.paddingLeft=18; c.paddingRight=18;
    c.itemSpacing=8; c.cornerRadius=8;
    c.fills = s.fill ? [bp(s.fill)] : [];
    if(s.border){ c.strokes=[bp(s.border)]; c.strokeWeight=s.bw; c.strokeAlign='INSIDE'; } else { c.strokes=[]; }
    const t=figma.createText(); t.fontName={family:'Inter',style:'Medium'}; t.fontSize=14; t.characters='Button';
    t.fills=[bp(s.text)]; c.appendChild(t);
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

For text-heavy components, apply the bound **text styles** (Hero, H1–H6, Paragraph…) instead of hardcoding size/line-height — they already reference the Responsive vars, so the text becomes responsive (Desktop/Mobile) and font changes propagate. Apply a style: `textNode.setTextStyleIdAsync(style.id)` after loading the style's font.

## Suggested component order (atoms → molecules)

Button → Input / Checkbox / Radio / Toggle → Badge / Tag → Card → Modal / Navbar. Build atoms first since molecules compose them. Every new component just calls `bp('<mapped-token>')` for its colors, so **everything stays connected to the token system automatically** — no detached hex values.
