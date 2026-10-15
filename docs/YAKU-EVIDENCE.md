# Yaku 1.1.7 — extraction & evidence index

What was pulled from the shipped app and where it lives.

## Artifacts

- Installer: `/opt/data/attachments/Yaku-Setup-1.1.7-3.exe` (NSIS-3 Unicode)
- 7zz tool: `/opt/data/venvs/bin/7zz` (static 7-Zip 23.01, from 7-zip.org/a/7z2301-linux-x64.tar.xz)
- Extracted tree: `/opt/data/cache/scratch/yaku-rea/`
  - `$PLUGINSDIR/app-64.7z` → `app/` (Electron), `app/resources/app.asar`
  - asar parsed (JSON header at byte 16, len 2463838; content base aligned `16+len+7 & ~7`)
  - app sources (node_modules skipped): `yaku-rea/asar-extract/` (156 files)
  - header dump: `yaku-rea/header.json`

## Key files

| file | what it holds |
|---|---|
| `dist/assets/Canvas-BqAttsHl.js` (2.2 MB minified) | the canvas: transform/zoom/ports/drag/resize |
| `dist/assets/boardService-v43t30zo.js` | board persistence: IndexedDB `YakuBoards` {boards, settings, snapshots} |
| `electron/main.cjs` | IPC + updater + licensing; `scrape-website`, `fetch-reddit-json` handlers |
| `electron/whistle*.cjs` | built-in browser/"whistle" engine |
| `api/*.js` | local server endpoints: transcripts (youtube→piped→kome), tiktok (tikwm), reddit json, website scrape, screenshot, url-metadata |

## Verified interaction facts (from Canvas bundle)

- Board transform: single `{x,y,scale}`; ctrl+wheel zoom exponential, clamp 0.01–5, zoom-to-cursor; plain wheel pan; two-finger touch pan/zoom.
- Ports: `data-connection-handle` + `data-handle-type="input"/"output"`; drag-to-connect with live tether; auto direction-fix so chat nodes are sinks.
- Node width clamp 240–1600; corner `data-resize-handle`; `data-scale-handle` for content zoom.
- Node kinds: video, document(text/pdf), text, website, audio, image, mindmap, book, screenshot, report, chat.
- Local API base: `http://127.0.0.1:3000/api/*` (fetch chain lives there in the real app; our plugin reuses the recovered chain in Python `service.py`).

## Cleanup

Scratch is pruned after 24h idle — if `yaku-rea/` is gone, re-extract:
`7zz x installer.exe` → `7zz x $PLUGINSDIR/app-64.7z` → parse `app.asar`
(header JSON at 16, content offsets relative to aligned base).