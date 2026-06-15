# Palette Input — accepting and normalizing source colors

The goal is to reduce whatever the user gives you to a simple map and then feed Step 1 of the recipe:

```js
// target shape
{ "Gray": {"50":"#FAFAFA", ..., "950":"#0A0A0A"}, "Violet": {...}, "Base": {"White":"#FFFFFF","Black":"#000000"} }
```

## Source A — DTCG / Token Studio / Figma variables export (JSON)

This is the most common and most reliable. Figma's "export variables" and Token Studio produce DTCG JSON. A single collection/mode file looks like:

```json
{
  "Gray": {
    "50": { "$type": "color", "$value": { "colorSpace": "srgb", "components": [0.98,0.98,0.98], "alpha": 1, "hex": "#FAFAFA" } }
  },
  "$extensions": { "com.figma.modeName": "Mode 1" }
}
```

- Read the file, walk each top-level group (skip `$extensions`), and for each leaf read `$value.hex` (or convert `components` → hex if `hex` is absent).
- The variable IDs (`com.figma.variableId`) signal the palette came from an **existing Figma collection** — ask whether to reference that collection rather than recreate it.
- Some exports use a simpler `{"$value": "#FAFAFA"}`. Handle both: `const hex = typeof leaf.$value === 'string' ? leaf.$value : leaf.$value.hex;`

## Source B — plain hex list

The user pastes families and hex (e.g. a Tailwind/Untitled-UI dump, or a color-scale generator's output). Parse into the target map. Confirm the step scale (50–950 vs 100–1200) and family names with the user, or infer from the keys present.

## Source C — Figma frame (read via MCP)

The original "vibe code your variables" method: the user has raw scales laid out on a Figma frame. Use the Figma MCP read tools (`get_variable_defs`, `get_metadata`, `get_screenshot`, or a read-only `use_figma` that walks the frame's rectangles/text) to extract swatch names and fills. Then normalize as above.

## Source D — generate from a seed

If the user has no scales and wants them generated, produce a standard ramp (50→950, ~11 steps) from one or two seed brand colors. Lowest control; fine for throwaway tests. Prefer asking for real scales or pointing them at a color-scale generator for production work.

## Normalization notes

- Keep the source's own step numbers and family names — don't silently renumber a 50–950 palette to 100–1200.
- Decide capitalization once (`Gray/500` vs `gray/500`) and match any existing Figma collection so you don't create near-duplicates.
- Confirm which families map to which semantic roles before building Alias (see architecture.md). If the user delegates ("you choose"), pick defaults and state them.
- A custom brand hue often hides among otherwise-stock scales (e.g. a custom `Violet` = the brand, with the rest being Tailwind). Treat the distinctive one as `primary`.
