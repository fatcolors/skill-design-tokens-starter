# Token Architecture & Rules

The model behind this skill is a three-tier color system plus a responsive type system. It comes from the common "brand / alias / mapped" (a.k.a. primitive / semantic / component) methodology. The point of tiers is **single source of truth**: change a raw color once in Brand and every alias and component token downstream updates automatically.

## Tier 1 — Brand (primitives)

**Purpose:** raw hue scales in their purest form. Pure hex. Nothing here implies intent, role, or UI usage — it's just `Red`, `Violet`, `Gray`, etc.

- **Naming:** `Group/Step`, e.g. `Violet/600`, `Gray/50`. Group = hue name; step = position on the scale.
- **Scale:** a numeric ramp where lower = lighter, higher = darker. Common ramps: `50,100,200…900,950` (Tailwind/Untitled-UI style) or `100…1200` with `50`/`25` intermediates. **Match the source palette's own steps and names** rather than imposing a different scale.
- **Foundations:** keep `White`/`Black` in their own group (e.g. `Base/White`, `Base/Black`).
- **Fonts:** font family and weight live here as STRING variables (`font-family/heading`, `font-family/body`, `font-weight/regular|medium|semibold|bold`). They get pulled into Alias only for multi-brand systems.
- **Modes:** one mode. (Multi-brand systems can use modes here, but that's rare.)
- **Scopes:** once the Alias tier exists, hide colors from the picker with `scopes = []`. Font strings keep functional scopes (`FONT_FAMILY` / `FONT_STYLE`) so they remain usable.

## Tier 2 — Alias (semantic)

**Purpose:** assign meaning at the *group* level. This is the only tier allowed to say "this scale is the primary scale, that one is the error scale." Each role gets the **full scale**, every step aliased to the matching Brand step (so `primary/600 → Violet/600`).

- **Always reference Brand** via `VARIABLE_ALIAS` — never copy raw hex. That's what preserves the single-source-of-truth chain.
- **Roles (default mapping):**
  - `primary` → the brand hue
  - `secondary` → a secondary/neutral-blue hue (e.g. slate)
  - `neutral` → the true gray
  - `info` → blue · `success` → green · `warning` → orange · `error` → red
  - `accent` → one vibrant complementary hue (designer's pick)
  - `foundations/white`, `foundations/black` → Base
- **Not every Brand scale must be aliased.** Leave unused hues as raw primitives. If the palette has far more scales than roles, pick the ones with clear roles and leave the rest.
- **Modes:** one mode for a single brand. Multiple brands (e.g. sub-brands) → one mode per brand here.
- **Scopes:** hide from picker (`[]`) — Alias is plumbing between Brand and Mapped.

## Tier 3 — Mapped (component)

**Purpose:** the tokens components actually consume. This is where light/dark lives and where tokens branch into specific UI uses.

- **Categories:** `surface` (backgrounds), `text`, `icon`, `border`.
- **Intents & states:** primary / secondary / tertiary / disabled / brand / inverse, plus status (info/success/warning/error) and interaction states (hover, focus, on-brand). Not every category needs every intent — be sensible.
- **Modes:** `Light` + `Dark` in **one** collection. Every token sets both a Light and a Dark value, each a `VARIABLE_ALIAS` to an Alias var. This is the key advantage over the native-import / Token-Studio routes, which split light and dark into separate collections you must merge later.
- **"On-color" tokens:** text/icons placed on a colored surface need their own tokens (e.g. `text/on-brand` = white in both modes) so they stay legible on the fill.
- **Scopes (so each token only appears where it's valid):**
  - surface → `["FRAME_FILL","SHAPE_FILL"]`
  - text → `["TEXT_FILL"]`
  - icon → `["SHAPE_FILL","FRAME_FILL"]`
  - border → `["STROKE_COLOR"]`

### A solid default Mapped set (43 tokens)

This set was validated end-to-end; use it as a starting point and trim/extend per the design.

- **surface:** primary, secondary, tertiary, inverse, disabled, brand, brand-hover, brand-subtle, accent, info-subtle, success-subtle, warning-subtle, error-subtle
- **text:** primary, secondary, tertiary, disabled, brand, on-brand, link, info, success, warning, error
- **icon:** primary, secondary, disabled, brand, on-brand, info, success, warning, error
- **border:** default, subtle, strong, disabled, brand, focus, info, success, warning, error

Light/Dark patterning that reads well: surfaces invert (white ↔ near-black, neutral/50 ↔ neutral/900); text inverts (neutral/900 ↔ neutral/50); brand surface uses a slightly lighter step in dark (primary/600 → primary/500); status fills use the subtle end in light and the deep end in dark (e.g. error/50 ↔ error/950); status text/icons use 600–700 in light and 300–400 in dark; borders neutral/200 ↔ neutral/700.

## Responsive (typography)

**Purpose:** type that adapts across breakpoints. `FLOAT` variables, two modes: `Desktop` + `Mobile`.

- **Per style:** `font-size/<style>`, `line-height/<style>`, `paragraph-spacing/<style>`.
- **Styles:** `hero`, `h1`–`h6`, `paragraph-lg`, `paragraph-md`, `paragraph-sm`, `caption`.
- **Accessibility:** keep body (`paragraph-md`) at **16px in both modes**; anything below 16 needs care. Headings scale down on mobile; body usually stays.
- **Scopes:** `["FONT_SIZE"]` / `["LINE_HEIGHT"]` / `["PARAGRAPH_SPACING"]`.

### Text styles

Create Figma text styles (Hero, H1–H6, Paragraph Large/Medium/Small, Caption) and **bind** each to the Responsive vars (`fontSize`, `lineHeight`, `paragraphSpacing`) plus the Brand font vars (`fontFamily`, `fontStyle`). Applying one style then yields type that is both themeable and responsive (switch the node's Responsive mode → Desktop/Mobile).

## Dimensions (spacing, radius, stroke)

Numeric, **mode-invariant** layout tokens — they don't change with theme, so they live in **Alias**, not Mapped (a Light/Dark collection would store identical values twice and wrongly imply they vary by theme). They alias to a raw dimension scale in Brand, mirroring how color roles alias to color scales.

- **Brand** holds the raw scale as hidden FLOAT primitives: `Dimension/0 … Dimension/512` plus `Dimension/full` (a large value like 9999 for pills/circles). A 4pt-based ramp, e.g. 0,1,2,4,8,12,16,20,24,32,40,48,64… with `scopes = []`.
- **Alias** holds the semantic, consumable tokens, each scoped to a single property so it only appears where valid:
  - `spacing/*` → scope `GAP` (covers auto-layout **gap and padding**): none, xxs, xs, sm, md, lg, xl, 2xl, 3xl, 4xl, 5xl.
  - `radius/*` → scope `CORNER_RADIUS` (trimmed — radius needs few steps): none, xs, sm, md, lg, xl, 2xl, **full**.
  - `stroke/*` → scope `STROKE_FLOAT` (border width): none, thin (1), thick (2), thicker (4).

Components consume these directly from Alias. The "components consume Mapped only" rule is a *color/theming* rule; dimensions aren't themed, so don't route them through Mapped.

## Scope quick-reference

| Use | scopes |
|---|---|
| Color primitive, usable | `["ALL_FILLS","STROKE_COLOR","EFFECT_COLOR"]` |
| Color primitive, hidden | `[]` |
| surface | `["FRAME_FILL","SHAPE_FILL"]` |
| text | `["TEXT_FILL"]` |
| icon | `["SHAPE_FILL","FRAME_FILL"]` |
| border | `["STROKE_COLOR"]` |
| font size / line height / paragraph spacing | `["FONT_SIZE"]` / `["LINE_HEIGHT"]` / `["PARAGRAPH_SPACING"]` |
| font family / weight (string) | `["FONT_FAMILY"]` / `["FONT_STYLE"]` |
| spacing / gap / padding | `["GAP"]` |
| corner radius | `["CORNER_RADIUS"]` |
| border width (stroke) | `["STROKE_FLOAT"]` |
| dimension primitive, hidden | `[]` |

## 2-tier variant

If the user wants primitive→semantic instead of brand→alias→mapped: keep **Brand** as the primitive tier, collapse Alias+Mapped into one **Semantic** collection (Light/Dark) whose tokens alias straight to Brand. Same code patterns, one fewer tier.
