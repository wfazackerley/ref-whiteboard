# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A single-file, zero-dependency web application: an interactive referee whiteboard for visualizing American football (IFAF) field scenarios. The entire application lives in `index.html` (~1400 lines of embedded HTML, CSS, and vanilla JavaScript). There is no build step — open `index.html` directly in a browser to run it.

## Running

Open `index.html` in a browser. No server, build tool, or package manager required.

## Architecture

All logic is in `index.html`. The code is organized into these sections:

**State** — a single `state` object holds all runtime state: interaction mode, tokens, strokes, history/future stacks, drawing settings, field dimensions, and Apple Pencil detection.

**Modes** — the current mode (`state.mode`) drives how pointer events are interpreted:
- `select` — drag tokens
- `place-a`, `place-b`, `official`, `flag`, `ball` — click field to place token
- `draw` — freehand annotation on canvas
- `erase` — erase drawn strokes

**Field coordinate system** — all positions are stored as fractions (0–1) of field width/height (`xFrac`, `yFrac`). `measureField()` computes the pixel rectangle of the field image; `repositionToken()` converts fractional coords back to pixels on resize.

**Token management** — tokens are DOM elements positioned over the field image. Each token has `{ team, label, xFrac, yFrac, el }`. Officials use two-letter labels (R, U, D, L, B, S, F, C).

**Drawing** — strokes are captured on a `<canvas>` overlay and stored as `{ color, width, points: [{xFrac, yFrac}] }`. `redrawCanvas()` replays all strokes. Erasing finds and removes strokes that intersect the erase point.

**History** — undo/redo uses `state.history` and `state.future` stacks (max 20 each), storing snapshots of tokens + strokes. Keyboard shortcuts: `Ctrl+Z` / `Cmd+Z` (undo), `Ctrl+Y` / `Ctrl+Shift+Z` / `Cmd+Shift+Z` (redo).

**Save/Load** — scenarios export/import as JSON containing tokens and strokes arrays. `default-scenario.json` is fetched on init to pre-populate the field.

## Assets

- `field-ifaf.svg` — the football field diagram rendered as background
- `default-scenario.json` — default player positions loaded on startup
- `media/` — SVG icons for the toolbar
- `fonts/` — Circular Std font files (woff)
- `grass-tile.png` — background texture

## Key functions (approximate line numbers)

| Function | Purpose |
|---|---|
| `init()` | Bootstrap: build UI, load field, load default scenario |
| `measureField()` | Compute field pixel rect after image loads or resize |
| `setMode(mode)` | Switch interaction mode, update UI |
| `placeToken(xFrac, yFrac, team, label)` | Add a token to the field |
| `beginDrag(e, token)` | Start dragging a token |
| `redrawCanvas()` | Repaint all annotation strokes |
| `undo()` / `redo()` | History navigation |
| `saveScenario()` / `loadScenarioFile()` | JSON export/import |
| `buildOfficialSubpanel()` | Build official-type picker buttons |
