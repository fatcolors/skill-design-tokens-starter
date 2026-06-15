# Generation Recipe — proven `use_figma` snippets

Every snippet below ran successfully end-to-end. Adapt the data, keep the shapes. All run via the `use_figma` tool with `skillNames: "figma-use,figma-generate-library"` and the file's `fileKey`. Each script is auto-wrapped in async — use top-level `await` and `return`; do not wrap in an IIFE or call `closePlugin`.

General helpers used throughout:

```js
function hexToRgb(hex){
  const h = hex.replace('#','');
  return { r:parseInt(h.slice(0,2),16)/255, g:parseInt(h.slice(2,4),16)/255, b:parseInt(h.slice(4,6),16)/255 };
}
```

## Step 0 — Inspect (read-only)

```js
const collections = await figma.variables.getLocalVariableCollectionsAsync();
const cols = collections.map(c => ({ name:c.name, id:c.id, modes:c.modes.map(m=>m.name), varCount:c.variableIds.length }));
const pages = figma.root.children.map(p => ({ name:p.name, id:p.id, children:p.children.length }));
return { file: figma.root.name, pages, collections: cols };
```

## Step 1 — Brand (raw scales)

Validate on one family first, then run the rest with the same loop. Use an idempotency guard so a re-run can't duplicate.

```js
const SCOPES = ["ALL_FILLS","STROKE_COLOR","EFFECT_COLOR"]; // hide later with [] if desired
const DATA = {
  "Base": { "White":"#FFFFFF", "Black":"#000000" },
  "Gray": { "50":"#FAFAFA","100":"#F5F5F5","200":"#E5E5E5","300":"#D4D4D4","400":"#A3A3A3","500":"#737373","600":"#525252","700":"#404040","800":"#262626","900":"#171717","950":"#0A0A0A" }
  // ...all hue families
};
let collection = (await figma.variables.getLocalVariableCollectionsAsync()).find(c=>c.name==="Brand");
if(!collection) collection = figma.variables.createVariableCollection("Brand");
const modeId = collection.modes[0].modeId;
const existing = new Set((await figma.variables.getLocalVariablesAsync()).filter(v=>v.variableCollectionId===collection.id).map(v=>v.name));
let created = 0;
for(const group of Object.keys(DATA)){
  for(const step of Object.keys(DATA[group])){
    const name = `${group}/${step}`;
    if(existing.has(name)) continue;
    const v = figma.variables.createVariable(name, collection, "COLOR");
    v.scopes = SCOPES;
    v.setValueForMode(modeId, hexToRgb(DATA[group][step]));
    created++;
  }
}
return { collectionId: collection.id, modeId, created };
```

## Step 2 — Alias (semantic, aliased to Brand)

```js
const SCOPES = ["ALL_FILLS","STROKE_COLOR","EFFECT_COLOR"];
const STEPS = ["50","100","200","300","400","500","600","700","800","900","950"];
const FAMILIES = { primary:"Violet", secondary:"Slate", neutral:"Gray", info:"Blue", success:"Green", warning:"Orange", error:"Red", accent:"Pink" };

const brandCol = (await figma.variables.getLocalVariableCollectionsAsync()).find(c=>c.name==="Brand");
const brandByName = {}; (await figma.variables.getLocalVariablesAsync()).filter(v=>v.variableCollectionId===brandCol.id).forEach(v=>{brandByName[v.name]=v.id;});
let alias = (await figma.variables.getLocalVariableCollectionsAsync()).find(c=>c.name==="Alias") || figma.variables.createVariableCollection("Alias");
const modeId = alias.modes[0].modeId;

function aliasTo(name, brandName){
  const bid = brandByName[brandName]; if(!bid) throw new Error("missing brand "+brandName);
  const v = figma.variables.createVariable(name, alias, "COLOR");
  v.scopes = SCOPES;
  v.setValueForMode(modeId, { type:"VARIABLE_ALIAS", id: bid });
}
for(const role of Object.keys(FAMILIES)) for(const s of STEPS) aliasTo(`${role}/${s}`, `${FAMILIES[role]}/${s}`);
aliasTo("foundations/white","Base/White");
aliasTo("foundations/black","Base/Black");
return { collectionId: alias.id };
```

## Step 3 — Mapped (Light/Dark, aliased to Alias)

Create the modes, then set both mode values per token. Values are `[lightAliasName, darkAliasName]`.

```js
const aliasCol = (await figma.variables.getLocalVariableCollectionsAsync()).find(c=>c.name==="Alias");
const aliasByName = {}; (await figma.variables.getLocalVariablesAsync()).filter(v=>v.variableCollectionId===aliasCol.id).forEach(v=>{aliasByName[v.name]=v.id;});

const mapped = figma.variables.createVariableCollection("Mapped");
const lightModeId = mapped.modes[0].modeId;
mapped.renameMode(lightModeId, "Light");
const darkModeId = mapped.addMode("Dark"); // throws on starter-tier files — guard if needed

function mk(name, lightName, darkName, scopes){
  const lid=aliasByName[lightName], did=aliasByName[darkName];
  if(!lid||!did) throw new Error("missing alias "+lightName+" / "+darkName);
  const v=figma.variables.createVariable(name, mapped, "COLOR");
  v.scopes=scopes;
  v.setValueForMode(lightModeId, {type:"VARIABLE_ALIAS", id:lid});
  v.setValueForMode(darkModeId,  {type:"VARIABLE_ALIAS", id:did});
}
const surface = {
 "surface/primary":["foundations/white","neutral/950"],
 "surface/secondary":["neutral/50","neutral/900"],
 "surface/brand":["primary/600","primary/500"],
 "surface/error-subtle":["error/50","error/950"]
 // ...full set in architecture.md
};
for(const n of Object.keys(surface)) mk(n, surface[n][0], surface[n][1], ["FRAME_FILL","SHAPE_FILL"]);
// repeat for text (["TEXT_FILL"]), icon (["SHAPE_FILL","FRAME_FILL"]), border (["STROKE_COLOR"])
return { collectionId: mapped.id, lightModeId, darkModeId };
```

## Step 4 — Responsive (Desktop/Mobile, FLOAT)

```js
const STYLES = ["hero","h1","h2","h3","h4","h5","h6","paragraph-lg","paragraph-md","paragraph-sm","caption"];
const SIZE = { hero:[64,40], h1:[48,32], "paragraph-md":[16,16] /* ...[desktop,mobile] */ };
const LH   = { hero:[72,48], h1:[56,40], "paragraph-md":[24,24] /* ... */ };
const PS   = { hero:[24,16], h1:[20,16], "paragraph-md":[16,16] /* ... */ };

const coll = figma.variables.createVariableCollection("Responsive");
const desktop = coll.modes[0].modeId; coll.renameMode(desktop, "Desktop");
const mobile  = coll.addMode("Mobile");
function mkFloat(name, d, m, scope){
  const v=figma.variables.createVariable(name, coll, "FLOAT");
  v.scopes=[scope];
  v.setValueForMode(desktop, d);
  v.setValueForMode(mobile, m);
}
for(const s of STYLES){
  mkFloat(`font-size/${s}`, SIZE[s][0], SIZE[s][1], "FONT_SIZE");
  mkFloat(`line-height/${s}`, LH[s][0], LH[s][1], "LINE_HEIGHT");
  mkFloat(`paragraph-spacing/${s}`, PS[s][0], PS[s][1], "PARAGRAPH_SPACING");
}
return { collectionId: coll.id, desktop, mobile };
```

## Step 5 — Font vars (Brand) + bound text styles

Font vars (STRING) go in Brand:

```js
const brand = (await figma.variables.getLocalVariableCollectionsAsync()).find(c=>c.name==="Brand");
const bmode = brand.modes[0].modeId;
for(const [n,val,sc] of [
  ["font-family/heading","Inter","FONT_FAMILY"], ["font-family/body","Inter","FONT_FAMILY"],
  ["font-weight/regular","Regular","FONT_STYLE"], ["font-weight/semibold","Semi Bold","FONT_STYLE"]
]){ const v=figma.variables.createVariable(n, brand, "STRING"); v.scopes=[sc]; v.setValueForMode(bmode, val); }
```

Text styles — load the font first; bind with the variable **object**. Note Inter's weight string is `"Semi Bold"` (with a space), not `"SemiBold"`.

```js
await figma.loadFontAsync({family:'Inter',style:'Semi Bold'});
const cols = await figma.variables.getLocalVariableCollectionsAsync();
const resp = cols.find(c=>c.name==="Responsive"); const brand = cols.find(c=>c.name==="Brand");
const dM = resp.modes.find(m=>m.name==='Desktop').modeId;
const all = await figma.variables.getLocalVariablesAsync();
const r={}; all.filter(v=>v.variableCollectionId===resp.id).forEach(v=>{r[v.name]=v;});
const b={}; all.filter(v=>v.variableCollectionId===brand.id).forEach(v=>{b[v.name]=v;});
const existing = new Set((await figma.getLocalTextStylesAsync()).map(s=>s.name));

for(const [name,key] of [["H1","h1"],["H2","h2"]]){ // headings: Semi Bold + heading family
  if(existing.has(name)) continue;
  const ts=figma.createTextStyle();
  ts.name=name;
  ts.fontName={family:'Inter',style:'Semi Bold'};
  ts.fontSize=r[`font-size/${key}`].valuesByMode[dM];                 // base value (variable overrides)
  ts.lineHeight={unit:'PIXELS', value:r[`line-height/${key}`].valuesByMode[dM]};
  ts.paragraphSpacing=r[`paragraph-spacing/${key}`].valuesByMode[dM];
  ts.setBoundVariable('fontSize', r[`font-size/${key}`]);
  ts.setBoundVariable('lineHeight', r[`line-height/${key}`]);
  ts.setBoundVariable('paragraphSpacing', r[`paragraph-spacing/${key}`]);
  ts.setBoundVariable('fontFamily', b['font-family/heading']);
  ts.setBoundVariable('fontStyle', b['font-weight/semibold']);
}
// body styles: load Inter Regular, bind font-family/body + font-weight/regular
```

## Step 6 — Hide primitives

```js
const cols = await figma.variables.getLocalVariableCollectionsAsync();
const ids = new Set([cols.find(c=>c.name==="Brand").id, cols.find(c=>c.name==="Alias").id]);
let hidden=0;
for(const v of await figma.variables.getLocalVariablesAsync()){
  if(ids.has(v.variableCollectionId) && v.resolvedType==="COLOR"){ v.scopes=[]; hidden++; }
}
return { hidden };
```

## Verification — resolve the full chain

Follow `VARIABLE_ALIAS` ids; downstream single-mode collections are read at their own `modes[0]`.

```js
async function resolveHex(varId, modeId){
  let v=await figma.variables.getVariableByIdAsync(varId);
  let val=v.valuesByMode[modeId]; let g=0;
  while(val && val.type==="VARIABLE_ALIAS" && g<12){
    v=await figma.variables.getVariableByIdAsync(val.id);
    const col=await figma.variables.getVariableCollectionByIdAsync(v.variableCollectionId);
    val=v.valuesByMode[col.modes[0].modeId]; g++;
  }
  if(!val || val.type==="VARIABLE_ALIAS") return "UNRESOLVED";
  const to=x=>Math.round(x*255).toString(16).padStart(2,'0').toUpperCase();
  return '#'+to(val.r)+to(val.g)+to(val.b);
}
```

## Optional — Light/Dark demo card

Build a card bound to Mapped tokens and force each copy to a mode with `node.setExplicitVariableModeForCollection(mappedCollection, modeId)`, then `await root.screenshot()` to show the swap in one image. Bind a fill: `node.fills = [figma.variables.setBoundVariableForPaint({type:'SOLID',color:{r:0,g:0,b:0}}, 'color', variableObject)]` (it returns a **new** paint — reassign it). Load fonts before any text. In auto-layout, append children before setting `layoutSizingHorizontal='FILL'`, and `resize()` before setting axis sizing modes.
