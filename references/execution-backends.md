# Reference: Execution backends (figma-cli vs Figma Console MCP)

This skill supports two interchangeable backends for talking to Figma Desktop.
**Ask the user which one to use at the start of the session** (see SKILL.md
Tooling section) and use that backend consistently for every step below.

Both backends are built on the same primitive: arbitrary Plugin API JavaScript
executed via `eval` with `enablePrivatePluginApi: true`. The script *bodies*
shown throughout this skill (the `(async () => { ... return JSON.stringify(...) })()`
patterns) work unchanged on either backend — only the way you invoke them differs.

## Running an eval script

**figma-cli:**
```bash
cd /Users/Nubeh/figma-cli && node src/index.js eval "(async () => { ... })()"
# or, for long scripts:
cd /Users/Nubeh/figma-cli && node src/index.js eval --file /tmp/script.js
```
- Daemon-backed, 60-second timeout per call
- For 100+ variant component sets, split into multiple eval calls

**Figma Console MCP:**
```
figma_execute({ code: "(async () => { ... })()", timeout: 30000 })
```
- Default timeout 5000ms, max 30000ms — **lower ceiling than figma-cli**. Split
  large generation scripts (e.g. per-variant or per-row) more aggressively than
  you would for figma-cli.
- Check `resultAnalysis.warning` in the response for silent failures (empty
  arrays, null returns) — figma-cli's CLI doesn't surface this automatically,
  so this is an extra signal Figma Console MCP gives you for free.

## Slot conversion

`component.createSlot(name)` + manual frame-property migration (see
`rules/slots.md`) is plain Plugin API code — it runs the same way under either
backend. figma-cli's `slot convert` command is just this script wrapped in a
CLI subcommand with selection-based targeting; under Figma Console MCP, run the
equivalent script directly via `figma_execute` (you can target the frame by ID
directly in the script — no selection step needed).

## Health check / connection status

**figma-cli:**
```bash
curl -s http://localhost:9222/json          # verify CDP connection
cd /Users/Nubeh/figma-cli && node src/index.js daemon start    # if not running
cd /Users/Nubeh/figma-cli && node src/index.js daemon restart  # if connection lost mid-task
```

**Figma Console MCP:**
```
figma_get_status({ probe: true })   # quick connection + roundtrip check
figma_diagnose({ verbose: true })   # human-readable health report, disambiguates
                                     # errors from other Figma MCP servers
```

## Screenshot / verification

**figma-cli:**
```bash
cd /Users/Nubeh/figma-cli && node src/index.js verify "SECTION_ID" | python3 -c \
  "import sys,json,base64; d=json.load(sys.stdin); open('/tmp/preview.png','wb').write(base64.b64decode(d['base64']))"
```
Used once, at the end of Phase 4, per the Completion checklist.

**Figma Console MCP:**
```
figma_capture_screenshot({ nodeId: "SECTION_ID" })
```
Figma Console MCP's tooling **mandates a tighter validation loop**: screenshot
the target area *before* creating (to find clear space and avoid overlaps),
after each major step in Phase 3.4 and Phase 4 (not just at the very end), and
again after fixing any issues — up to 3 iterations. On failure/retry, delete
partial artifacts (`node.remove()`) before retrying; never leave orphaned
frames or duplicate pages/sections behind.

Use `figma_capture_screenshot` (live plugin state) rather than
`figma_take_screenshot` (REST API, may lag behind just-made changes).

## Page / section housekeeping

Both backends require: never create a duplicate page or Section if one with the
matching name already exists — check first
(`figma.root.children.find(...)` / `figma.loadAllPagesAsync()` then search).
Phase 4's "Remove any previous section" step already covers this for Sections.

## Component / dependency lookup

For resolving atomic dependencies (Phase 3.1) or finding an existing component
set to update (Phase 3.3):

**figma-cli:** `figma.root.findAll(...)` via eval, or the optional
`figma-map.json` lookup table (`references/figma-map-lookup.md`).

**Figma Console MCP:** `figma_search_components({ query })` searches the open
file's local components (or a published library via `libraryFileKey`). Its
docs note that **component keys returned for library instantiation are
session-specific** — re-search rather than reusing a cached key from a prior
session. This does *not* apply to plain node IDs of nodes already in the open
file (e.g. the IDs in this skill's Component references table) — those remain
valid as long as the node hasn't been deleted/recreated, on either backend. If
`figma.getNodeByIdAsync(id)` returns null, re-resolve by page/name before
assuming the ID is stale.
