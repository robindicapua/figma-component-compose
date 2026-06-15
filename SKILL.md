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

**Tooling:** this skill supports two backends — figma-cli (`/Users/Nubeh/figma-cli`)
and Figma Console MCP (`mcp__figma-console__*`). **Before starting Phase 1, ask
the user which one to use for this session:**

> Should I use figma-cli or Figma Console MCP for this session?

Use that backend for every Figma operation in the session. All eval-script
examples in this skill are shown in figma-cli syntax — see
[`references/execution-backends.md`](references/execution-backends.md) for the
Figma Console MCP equivalent of each one (the script bodies are identical;
only the invocation and a few backend-specific behaviors — timeouts, screenshot
cadence, health checks — differ).

Never use any other Figma MCP tools (e.g. `mcp__claude_ai_Figma__*`) for this
skill's canvas-write operations.

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

### Slots — clarify before building

If the spec or request describes named content regions as "slots" (e.g. a
Card's Header/Body/Footer, an action group, a list of items), **ask the user
before Phase 3**:

> Should [SlotName] be a real Figma slot (`isSlot`, instance-editable —
> designers can add/remove/replace content per instance without detaching),
> or a plain layout frame with default placeholder content?

Don't assume based on whether the slot has a fixed or dynamic item count.
"Slot" in a spec can mean either the Figma-mechanism slot or just a named
layout region — only the user knows which they intend, and guessing means
redoing the build.

If real Figma slots are wanted, follow the `slot convert` workflow in
[`references/rules/slots.md`](references/rules/slots.md) for each slot frame
during Phase 3, after the frame and its default content exist.

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

### Mode-invariant tokens

Spacing, size, and radius tokens do **not** change between Light and Dark modes —
they must alias the same primitive in every mode. When creating or auditing these
tokens, verify that every mode value is a `VARIABLE_ALIAS` pointing to the same
source variable. A raw number (e.g. `0`) in any mode is a bug.

Categories that are always mode-invariant in this DS:
- `spacing/*` (gap, padding)
- `size/*` (icon sizes, control dimensions)
- `radius/*` (corner radii)

If you find a mode with a raw value instead of an alias, fix it by calling
`variable.setValueForMode(modeId, { type: 'VARIABLE_ALIAS', id: primitiveVarId })`
using the same alias as the other modes.

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

### 3.1 Classify component & resolve dependencies

Before querying variables or generating, determine whether this component should
be generated as a component set, and if so, resolve its atom dependencies.

**Read [`references/rules/atomic-dependencies.md`](references/rules/atomic-dependencies.md)**
and follow the classification and resolution workflow:

1. **Classify** the component as Visual, Layout Wrapper, or Compositional using
   the CSS and metadata signals described in the rule
2. **If Layout Wrapper**: present the clarity checkpoint to the user and wait for
   their choice before proceeding. Do NOT generate a component set unless the
   user explicitly requests it.
3. **If Visual or Compositional**: extract dependency names from metadata
   (`composition.nestedComponents`, `composition.commonPartners`) and source imports
4. **Resolve dependencies** using the lookup table or Figma traversal (see below)
5. **Build a dependency map**: for each found dependency, record the component
   set ID and available variants. For missing dependencies, warn the user.
6. **Pass the dependency map** to Step 3.4 (generation) so the eval script uses
   `createInstance()` instead of raw frames for found dependencies.

#### Dependency resolution with optional lookup table

An external JSON lookup table may exist that maps component names to Figma node
IDs. Read [`references/figma-map-lookup.md`](references/figma-map-lookup.md) for
the full schema and generation instructions.

**Resolution order:**

1. **Check for lookup table**: Look for a `figma-map.json` file provided by the
   user (typically in `~/.claude/data/`). If found, extract the `figmaFileKey`
   and compare it to this project's Figma file key. If they match, use the map.
2. **Direct ID lookup (fast path)**: For each dependency name, check
   `components.<name>.componentSetId` in the map. If non-null, use
   `figma.getNodeByIdAsync(id)` to fetch the node directly. This avoids
   cross-page traversal entirely.
3. **Fallback to findAll (slow path)**: If the map doesn't exist, the file keys
   don't match, or a dependency isn't in the map, fall back to
   `figma.root.findAll(n => n.type === 'COMPONENT_SET' && n.name === name)`.

The lookup table is **optional** — the workflow must work without it. Never
hardcode a specific file path; treat it as an external data source that may or
may not be present.

### 3.2 Confirm Figma variables exist

Phase 2's "Token → Figma variable path" section already covers querying local
variables and converting this DS's `--ds-*` tokens to Figma variable paths —
run that eval script before generating. For additional Plugin API binding
patterns and code examples, see
[`references/figma-plugin-api-patterns.md`](references/figma-plugin-api-patterns.md).

### 3.3 Navigate to the target page

Each component has its own page in the Figma file (see the Component references
table at the end of this skill for known page/set/section IDs). For a new
component, the page typically follows the `> ComponentName` naming convention.

```bash
cd /Users/Nubeh/figma-cli && node src/index.js eval "(async () => {
  const page = figma.root.children.find(p => p.name.trim() === '> ComponentName');
  if (!page) return 'Page not found';
  figma.currentPage = page;
  return 'Navigated to: ' + page.name;
})()"
```

### 3.4 Generate the component set

Build a single eval script that creates all variant combinations. The script must:

1. **Load fonts** (load the weights this design system requires — see
   [`references/figma-typography.md`](references/figma-typography.md))
2. **Fetch variables** and create a lookup helper
3. **Create component variants** by iterating over all dimension combinations:
   - For each combination of variant properties (e.g., `filled + default`), create
     a `figma.createComponent()`
   - Name it using Figma's variant syntax, matching code values exactly
     (lowercase): `Variant=filled, State=default`
   - Set auto-layout, sizing, padding, gap, corner radius
   - Set fills/strokes with placeholder colors, then bind to Figma variables
   - Create child elements (text labels, icons) with proper variable bindings
   - For children that map to resolved dependencies (from 3.1), use
     `componentNode.createInstance()` instead of creating raw frames. See
     [`references/rules/atomic-dependencies.md`](references/rules/atomic-dependencies.md)
     for instance creation and variant selection patterns.
   - Add boolean component properties for optional elements
4. **Combine all variants** using `figma.combineAsVariants(components, figma.currentPage)`
5. **Arrange in a grid** per Phase 2's Grid layout convention (rows = `Variant`,
   columns = `State`) — remove auto-layout from the set and position manually
6. **Return a report** with created count and any unmapped variables

For detailed Figma Plugin API patterns and code examples, read
[`references/figma-plugin-api-patterns.md`](references/figma-plugin-api-patterns.md).

#### Pattern detection checklist

Before generating, scan the component source for these patterns and load the
relevant rule:

| Pattern | Detection Signal | Rule File |
|---|---|---|
| **Sizing & layout** | Any component | [`references/rules/sizing-modes.md`](references/rules/sizing-modes.md) (always read) |
| **Icons** | lucide-react imports, icon props, SVG elements, spinners | [`references/figma-icon-library.md`](references/figma-icon-library.md) + [`references/rules/icon-recoloring.md`](references/rules/icon-recoloring.md) |
| **Typography** | Text nodes, font tokens, text-style CSS properties | [`references/figma-typography.md`](references/figma-typography.md) |
| **Nested components** | `.map()` loops, repeated elements with per-item state (isActive, isSelected) | [`references/rules/nested-components.md`](references/rules/nested-components.md) |
| **Dynamic item count** | Positive: prop is an array mapped with `.map()`; runtime-determined count; no fixed upper bound. Negative: named structural children (header/footer); fixed count (OTP=6, rating=5); single-child wrapper (Tooltip trigger); render-props | [`references/rules/slots.md`](references/rules/slots.md) |
| **Floating overlays** | Imports from `@floating-ui/*`, `@radix-ui/react-popover`, `@radix-ui/react-dropdown-menu`, `@radix-ui/react-tooltip`, `cmdk`, `@headlessui/react`; uses `createPortal`; `position: absolute\|fixed` with high z-index on the expanded part; conditional render of menu/dropdown/popover content | [`references/rules/floating-overlays.md`](references/rules/floating-overlays.md) |
| **Atomic dependencies** | Molecule/organism; metadata `nestedComponents` non-empty; imports design-system components | [`references/rules/atomic-dependencies.md`](references/rules/atomic-dependencies.md) |

**Cross-dependency**: `Nested components` and `Dynamic item count` often fire
together. When a parent maps over items AND each item has per-item state
(isActive, isSelected), you need BOTH: create a sub-component set per
`nested-components.md`, then use slot Pattern A (instance-filled) in the parent
per `slots.md`. If items are heterogeneous or have no per-item state, only
`slots.md` applies (Pattern B, frame-filled).

#### Key rules for the eval script

- Wrap everything in `(async () => { ... })()`
- Always set a placeholder fill/stroke BEFORE binding a variable
- Append children to parent BEFORE setting `layoutSizingHorizontal`
- Apply text style (`textStyleId`) BEFORE setting `.characters` (avoids Inter
  font loading issues)
- Return `JSON.stringify()` for structured results
- Use `figma.variables.setBoundVariableForPaint()` for variable binding (see the
  critical API rule below)
- For complex scripts, write to a temp `.js` file and use `eval --file <path>`,
  then clean up after
- Icons should use this project's icon component set when available — read
  [`references/figma-icon-library.md`](references/figma-icon-library.md)
- All text must use the project's font family via Figma text styles — read
  [`references/figma-typography.md`](references/figma-typography.md)

### 3.5 Report results

After generation, present a summary:

```
## Component Generated: Badge

- **Variants created**: 12 (4 variants x 3 states)
- **Properties**: Variant, State, Show Dot (boolean)
- **Variables bound**: 24 fill bindings, 12 stroke bindings
- **Unmapped tokens**: (none)
  OR
- **Unmapped tokens**:
  - `--ds-theme-color-utility-coral` -- no matching Figma variable found
- **Dependencies resolved**: 2/2 (Icon, Badge Dot found)
  OR
- **Dependencies**: none (atom component)
```

If there are unmapped tokens, suggest the user create the missing variables in
Figma or check for naming mismatches.

### 3.6 Operational notes

- Never delete existing content on the page. Only add new nodes.
- The figma-cli daemon has a 60-second timeout. For very large component sets
  (100+ variants), consider splitting into multiple eval calls.
- If the connection is lost mid-generation, run
  `cd /Users/Nubeh/figma-cli && node src/index.js daemon restart` and retry.
- The eval script can be substantial in size — the daemon handles large payloads.

---

**Project-specific overrides that take precedence over the rest of Phase 3:**

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
- [ ] All color, spacing, radius, **and dimension** values bound to Figma variables —
  no hardcoded hex and no hardcoded px on fixed-size elements (icons, control boxes,
  avatars). Use `node.setBoundVariable('width', sizeVar)` and
  `node.setBoundVariable('height', sizeVar)` for elements whose size comes from a
  `size/*` token.
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

These node IDs are stable on either backend as long as the nodes haven't been
deleted/recreated. If `figma.getNodeByIdAsync(id)` returns null, re-resolve by
page/name (`figma_search_components` on Figma Console MCP, or
`figma.root.findAll(...)` on either backend) rather than assuming the table is
wrong.
