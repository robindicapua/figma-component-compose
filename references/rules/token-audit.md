# Rule: Token Binding Audit

## When to apply

Always — run this immediately after Phase 3.4 (component creation) and after
**any** ad-hoc edit to an existing component (new variant, restructure, style
fix), before Phase 4 (Section wrap) and before reporting completion. This
includes small follow-up fixes made outside the main build script (e.g.
"add a stroke to this node") — those are exactly where hardcoded values sneak
in.

## Why this exists

`figma_lint_design` (`rules: ['design-system']`) catches **hardcoded colors**,
but it does not check `cornerRadius`, `strokeWeight`, padding, or `itemSpacing`
against variables. Those dimension properties can be hardcoded with zero
warning from any tool — the only way to catch them is to check
`node.boundVariables` directly. Run this audit as a script via `figma_execute`
(or figma-cli `eval`) — don't delegate it to a subagent; it needs the live
node IDs from the build that just happened, and a fresh agent would have to
rediscover them.

## The audit script

Walks a node subtree and flags any non-zero/non-default fill, stroke, corner
radius, stroke weight, padding, or gap that has **no** `boundVariables` entry:

```javascript
(async () => {
  const root = await figma.getNodeByIdAsync('ROOT_NODE_ID'); // component set or component
  const unbound = [];

  function check(node, path) {
    const bv = node.boundVariables || {};

    (node.fills || []).forEach((paint, i) => {
      if (paint.type === 'SOLID' && paint.visible !== false && !paint.boundVariables?.color) {
        unbound.push({ path, prop: `fills[${i}].color`, value: paint.color });
      }
    });
    (node.strokes || []).forEach((paint, i) => {
      if (paint.type === 'SOLID' && paint.visible !== false && !paint.boundVariables?.color) {
        unbound.push({ path, prop: `strokes[${i}].color`, value: paint.color });
      }
    });

    ['topLeftRadius', 'topRightRadius', 'bottomLeftRadius', 'bottomRightRadius']
      .forEach((p) => {
        if (typeof node[p] === 'number' && node[p] !== 0 && !bv[p]) {
          unbound.push({ path, prop: p, value: node[p] });
        }
      });

    if (node.strokes?.length && node.strokeWeight !== 0) {
      // Uniform strokeWeight binds as a single key, but per-side weights
      // (set via setBoundVariable('strokeTopWeight', ...) etc.) are the
      // more common path — check both before flagging.
      const sideBound = ['strokeTopWeight', 'strokeBottomWeight', 'strokeLeftWeight', 'strokeRightWeight']
        .every((s) => bv[s]);
      if (!bv.strokeWeight && !sideBound) {
        unbound.push({ path, prop: 'strokeWeight', value: node.strokeWeight });
      }
    }

    ['paddingLeft', 'paddingRight', 'paddingTop', 'paddingBottom', 'itemSpacing']
      .forEach((p) => {
        if (typeof node[p] === 'number' && node[p] !== 0 && !bv[p]) {
          unbound.push({ path, prop: p, value: node[p] });
        }
      });

    if ('children' in node) {
      node.children.forEach((c) => check(c, `${path}/${node.name}`));
    }
  }

  check(root, '');
  return JSON.stringify({ unboundCount: unbound.length, unbound });
})()
```

## Resolving findings — never silently hardcode

For each entry in `unbound`:

0. **`itemSpacing` with ≤ 1 child**: a gap value here drives nothing (auto-layout
   gap only applies between children). Don't tokenize a dormant value — set it
   to `0`. If/when a second child (e.g. an icon) is added later, pick a real
   `spacing/gap-*` token at that point, when the gap actually becomes visible.
1. **Search local variables for an exact value match** (color: same hex in the
   active mode; numeric: same number). Use `figma.variables.getLocalVariablesAsync()`
   filtered by `resolvedType` (`COLOR` or `FLOAT`).
2. **If a matching variable is found**: bind it
   (`setBoundVariableForPaint` for paints, `node.setBoundVariable(prop, variable)`
   for scalars) and note the fix in the Phase 3.5 report under "Variables bound".
3. **If no matching variable exists**: STOP. Do not invent a value or pick the
   "closest" token silently. Ask the user, e.g.:

   > No token matches `strokeWeight: 2` on `Variant=filled, State=focus`.
   > Should I create a new `border-width/2` variable (and the matching
   > `--ds-*` code token), bind to an existing token with a different value,
   > or leave this value un-tokenized intentionally?

   This mirrors what should have happened when the focus-ring's
   `cornerRadius` and `strokeWeight` were first hardcoded — both should have
   triggered this question instead of a silent literal.

## Complementary check

Run `figma_lint_design({ nodeId: 'ROOT_NODE_ID', rules: ['design-system'] })`
alongside the audit above. It's cheap and catches hardcoded-color cases this
script might miss (e.g. GRADIENT/IMAGE paints), but it is **not** a substitute
for the dimension checks above.

## Report format addition

Add a line to the Phase 3.5 report:

```
- **Token audit**: 0 unbound properties
  OR
- **Token audit**: 2 unbound properties fixed (bound to existing tokens),
  1 flagged — see above
```
