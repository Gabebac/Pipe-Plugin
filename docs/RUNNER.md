# RUNNER.md — how graph execution works (living doc)

## Model

A `pipeline`/`model` board is a DAG of typed nodes. Running it:

1. Topologically order nodes from the trigger nodes (nodes with no incoming edges).
2. Each node executes against its **target backend** with inputs collected
   from its incoming wires.
3. Node states: `pending → running → done | error | skipped`. Stored per run
   in `plugin-data/pipe/runs/<run-id>.json`; the canvas can render the run
   state on the graph (like ComfyUI's colored node borders).

## Run record

```json
{"id":"r-main-001","board":"main","started":"ISO","status":"done",
 "steps":[{"node":"n1","target":"n8n","status":"done","ms":812,
           "out":{"summary":"..."},"refs":["workflow:TD-notify"]},
          {"node":"n2","target":"comfy","status":"error","err":"offline: queued"}]}
```

## Targets

| target | endpoints used | auth | verified |
|---|---|---|---|
| `n8n` | `POST /api/v1/workflows/{id}/run` or webhook trigger; REST per `devops/n8n-workflows` skill | `N8N_API` header (from config) | healthz OK 2026-10-15 |
| `comfy` | `POST /prompt` (API-format graph), poll `/history/{id}`, fetch `/view` | none (LAN) | **offline** — must queue |
| `agent` | in-process tool call | n/a | — |

## n8n specifics (from the skill + homelab refs)

- Never PUT `active` in a workflow body (HTTP 400). Activate via endpoint.
- Deploy then **re-GET** to verify; one bounded live action as proof.
- Host: 192.168.0.78:5678 (Zima box). API via `/api/v1/` with API key.

## ComfyUI specifics

- Submit API-format graph (`{"prompt": {id: {class_type, inputs}}}`).
- Models may look absent over `/models/...` (`[]`) — trust run errors instead.
- Offline behavior: mark node `queued`, notify Pooh, retry on next Run.
- KSampler widgets: `[seed,"fixed",steps,cfg,sampler,scheduler,denoise]` (7).

## Open questions

- [ ] n8n API key location on the Zima box (check compose env / n8n settings UI).
- [ ] Does the runner run live from the desktop canvas (POST to plugin REST) — yes, `/chat` style; reuse `/api/plugins/pipe/run`.