---
name: figma-component-compose
description: >
  End-to-end workflow for creating or updating a Figma component from code or a
  spec/documentation requirement. Covers: extracting the component API from source,
  mapping props to Figma properties, binding DS tokens to Figma variables, building
  the component set, and wrapping it in an annotated Section. Always use this skill
  when a task involves creating, rebuilding, or syncing any component in Figma.
---

# Figma Component Compose

Creates a fully structured Figma component set from this design system's codebase
or from a documentation/spec requirement. The output is a component set bound to
DS variables, with variant properties aligned to the code API, wrapped in an
annotated Section.

**Tooling:** always use figma-cli (`/Users/Nubeh/figma-cli`). Never use Figma
Console MCP tools.

---

## When to use

- Creating a new Figma component that exists in code but not yet in Figma
- Rebuilding a Figma component after the code API changes (new variant, renamed prop)
- Syncing a Figma component to match the source of truth (code or design spec)
- Any task phrased as "add X to Figma", "create the Figma component for X", "update
  Figma to match the new variant"

---

## Workflow: 4 phases

```
Phase 1: Extract  →  Phase 2: Plan  →  Phase 3: Build  →  Phase 4: Annotate
(read code)           (map to Figma)    (create in Figma)  (section wrap)
```

---

## Phase 1 — Extract the component API

Read these files in order:

1. **`packages/ui/src/components/{name}/{name}.tsx`**
   - `variant` prop union → row axis values (e.g. `'filled' | 'outline' | 'danger' | 'success'`)
   - Other prop unions (size, color, intent) → additional axes
   - Boolean show/hide props → Figma boolean properties
   - `variantVars` map → DS token → CSS variable mapping

2. **`packages/ui/src/components/{name}/{name}.metadata.yaml`**
   - `description` field → section annotation subtitle (if desired)
   - `variants` list → cross-check against TSX

3. **`packages/ui/src/tokens/semantic.tokens.json`** + **`foundation.tokens.json`**
   - Resolve token aliases to final hex values (needed for any placeholder fills)

**What to produce from Phase 1:**

| Output | Source | Figma use |
|---|---|---|
| Variant row values | `variant` union in TSX | `Variant` property values |
| State column values | Design convention (see below) | `State` property values |
| Boolean toggles | Bool props in TSX | Figma boolean properties |
| Token → CSS var map | `variantVars` in TSX | Maps to Figma variable paths |
| Hex fallbacks | Resolved from tokens JSON | Placeholder fills before var binding |

---

## Phase 2 — Plan the Figma structure

### Prop → Figma property mapping

| Code prop | Figma property name | Notes |
|---|---|---|
| `variant` | `Variant` | Values must match code exactly (lowercase) |
| `size` | `Size` | Values: `sm`, `md`, `lg` |
| `disabled` | `State` (value: `Disabled`) | Merge with other states |
| `showLabel` / `showIcon` etc. | `Show Label` / `Show Icon` | Boolean property |
| Hover, focus (no code prop) | `State` (values: `Hover`, `Focus`) | Figma-only; designers need them |

**State axis convention for this DS:**
Always add a `State` column axis with at minimum: `Default`, `Hover`, `Disabled`.
Add `Focus` and `Error` when relevant to the component type.

### Grid layout

- **Rows** = `Variant` (the primary intent/type prop)
- **Columns** = `State` (default → hover → disabled, left to right)
- Additional boolean axes become separate column groups separated by extra gap

### Token → Figma variable path

This DS uses CSS variables in the form `--ds-theme-color-interactive-brand-default`.
Strip `--ds-` prefix and replace `-` with `/` to get the Figma variable path:
`theme/color/interactive/brand/default`.

```
CSS variable                              → Figma variable path
--ds-theme-color-interactive-brand-default  → theme/color/interactive/brand/default
--ds-theme-color-content-inverse            → theme/color/content/inverse
--ds-theme-spacing-content-md               → theme/spacing/content-md
--ds-theme-radius-default                   → theme/radius/default
```

Before building, confirm which variables exist in the file:
```bash
cd /Users/Nubeh/figma-cli && node src/index.js eval "(async () => {
  const vars = await figma.variables.getLocalVariablesAsync();
  return JSON.stringify(vars.map(v => v.name));
})()"
```

---

## Phase 3 — Build the component

For the heavy-lifting of the generation step — variable binding, Plugin API patterns,
auto-layout rules, icon handling, nested components, slots — read and follow:

```
.agent/skills/claude-skills/skills/figma-component-generator/SKILL.md
```

That skill covers Steps 1–7 in full. Apply its eval script patterns here.

**Project-specific overrides that take precedence over the generic skill:**

### Font
Always inspect existing text nodes to confirm font style names before loading.
Known quirk: Figma style is `"Semi Bold"` (with space), **not** `"SemiBold"`.

```bash
cd /Users/Nubeh/figma-cli && node src/index.js eval '(function(){
  const nodes = figma.currentPage.findAll(n => n.type === "TEXT");
  return JSON.stringify([...new Set(nodes.map(n => JSON.stringify(n.fontName)))]);
})()'
```

### Auto-layout padding and gap
Use DS semantic spacing tokens, not arbitrary px values:

| Role | Token | Value |
|---|---|---|
| Component internal padding (default) | `--ds-theme-spacing-content-md` | 16px |
| Component internal padding (compact) | `--ds-theme-spacing-content-sm` | 12px |
| Gap between elements | `--ds-theme-spacing-gap-sm` | 12px |
| Gap (tight) | `--ds-theme-spacing-gap-xs` | 4px |

### Corner radius
| Role | Token | Value |
|---|---|---|
| Default rounding | `--ds-theme-radius-default` | 8px |
| Pill/full-rounded | `--ds-theme-radius-full` | 32px |
| Sharp (tags, badges) | `--ds-theme-radius-sharp` | 4px |

### Variant naming
Figma variant names must match code values exactly (lowercase):
`Variant=filled, State=default` — NOT `Variant=Filled, State=Default`.

This ensures Code Connect and auto-matching work without manual overrides.

### layoutMode before repositioning
Always set `compSet.layoutMode = "NONE"` before repositioning children.
`VERTICAL` auto-layout silently ignores `n.x`/`n.y` assignments — this is the
most common silent failure when building grids.

---

## Phase 4 — Annotate (Section wrap)

After the component set is built and positioned, wrap it in a named Section.

### Remove any previous section
```bash
cd /Users/Nubeh/figma-cli && node src/index.js eval '(function(){
  const existing = figma.currentPage.findAll(
    n => n.type === "SECTION" && n.name === "COMPONENT_NAME"
  );
  existing.forEach(n => n.remove());
  return "removed " + existing.length;
})()'
```

### Section annotation eval script

Write to `/tmp/{component}-compose.js`, then run:
```bash
cd /Users/Nubeh/figma-cli && node src/index.js eval "$(cat /tmp/{component}-compose.js)"
```

Template (fill ALL-CAPS placeholders):

```javascript
(async () => {
  await Promise.all([
    figma.loadFontAsync({ family: "Inter", style: "Semi Bold" }),
    figma.loadFontAsync({ family: "Inter", style: "Medium" }),
  ]);

  const compSet = figma.getNodeById("COMPONENT_SET_ID");
  compSet.layoutMode = "NONE"; // always disable before repositioning

  // Fix variant grid
  const PAD = 20, VW = 78, VH = 40, CG = 20, RG = 20; // adjust per component
  const VARIANT_GRID = [
    // ["nodeId", rowIndex, colIndex],
  ];
  VARIANT_GRID.forEach(([id, row, col]) => {
    const n = figma.getNodeById(id);
    if (n) { n.x = PAD + col*(VW+CG); n.y = PAD + row*(VH+RG); }
  });

  // Apply prefix-element offset if columns differ by optional prefix (see rule below)
  // REDUCED_VARIANT_IDS.forEach((id, row) => {
  //   figma.getNodeById(id).y = PAD + row*(VH+RG) + PREFIX_OFFSET;
  // });

  const numCols = Math.max(...VARIANT_GRID.map(([,,c]) => c)) + 1;
  const numRows = Math.max(...VARIANT_GRID.map(([,r]) => r)) + 1;
  const CSW = PAD + numCols*VW + (numCols-1)*CG + PAD;
  const CSH = PAD + numRows*VH + (numRows-1)*RG + PAD;
  compSet.resizeWithoutConstraints(CSW, CSH);

  const S_PAD = 40, LABEL_W = 60, LABEL_G = 8;
  const CS_X = S_PAD + LABEL_W + LABEL_G; // 108
  const CS_Y = 148;
  const SEC_W = CS_X + CSW + S_PAD;
  const SEC_H = CS_Y + CSH + S_PAD + 8;

  // Create or reuse section
  const section = figma.createSection();
  section.name = "COMPONENT_NAME";
  section.x = compSet.x - CS_X;
  section.y = compSet.y - CS_Y;
  try { section.resize(SEC_W, SEC_H); } catch(e) {}
  try { section.fills = [{ type:"SOLID", color:{ r:248/255, g:249/255, b:250/255 } }]; } catch(e) {}

  section.appendChild(compSet);
  compSet.x = CS_X;
  compSet.y = CS_Y;

  const hex2rgb = h => {
    h = h.replace("#","");
    return { r:parseInt(h.slice(0,2),16)/255, g:parseInt(h.slice(2,4),16)/255, b:parseInt(h.slice(4,6),16)/255 };
  };
  const txt = (chars, style, size, hex) => {
    const t = figma.createText();
    t.fontName = { family:"Inter", style };
    t.fontSize = size;
    t.characters = chars;
    t.fills = [{ type:"SOLID", color:hex2rgb(hex) }];
    section.appendChild(t);
    return t;
  };

  const title = txt("COMPONENT_NAME", "Semi Bold", 26, "#141414");
  title.x = S_PAD; title.y = S_PAD;

  const divider = figma.createRectangle();
  divider.resize(SEC_W - S_PAD*2, 1);
  divider.fills = [{ type:"SOLID", color:hex2rgb("#dee2e6") }];
  divider.strokes = [];
  section.appendChild(divider);
  divider.x = S_PAD; divider.y = 96;

  const COL_LABELS = ["COL_1","COL_2"]; // e.g. state names
  COL_LABELS.forEach((label, col) => {
    const t = txt(label, "Medium", 11, "#868e96");
    t.textAlignHorizontal = "CENTER";
    t.resize(VW, 16);
    t.x = CS_X + PAD + col*(VW+CG);
    t.y = CS_Y - 28;
  });

  const ROW_LABELS = ["ROW_1","ROW_2"]; // e.g. variant names
  ROW_LABELS.forEach((label, row) => {
    const rowCY = CS_Y + PAD + row*(VH+RG) + VH/2;
    const t = txt(label, "Medium", 12, "#495057");
    t.textAlignHorizontal = "RIGHT";
    t.resize(LABEL_W, 16);
    t.x = S_PAD; t.y = rowCY - 8;
  });

  figma.currentPage.selection = [section];
  figma.viewport.scrollAndZoomIntoView([section]);
  return JSON.stringify({ done:true, sectionId:section.id, w:SEC_W, h:SEC_H });
})()
```

### Verify
```bash
cd /Users/Nubeh/figma-cli && node src/index.js verify "SECTION_ID" | python3 -c \
  "import sys,json,base64; d=json.load(sys.stdin); open('/tmp/preview.png','wb').write(base64.b64decode(d['base64']))"
```

---

## Optical alignment rule — prefix element offset

When grid columns differ by an optional prefix element (label, avatar, icon header),
top-aligning all rows misaligns the primary interactive element.

**Fix:** inspect the taller variant's children to find the primary element's `y` offset,
then shift all shorter column variants down by that same amount.

```javascript
const fullVariant = figma.getNodeById("FULL_VARIANT_ID");
const primaryEl = fullVariant.children.find(c => c.name === "Input"); // adapt name
const OFFSET = primaryEl.y; // e.g. 24 for Text Input

reducedVariantIds.forEach((id, row) => {
  figma.getNodeById(id).y = PAD + row*(ROW_H+RG) + OFFSET;
});
// Row height stays equal: OFFSET + reducedH === fullH ✓
```

**Apply when:** column axis represents "without X" (label, icon, avatar).
**Skip when:** column axis represents purely stylistic differences (state, color, size).

---

## Paint variable binding — critical API rule

`figma.variables.setBoundVariableForPaint(paint, field, variable)` is a **pure function**.
It returns a new paint object — it does NOT mutate the original.

**Always use the return value:**
```javascript
// CORRECT
node.fills = node.fills.map(p => figma.variables.setBoundVariableForPaint(p, 'color', myVar));
node.strokes = node.strokes.map(p => figma.variables.setBoundVariableForPaint(p, 'color', myVar));

// WRONG — silent no-op, no error thrown, binding never applied
figma.variables.setBoundVariableForPaint(node.strokes[0], 'color', myVar);
node.strokes = node.strokes; // original is unchanged
```

`node.setBoundVariable(property, alias)` is a separate API for **scalar** properties only
(`strokeWeight`, `opacity`, `cornerRadius`, etc.) — do not use it for paint colors.

---

## Completion checklist

Before marking the task done:

- [ ] Variant property names match code prop names exactly (lowercase values)
- [ ] All color/spacing/radius values bound to Figma variables — no hardcoded hex
- [ ] Auto Layout used on every component frame
- [ ] `layoutMode = "NONE"` set on component set before any repositioning
- [ ] Component set named after the component (PascalCase: `Button`, `TextInput`)
- [ ] Section created with title, divider, row labels, column labels
- [ ] `figma-cli verify` screenshot reviewed — alignment looks correct
- [ ] Prefix element offset applied if columns differ by optional prefix element

---

## Component references

| Component | Page | Set ID | Section ID | Grid |
|---|---|---|---|---|
| Button | Button | `100:36` | `104:40` | 4 rows × 3 cols (Variant × State) |
| Text Input | TextInput | `1:245` | `105:60` | 4 rows × 2 cols (State × Show Label) |
