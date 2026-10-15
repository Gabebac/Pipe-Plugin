# RECON.md — ComfyUI recon + competitive research (2026-10-15)

## Part 1 · ComfyUI recon (source + runtime evidence)

Sources: ComfyUI_frontend repo cloned shallow at
`/opt/data/cache/scratch/comfy-frontend` (litegraph core), ComfyUI README/API
docs, frontend PR #4624 + RFC 0005, `devops/comfyui-ops` skill (runtime facts
for the gaming-PC instance). No REA needed — everything relevant is open source.

### What ComfyUI does that v0.1 Pipe does not (adopt list)

| Mechanic | Evidence | Adopt in Pipe as |
|---|---|---|
| **Partial execution** — run only selected/output nodes; backend computes transitive closure; cached nodes stay cached | frontend PR #4624 (moved to backend, `partial_execution_targets` param) | Runner: `run(selected)` reuses prior step outputs; run-record keeps per-node outputs |
| **Subgraphs** — selection → reusable component with typed I/O, nested, editable in place, publishable to library | docs.comfy.org/interface/features/subgraph + RFC 0005; backend gets *flattened* graph | Library boards + "Pack selection → library node" (flattens on run) |
| **Reroute dots** — move/re-route links cleanly instead of crossing spaghetti | litegraph `Reroute.ts` | v0.3+; plain bezier v0.2 |
| **Group ↔ bypass/mute** — bypass node = wire-through, mute = ignore | README shortcuts (Ctrl+M / Ctrl+B) | Node `enabled` states in runner + canvas badge |
| **Keyboard surface** — space-drag pan, ctrl+enter run, ctrl+z/y, dbl-click node search, `.` fit view | README shortcut table | Same keymap (documented deviations ok) |
| **Drag auto-pan + drop-on-node** — dragging a wire near edge pans canvas; dropping on node body defaults to best slot | `LGraphCanvas.ghost`/`LinkConnector` tests | Link drag with edge auto-pan + drop-on-node default port |
| **Cache-aware re-run** ("Everything is cached" — skip unchanged subtree) | core execution design | Run-record caches node outputs; unchanged inputs → `skipped(cached)` |
| **Offline/partner split** | `--offline`, `--disable-partner-nodes` | Target config: n8n vs comfy vs agent per node, never silently falls back |

### Anti-patterns ComfyUI's own docs admit (avoid list)

Their `docs/architecture/entity-problems.md` catalogs legacy litegraph sins:
renderer mutating model state, `_version++` from 12+ call sites, undo
checkpoints scattered across the canvas — they're migrating to an ECS/Pinia
store model because of it. **Pipe does it right from day one:** one owned graph
store, pure render, explicit history commit per mutation (see auxx/Auxx writeup:
"own your graph state, don't bolt state onto the lib").

### Runtime / API facts for the comfy target

- Endpoints: `POST /prompt` (API-format graph `{"prompt": {id:{class_type,inputs}}}`),
  poll `GET /history/{prompt_id}`, outputs `GET /view?filename=...`.
- Widgets: KSampler `[seed,"fixed",steps,cfg,sampler,scheduler,denoise]` (7).
- The gaming-PC instance went offline (192.168.0.68:8188 timeout 2026-10-15)
  → target must: health-check first, queue + notify on offline, never hard-fail.
- comfy-mcp exists (39 tools) but Pipe's runner talks HTTP directly — one
  fewer indirection; MCP is for ad-hoc agent use.

## Part 2 · Competitive round (what else is out there)

| Tool | Standout ideas worth stealing | Verdict |
|---|---|---|
| **n8n** | typed connections w/ validation; data pinned between runs; executions list per node | we RUN n8n — adopt *pin data* (test inputs) + per-node execution history |
| **React Flow ecosystem (auxx.ai writeup)** | own the state; 12 stores split by concern; **snapshot undo (50-deep)** rejected command-undo for 30+ mutation shapes; **edge `+` button inserts node mid-wire**; sendBeacon save on tab close; debounced autosave 5s | adopt: snapshot undo, edge-insert button, autosave+beacon (Hermes plugin store is slow — optimistic UI already proven in yaku v0.1) |
| **Windmill** | groups collapse→single node ("collapsed by default" persisted); markdown sticky notes; drag-shows-ghost of subflow | adopt: collapsible groups w/ persisted default; notes on canvas |
| **Rensei** | scope breadcrumbs (Root > Group > ...), `?scope=` deep-link; read-only centrally gated | breadcrumb nav when inside library/packed nodes (v0.3) |
| **Cruvero premium editor** | ELK auto-layout ("Tidy up" button); A* orthogonal edge routing; command palette (Cmd+K); execution Gantt timeline | v0.3+: auto-arrange, exec timeline (not A* routing — expensive, bezier+reroute enough) |
| **litegraph/Drawflow/Rete** | Rete typed sockets = our typed ports; litegraph canvas2d perf not needed at our scale | DOM nodes + SVG wires (yaku model, proven fine to ~200 nodes) |
| **Houdini/Nuke (concept)** | node graphs with typed *color-coded* sockets; quick-search palette on tab | type colors on ports + dbl-click canvas → node search |

## Part 3 · Resulting priority deltas for Pipe

New/changed from recon (vs. original build list):

1. **Snapshot undo/redo (50-deep)** — cheapest correctness win; no command-pattern.
2. **Edge-insert `+` button + drop-on-node + drag auto-pan** — the "editing feels terrible" complaint, fixed with these three.
3. **Pin/test inputs per node** (n8n-style) — lets you test-run one node without upstream data.
4. **Partial run**: run selection / node-only test-run reusing cached step outputs.
5. **Run-record replay on canvas**: node overlays + exec timeline (v0.3).
6. **Groups collapapsible w/ persisted default; markdown notes** (v0.3).
7. **Auto-arrange (ELK)** only if/when graphs routinely exceed ~40 nodes (v0.4+).
8. Deliberately skipped: A* edge routing, Yjs collaboration, canvas2d/WebGL rendering.