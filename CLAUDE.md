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

**Field coordinate system** — all positions are stored as fractions (0–1) of field width/height (`xFrac`, `yFrac`). `measureField()` computes the pixel rectangle of the field image; `repositionToken()` converts fractional coords back to pixels on resize.

**Token management** — tokens are DOM elements positioned over the field image. Each token has `{ team, label, xFrac, yFrac, el }`. Officials use two-letter labels (R, U, D, L, B, S, F, C). Player number labels are editable via double-tap/click (inline `contenteditable` on the token element).

**Drawing** — strokes are captured on a `<canvas>` overlay and stored as `{ color, width, points: [{xFrac, yFrac}] }`. `redrawCanvas()` replays all strokes. Erasing finds and removes strokes that intersect the erase point. Color and stroke-width options are shown in the draw sub-panel.

**Snap to grid** — when enabled, tokens snap to the nearest yard (IFAF) or half-yard (Flag) when placed or dropped. The snap toggle is in the top toolbar. Apple Pencil hover events are filtered to avoid spurious drags.

**History** — undo/redo uses `state.history` and `state.future` stacks (max 20 each), storing snapshots of tokens + strokes. Keyboard shortcuts: `Ctrl+Z` / `Cmd+Z` (undo), `Ctrl+Y` / `Ctrl+Shift+Z` / `Cmd+Shift+Z` (redo).

**Save/Load** — scenarios export/import as JSON containing tokens and strokes arrays. `default-scenario.json` is fetched on init to pre-populate the field.

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
| `init()` | ~865 | Bootstrap: build UI, load field, load default scenario |
| `measureField()` | ~933 | Compute field pixel rect after image loads or resize |
| `switchEnvironment(envKey)` | ~981 | Swap field image, reset tokens/strokes, update snap resolution |
| `snapFrac(xFrac, yFrac, token, dragDeltaX)` | ~946 | Snap fractional coords to nearest grid line |
| `setMode(mode)` | ~1027 | Switch interaction mode, update UI |
| `placeToken(xFrac, yFrac, team, label)` | ~1152 | Add a token to the field |
| `beginDrag(e, token)` | ~1414 | Start dragging a token |
| `redrawCanvas()` | ~1657 | Repaint all annotation strokes |
| `undo()` / `redo()` | ~1692 / ~1727 | History navigation |
| `buildOfficialSubpanel()` | ~1770 | Build official-type picker buttons |
| `saveScenario()` / `loadScenarioFile()` | ~1825 / ~1860 | JSON export/import |
