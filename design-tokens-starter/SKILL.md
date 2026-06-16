---
name: design-tokens-starter
description: >-
  Build a multi-tier design system directly in Figma via the Figma MCP: Brand (raw color
  scales) → Alias (semantic roles) → Mapped (component tokens, Light/Dark) → Typography (type scale +
  fonts, Desktop/Mobile) → bound text styles, plus token-bound components (buttons, inputs, cards…)
  whose every color, gap, padding and radius is token-bound so they theme automatically. Use whenever
  the user wants to build, set up, or restructure design tokens, Figma variables, a token/variable
  system or collections, convert a palette into Figma variables, create a
  primitive→semantic→component (brand→alias→mapped) architecture, build a Figma design system, or
  create a component using existing variables/tokens — even when phrased loosely ("set up my design
  tokens", "turn this palette into Figma variables", "vibe code my Figma variables", "create a Button
  with my tokens"). Writes variables directly via the Plugin API with real cross-collection aliases
  and modes, avoiding the native-import limitation that breaks references.
---

# Design Tokens Starter

Build a production-grade, multi-tier design-token system as **real Figma variables**, fully wired with cross-collection references and modes, by driving the Figma Plugin API through the `use_figma` MCP tool.

## Why this skill exists (read this first)

There are three ways to get tokens into Figma. This skill deliberately uses the third:

1. **Figma's native "Import variables" (JSON)** — breaks on multi-tier systems. It cannot resolve *cross-collection references* (aliases), so `Mapped → Alias → Brand` chains either flatten or break. This is the #1 reason people's token imports "don't work."
2. **Token Studio plugin** — handles references, but is a manual, multi-step dance (export JSON → load plugin → export to Figma → manually merge the split light/dark collections).
3. **Direct write via `use_figma` (this skill)** — programmatically creates collections, modes, variables, and `VARIABLE_ALIAS` bindings. Light/Dark live as two modes in *one* collection from the start (no merge cleanup), and every reference is exact. Fully automated.

If the user hit "issues importing JSON to Figma variables," that's limitation #1 — and the fix is this skill.

## Prerequisites (do these before any write)

- **Load the `figma-use` skill** (and `figma-generate-library`) before calling `use_figma`. They carry the Plugin-API rules (return values, page reset, color 0–1 range, atomic failures, scopes). Pass `skillNames: "figma-use,figma-generate-library"` on every `use_figma` call.
- **Get the target file URL** and extract the `fileKey` (`figma.com/design/<fileKey>/...`). Confirm whether it's a clean file (create everything) or already has a Brand/primitive collection (reference it, don't duplicate).
- **Confirm modes are available.** Light/Dark and Desktop/Mobile need ≥2 modes per collection. That requires a paid Figma tier (Pro/Org). Check `whoami`; if the file's team is starter-tier, `addMode` throws — fall back to single-mode or warn the user.

## The architecture

A three-tier color system plus a responsive type system. Full rules, role-mapping guidance, and the scope table live in **[references/architecture.md](references/architecture.md)** — read it before designing the token set.

```
Brand (primitives)      Alias (semantic)          Mapped (component)        Typography (type)
raw hex scales     →    role → Brand         →    intent+state → Alias      font-size/line-height/
50–950, by hue          primary/secondary/        surface/text/icon/border  paragraph-spacing + fonts
no meaning              info/success/error/...    × Light / Dark modes      × Desktop / Mobile modes
hidden from pickers     hidden from pickers        the only color layer      feeds bound text styles
```

(Brand also holds the raw `Scale/*` dimension primitives; Alias also holds the semantic `spacing`/`radius`/`stroke` tokens — see the Dimensions note below.)

The guiding rule from the source methodology: **only Mapped (and Typography) tokens are consumed by components.** Brand and Alias are the plumbing — hide their *color* vars from the property picker (empty `scopes`) so designers reach for semantic tokens.

## Workflow

Work **incrementally and sequentially** — never run two `use_figma` calls in parallel (Figma state must mutate serially), and validate after each tier. The exact, proven JS for every step is in **[references/generation-recipe.md](references/generation-recipe.md)** — adapt those snippets rather than writing from scratch.

1. **Gather the palette.** Accept a DTCG/Token-Studio JSON export, a plain hex list, or read raw scales from a Figma frame via the MCP. Normalization guidance: **[references/palette-input.md](references/palette-input.md)**.
2. **Decide the token set.** Pick tiers (3-tier default; 2-tier primitive→semantic is supported), the role→scale mapping, and an accent. If the user says "you choose," pick sensible defaults and state them — don't stall on questions. Defaults and reasoning are in `architecture.md`.
3. **Inspect the file** (read-only `use_figma`): list collections, modes, pages, naming conventions. Match what exists; never duplicate an existing Brand collection.
4. **Brand** — create the collection (1 mode) and all raw color variables (`Group/Step`, e.g. `Violet/600`). Validate the pattern on one family first, then batch the rest.
5. **Alias** — create the collection; each semantic role gets the full scale, every step a `VARIABLE_ALIAS` to the matching Brand variable. Verify a few links resolve.
6. **Mapped** — create the collection with **Light** + **Dark** modes; each component token sets a Light value and a Dark value, both aliased to Alias vars. Resolve the full chain in both modes to prove it.
7. **Typography** — create **one** collection with **Desktop** + **Mobile** modes holding both the type scale (`FLOAT`: `font-size` / `line-height` / `paragraph-spacing`) **and** the fonts (`STRING`: `font-family/*`, `font-weight/*`). Keep body (`paragraph-md`) ≥16px for accessibility. Family/weight take the **same value in both modes** (breakpoint-invariant); sizes differ per mode. Keeping fonts and sizes in one collection is the default — see `architecture.md` for the rationale and the older Brand+Responsive split variant.
8. **Text styles** — create Figma text styles (Hero, H1–H6, Paragraph L/M/S, Caption) bound to the Typography vars (`fontSize`, `lineHeight`, `paragraphSpacing`, `fontFamily`, `fontStyle`).
9. **Hide primitives** — set `scopes = []` on all Brand + Alias color vars so only Mapped/Typography appear in pickers.
10. **Verify + demo** (optional but recommended) — resolve a sample of the chain across modes, and build a small card bound to Mapped tokens rendered in Light and Dark to visually confirm the swap.
11. **Components** (the rest of the design system) — once tokens exist, build token-bound components (Button, Input, Card, Badge…) whose every fill/border/label color is a Mapped token **and whose every gap, padding, corner radius and stroke width is a bound `spacing`/`radius`/`stroke` token** (never a raw number) — so they inherit Light/Dark theming *and* scale changes for free. Full pattern, the dimension-binding helpers, the variant-matrix recipe, and the gotchas (hug-sizing, `combineAsVariants` grid, component properties) are in **[references/components.md](references/components.md)**. Build atoms before molecules, one component at a time, screenshot to verify.

The defining promise of this skill: **anything you build references the tokens**, so a new component is automatically wired into the system — change a base color or flip a mode and every component updates. Never hardcode hex in a component.

**Scale (spacing / radius / stroke):** also add a 4pt numeric scale — raw `Scale/*` primitives in Brand (hidden), and semantic `spacing/*` (scope `GAP` = gap + padding), `radius/*` (`CORNER_RADIUS`), `stroke/*` (`STROKE_FLOAT`) in **Alias**, not Mapped (they're mode-invariant). Components must **bind** these to their layout properties (`itemSpacing`/`counterAxisSpacing`, the four `padding*`, the four `*Radius`, `strokeWeight`) with `setBoundVariable` — never hardcode the number. See [references/architecture.md](references/architecture.md) and Step 4b in [references/generation-recipe.md](references/generation-recipe.md).

## Key implementation rules (the things that bite)

These are distilled from real runs — see the recipe file for code.

- **Colors are 0–1**, not 0–255. Convert hex → `{r,g,b}` with `parseInt(...)/255`.
- **Always set `scopes` explicitly.** Color primitives meant for direct use: `["ALL_FILLS","STROKE_COLOR","EFFECT_COLOR"]`. Hidden primitives: `[]`. Per category: surface `["FRAME_FILL","SHAPE_FILL"]`, text `["TEXT_FILL"]`, icon `["SHAPE_FILL","FRAME_FILL"]`, border `["STROKE_COLOR"]`. Type vars: `["FONT_SIZE"]` / `["LINE_HEIGHT"]` / `["PARAGRAPH_SPACING"]`. Font strings: `["FONT_FAMILY"]` / `["FONT_STYLE"]`.
- **Aliases:** `v.setValueForMode(modeId, { type: "VARIABLE_ALIAS", id: targetVar.id })`. The target must already exist — build tiers in order.
- **Modes:** new collections start with one mode (`"Mode 1"`). `collection.renameMode(modeId, "Light")`, then `const darkId = collection.addMode("Dark")`. Set a value for *every* mode on *every* variable.
- **Resolving a chain** for verification means following `VARIABLE_ALIAS` ids; downstream single-mode collections (Alias, Brand) are read at their own `modes[0].modeId`, not the Mapped mode. See the resolver snippet.
- **Text styles** bind with `textStyle.setBoundVariable(field, variableObject)` (object, not id) for `fontSize`, `lineHeight`, `paragraphSpacing`, `fontFamily`, `fontStyle`. Load the concrete font (`figma.loadFontAsync`) before creating the style; the bound `fontFamily`+`fontStyle` values must correspond to a loaded font.
- **Layout dimensions on components bind the same way:** `node.setBoundVariable(field, variableObject)` where field is `itemSpacing`, `counterAxisSpacing`, `paddingLeft|Right|Top|Bottom`, `topLeftRadius|topRightRadius|bottomLeftRadius|bottomRightRadius`, or `strokeWeight`. Set a base number first (the variable overrides it), then bind to the Alias `spacing/`, `radius/` or `stroke/` token. This is *not optional* — a component left with a raw `itemSpacing`/`cornerRadius` number is the #1 thing that silently drops out of the token system.
- **Idempotency:** before creating, check existing names in the collection and skip duplicates — `use_figma` is atomic so a failed batch creates nothing, but a *succeeded* batch re-run would duplicate.
- **Batch sizing:** a flat data-loop creating ~100+ variables in one call is fine and fast. But validate the *pattern* (one family, or one category across both modes) in a small first call before scaling, since that's where API-shape mistakes surface.
- **Return IDs/counts** from every call so later steps and verification can reference them.

## Decision defaults (when the user says "you choose")

- Tiers: 3-tier Brand→Alias→Mapped + Typography.
- Roles: `primary`→brand hue, `secondary`→a neutral-blue (slate), `neutral`→true gray, `info`→blue, `success`→green, `warning`→orange, `error`→red, plus one vibrant `accent`. `foundations`→white/black.
- Modes: Light/Dark on Mapped, Desktop/Mobile on Typography.
- Typography: one consolidated collection (type scale + fonts). Offer the Brand+Responsive split only if asked.
- Hide Brand + Alias color vars from pickers.

State the choices you made; keep moving.

## Common pitfalls

- Building Mapped before Alias exists → alias targets missing. Order matters.
- Forgetting a value for the Dark (or Mobile) mode → that variable shows blank in that mode.
- Using `figma.currentPage = page` (throws) — use `await figma.setCurrentPageAsync(page)`. Variables are document-level so page context rarely matters here, but text styles/demo nodes live on a page.
- Leaving `ALL_SCOPES` → pollutes every picker.
- Trying to add a second mode on a starter-tier file → `addMode` throws; detect and degrade gracefully.
