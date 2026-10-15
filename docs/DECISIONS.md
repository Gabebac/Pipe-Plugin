# DECISIONS.md — why Pipe is shaped the way it is

Each decision lists the evidence behind it. No vibes.

## 1. Canvas interaction model comes from the real Yaku app, recovered not guessed

Decompiled `Yaku-Setup-1.1.7-3.exe` (NSIS → app-64.7z → `resources/app.asar`,
header parsed, 156 app files extracted, path preserved under
`/opt/data/cache/scratch/yaku-rea/`). From `dist/assets/Canvas-BqAttsHl.js`
(the real shipped canvas bundle):

- **Transform model**: one `{x, y, scale}` on the board layer; nodes are plain
  absolutely-positioned divs inside it. Yaku's own code:
  `transform` state + `getEffectiveTransform` + `zoomIn/zoomOut`.
- **Zoom**: ctrl+wheel, exponential — `Math.exp(-deltaY * (5e-4 + sensitivity/1e3*0.0495))`,
  clamped `0.01–5`, zoom-around-cursor (`x' = cx - (cx - x)*ratio`). Plain wheel = pan.
- **Connections**: ports declared with `data-connection-handle` +
  `data-handle-type="input|output"`; drag from port → live tether → drop on a
  node/port; chat nodes reverse direction automatically (chat targets always
  become the sink). This is the same model ComfyUI uses; it replaces our old
  "Connect button / click a wire to cut" UI, which Pooh called terrible.
- **Node sizing**: `width` clamped 240–1600 per node, height via a
  `data-resize-handle` corner grip; node inner content scales separately
  (`data-scale-handle`).
- **Node types in Yaku**: video, document/pdf, text, website, audio, image,
  mindmap, book, screenshot, report, chat. Our node kinds extend this set
  with typed port nodes (trigger/action/model) — ports are the addition.
- Why not reuse React Flow: the Hermes desktop plugin surface lints
  `desktop/*.js` to the plugin SDK only (`plugin_validate_desktop.py`) — no
  arbitrary npm deps in the renderer. So we implement the recovered model
  directly on plain React + absolutely-positioned divs, like Yaku does.

## 2. n8n is the action-pipeline runner; ComfyUI is the model-graph runner

- ComfyUI partner/provider nodes **cannot use user API keys** (comfy.org
  proxy only) — any chain of provider models with his keys must run in n8n.
  Evidence: `devops/comfyui-ops` skill, provider routing section.
- ComfyUI's real value: local-model GPU graphs (ControlNet/LoRA/img2img) on
  the gaming PC (RTX 3050). Evidence: `devops/comfyui-ops` skill.
- Endpoints verified from the Hermes backend on 2026-10-15:
  n8n `http://192.168.0.78:5678/healthz` → `{"status":"ok"}`;
  ComfyUI `http://192.168.0.68:8188/system_stats` → timeout (PC off/asleep).
  ⇒ Runner must treat ComfyUI as possibly-offline: queue + notify, never hard-fail.

## 3. One shared board store; runs are separate records

`plugin-data/pipe/boards/*.json` — same files for agent tools and desktop
tab (this worked in yaku-canvas; keep). Executions go to `runs/` with
timestamps + node statuses so a failed run can be inspected, not guessed.

## 4. Renaming to Pipe

Plugin name `pipe`, repo `Gabebac/Pipe-Plugin`. Old repo stays as archive
(`hermes-yaku-canvas`). Tool names become `pipe_*` (add/list/get/run/watch).
Desktop route `/pipe` («Open Pipe»).

## 5. Multiple models during development

Every session writes what it learned into `docs/RUNNER.md` + this file, and
commits. A fresh model (or human) must be able to continue from the repo
alone: decisions + evidence + current status live in git, not in one model's head.