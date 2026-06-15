# Rule: `slot convert` quirks (figma-cli)

## When to apply

Whenever Phase 3 follows the base `slot convert` workflow from
`.agent/skills/claude-skills/skills/figma-component-generator/references/rules/slots.md`
to turn a default-content frame into a real Figma slot.

## Additional fixes discovered in this project

These are on top of the base workflow — apply them after every `slot convert` call:

- **`slot convert` resets `layoutSizingHorizontal`/`layoutSizingVertical` to
  `FIXED`** — if the original frame was set to `FILL` (e.g. to stretch across
  the parent), restore that after conversion:
  ```javascript
  slot.layoutSizingHorizontal = 'FILL';
  ```
- **`slot convert` appends the new slot to the end of the parent's
  `children`**, losing its original position. Restore order afterwards:
  ```javascript
  parent.insertChild(originalIndex, slot);
  ```
- If `slot convert` fails with `ETIMEDOUT` / "No frame selected" even though
  the select step reported success, the figma-cli daemon may not be running —
  run `node src/index.js daemon start` and retry the chained select + convert
  command.
