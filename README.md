# Figma Component Compose

An AI agent skill for creating, rebuilding, or extending Figma components in the
agentic-design-system library. It handles the full pipeline: extracting the
component API, building the component set with DS variables bound, and wrapping
the result in a consistent annotated Section on the canvas.

The skill lives at `.agent/skills/figma-component-compose/SKILL.md` and is
invoked automatically by Claude Code whenever a task touches Figma component work.

---

## What it produces

For any component, the skill outputs:

- A **Figma component set** with variant properties that match the code API
  exactly (same prop names, same values, lowercase)
- All visual properties (color, spacing, radius) **bound to DS Figma variables**,
  not hardcoded
- An **annotated Section** on the canvas: component name as title, row labels
  (variant type), column labels (state), near-white background, consistent padding

---

## Workflows by starting point

### Starting from existing code

The most common case. A component already exists in `packages/ui/src/components/`
and you want to create or sync its Figma representation.

1. Point the agent at the component folder (e.g. `button/`)
2. It reads `button.tsx` for the variant prop union, `variantVars` token map, and
   any boolean props; reads `button.metadata.yaml` for the description
3. It maps each prop to a Figma variant property (`variant` → `Variant`,
   `disabled` → `State=Disabled`, etc.)
4. It resolves DS CSS tokens to Figma variable paths
   (`--ds-theme-color-interactive-brand-default` → `theme/color/interactive/brand/default`)
5. It builds the component set, binds variables, and annotates the section

**Prompt example:**
> "Create the Figma component for Button using the code in `packages/ui/src/components/button/`."

---

### Starting from a design

You have a Figma frame, mockup, or exploration that you want to promote into a
proper, reusable component set.

1. Provide the Figma node URL or node ID of the existing frame(s)
2. The agent inspects the frame structure and infers the variant dimensions
   (what varies between states, what stays the same)
3. It identifies which fills/strokes map to DS tokens and replaces hardcoded
   values with variable bindings
4. It promotes the frame to a component, creates the variant combinations, and
   builds the component set
5. It annotates the section

This flow is best when you have a polished visual direction but haven't written
code yet. The resulting Figma component becomes the design-side source of truth
that can then drive the code implementation.

**Prompt example:**
> "I have a card design at node `42:100`. Turn it into a proper DS component with
> variants for default/hover/disabled states, using our variables."

---

### Starting from a spec or documentation

You have a written specification — a Notion page, a design brief, a PR description,
or an API table — that defines what the component should look like and how it
should behave.

1. Share the spec (paste it, link a file, or describe it)
2. The agent extracts the component API: name, variants, states, boolean toggles,
   sizing rules
3. It plans the Figma structure, confirms the grid layout with you if needed
4. It builds the component from scratch using DS tokens and variables
5. It annotates the section

This is useful early in a component's lifecycle, before any code is written, to
validate the API shape visually before committing to implementation.

**Prompt example:**
> "Here's the spec for a Status Badge component: it has three variants (info,
> warning, error), can show or hide a leading dot, and has default and disabled
> states. Build the Figma component."

---

### Starting from an existing Figma component

You have a component set already on a Figma page and want to extend, duplicate,
or clean it up — for example, adding a new variant, restructuring the grid, or
rebuilding it with proper variable bindings.

1. Provide the component set node ID (from the Figma URL, `figma-cli find`, or
   `figma_search_components` if using Figma Console MCP)
2. Describe what to change: add a variant, add a state, rename a property,
   rebind to variables, or restructure the grid
3. The agent inspects the current structure, makes the targeted change, and
   re-annotates the section

For **duplication** (creating a similar component as a starting point), the agent
copies the component set to a new page, renames everything, adjusts the props,
and re-annotates.

For **variable rebinding** (an existing component uses hardcoded values), the
agent walks every variant's fills/strokes and replaces them with variable bindings
derived from the DS token map.

**Prompt examples:**
> "Add a `ghost` variant to the Button component set."

> "The TextInput component set is using hardcoded hex colors. Rebind everything
> to DS variables."

> "Duplicate the Button component as a starting point for an IconButton component."

---

## Key conventions

| Convention | Rule |
|---|---|
| Figma property names | Match code prop names (`variant` → `Variant`, values lowercase) |
| State axis | Always `State` with `Default`, `Hover`, `Disabled`; add `Focus`/`Error` when relevant |
| Variable binding | All colors, spacing, radius → DS variables. No hardcoded values. |
| Section background | `#f8f9fa` (DS `gray.50`) |
| Font style | `"Semi Bold"` with a space — not `"SemiBold"` |
| Auto-layout | Always enabled on component frames |
| Grid layout | Rows = variant type, columns = state (unless component shape dictates otherwise) |

---

## Files in this skill

| File | Purpose |
|---|---|
| `README.md` | This file — human-readable overview and workflow guide |
| `SKILL.md` | AI agent instructions — the detailed 4-phase workflow, templates, rules |
| `references/figma-plugin-api-patterns.md` | Figma Plugin API patterns and code examples for variable binding, auto-layout, etc. |
| `references/figma-typography.md` | Font loading and text-style rules |
| `references/figma-icon-library.md` | Icon component set lookup and usage |
| `references/figma-map-lookup.md` | Optional external `figma-map.json` lookup table schema for dependency resolution |
| `references/execution-backends.md` | Maps every figma-cli command/eval pattern used in this skill to its Figma Console MCP equivalent |
| `references/rules/*.md` | Narrow, single-topic rules referenced from `SKILL.md` Phase 3 (sizing modes, icon recoloring, nested components, slots, floating overlays, atomic dependencies) |

This skill is self-contained — Phase 3 inlines the component-generation workflow
and all supporting reference files live under `references/`. It does not depend
on the `claude-skills` submodule.

---

## Tooling backends

This skill supports two interchangeable backends for talking to Figma Desktop:
**figma-cli** and **Figma Console MCP**. At the start of a session, the agent
asks which one to use, then uses it consistently for every Figma operation in
that session. See `references/execution-backends.md` for the mapping between
the two.
