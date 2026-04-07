# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A single-file, zero-dependency web application: an interactive referee whiteboard for visualizing American football field scenarios. The entire application lives in `index.html` (~1900 lines of embedded HTML, CSS, and vanilla JavaScript). There is no build step — open `index.html` directly in a browser to run it.

Supports two environments: **IFAF 11v11** (120 × 56 yd field) and **Flag** (70 × 25 yd field).

## Running

Open `index.html` in a browser. No server, build tool, or package manager required.

## Architecture

All logic is in `index.html`. The code is organized into these sections:

**State** — a single `state` object holds all runtime state: interaction mode, tokens, strokes, history/future stacks, drawing settings, field dimensions, and Apple Pencil detection.

**Environments** — defined in `ENVIRONMENTS` constant; each entry specifies the field SVG, pixel-to-fraction boundary coords, field dimensions in yards, and snap resolution. `switchEnvironment(envKey)` swaps the field image and resets tokens/strokes.

**Modes** — the current mode (`state.mode`) drives how pointer events are interpreted:
- `select` — drag tokens; double-tap/click to edit player number
- `place-a`, `place-b`, `official`, `flag`, `ball`, `ltg` — click field to place token
- `draw` — freehand annotation on canvas
- `erase` — erase drawn strokes

**Vision cones** — when Cone Mode is active (`state.coneActive`), clicking an official begins a 3-click cone creation flow. Two variants exist (`state.coneVariant`): `'midline'` (click midline tip, then a point to set symmetric half-angle) and `'edge'` (click two edge endpoints; midline and angle derived via `edgesToConeShape()`). Cones are stored as `{ id, officialId, tipXFrac, tipYFrac, halfAngle }` in `state.cones`. Cones render on a dedicated `#cone-canvas` (z-index 5) that always sits behind tokens, regardless of whether Draw/Erase mode raises the annotation canvas z-index. Deleting an official also removes their cone via `removeConeForToken()`.

**Field coordinate system** — all positions are stored as fractions (0–1) of field width/height (`xFrac`, `yFrac`). `measureField()` computes the pixel rectangle of the field image; `repositionToken()` converts fractional coords back to pixels on resize.

**Token management** — tokens are DOM elements positioned over the field image. Each token has `{ team, label, xFrac, yFrac, el }`. Officials use two-letter labels (R, U, D, L, B, S, F, C). Player number labels are editable via double-tap/click (inline `contenteditable` on the token element).

**Drawing** — strokes are captured on `#draw-canvas` and stored as `{ color, width, points: [{xFrac, yFrac}] }`. `redrawCanvas()` replays all strokes on `#draw-canvas` and all cones on `#cone-canvas`. Erasing finds and removes strokes that intersect the erase point. Color and stroke-width options are shown in the draw sub-panel. In Draw/Erase mode `#draw-canvas` is raised to z-index 30 for pointer capture; `#cone-canvas` stays at z-index 5 so cones remain behind tokens in all modes.

**Snap to grid** — when enabled, tokens snap to the nearest yard (IFAF) or half-yard (Flag) when placed or dropped. The snap toggle is in the top toolbar. Apple Pencil hover events are filtered to avoid spurious drags.

**History** — undo/redo uses `state.history` and `state.future` stacks (max 20 each), storing snapshots of tokens + strokes. Keyboard shortcuts: `Ctrl+Z` / `Cmd+Z` (undo), `Ctrl+Y` / `Ctrl+Shift+Z` / `Cmd+Shift+Z` (redo).

**Save/Load** — scenarios export/import as JSON containing tokens and strokes arrays. `default-scenario.json` is fetched on init to pre-populate the field.

## Canvas layers (z-index order, low → high)

| Element | z-index | Purpose |
|---|---|---|
| `#cone-canvas` | 5 | Vision cones — always behind tokens |
| `#draw-canvas` | 10 (30 in draw/erase) | Annotation strokes and traces |
| `#token-layer` | 20 | All token DOM elements |
| `#cone-handle-layer` | 25 | Drag handles for editing cones |

## Assets

- `field-ifaf.svg` — IFAF 11v11 field diagram
- `field-flag.svg` — Flag football field diagram
- `default-scenario.json` — default player positions loaded on startup
- `privacy.html` — standalone privacy policy page
- `media/` — SVG icons for the toolbar; `logo.svg` is the app wordmark
- `fonts/` — Circular Std font files (woff/woff2)
- `grass-tile.png` — repeating background texture

## Tool usage

Prefer direct tool calls (Read, Glob, Grep) over subagents for codebase exploration. This is a single-file project — use direct tools.

## Key functions (approximate line numbers)

| Function | Line | Purpose |
|---|---|---|
| `init()` | ~880 | Bootstrap: build UI, load field, load default scenario |
| `measureField()` | ~950 | Compute field pixel rect after image loads or resize |
| `switchEnvironment(envKey)` | ~998 | Swap field image, reset tokens/strokes, update snap resolution |
| `snapFrac(xFrac, yFrac, token, dragDeltaX)` | ~963 | Snap fractional coords to nearest grid line |
| `setMode(mode)` | ~1090 | Switch interaction mode, update UI |
| `placeToken(xFrac, yFrac, team, label)` | ~1170 | Add a token to the field |
| `removeToken(token)` | ~1350 | Remove a token from the DOM and state |
| `removeConeForToken(token)` | ~1360 | Remove and return a ref's cone (called before history push on delete) |
| `beginDrag(e, token)` | ~1440 | Start dragging a token |
| `redrawCanvas()` | ~1790 | Repaint strokes on draw-canvas and cones on cone-canvas |
| `undo()` / `redo()` | ~1830 / ~1880 | History navigation |
| `buildOfficialSubpanel()` | ~1940 | Build official-type picker buttons |
| `toggleConeMode(variant)` | ~2130 | Activate/deactivate cone mode; variant is `'midline'` or `'edge'` |
| `onConeOfficialPointerDown(token)` | ~2150 | Handle official click in cone mode (start creation or show handles) |
| `onConeFieldClick(xFrac, yFrac)` | ~2165 | Route field click to midline or edge creation handler |
| `edgesToConeShape(official, ...)` | ~2230 | Derive cone data from two edge endpoints |
| `drawCone(ctx, official, cone, isGhost)` | ~2090 | Draw a cone onto any canvas context |
| `buildConeHandles(cone)` | ~2260 | Build drag handles for editing a placed cone |
| `saveScenario()` / `loadScenarioFile()` | ~1990 / ~2025 | JSON export/import |
