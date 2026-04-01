# RefBoard

An interactive referee whiteboard for visualizing American football field scenarios. Place players, officials, and props on a field diagram, draw annotations, and save/load scenarios as JSON.

**No build step. No dependencies. Open `index.html` in a browser.**

## Features

- **Two environments** — IFAF 11v11 (120 × 56 yd) and Flag football (70 × 25 yd)
- **Tokens** — Team A players, Team B players, officials (R/U/D/L/B/S/F/C), penalty flag, ball, and line-to-gain marker
- **Editable numbers** — double-tap or double-click any player token to edit its number
- **Freehand drawing** — annotate the field with configurable color and stroke width
- **Snap to grid** — tokens snap to the nearest yard (1 yd IFAF, 0.5 yd Flag) when placed or dropped
- **Trace movement** — optionally draw a trail as you drag tokens
- **Undo / Redo** — full history (Cmd/Ctrl+Z, Cmd/Ctrl+Shift+Z)
- **Save / Load** — export and import scenarios as `.json` files
- **Apple Pencil support** — hover events filtered to prevent spurious drags

## Usage

1. Open `index.html` in any modern browser (Chrome, Safari, Firefox).
2. Select an environment (IFAF 11v11 or Flag) from the top-left dropdown.
3. Use the bottom toolbar to choose a mode, then click the field to place tokens.
4. Switch to **Move** mode to drag tokens around.
5. Use **Draw** / **Erase** to annotate.
6. **Save Scenario** exports a `.json` file; **Load Scenario** restores one.
7. **Reset** clears the field back to the default scenario.

## Keyboard shortcuts

| Shortcut | Action |
|---|---|
| Cmd/Ctrl+Z | Undo |
| Cmd/Ctrl+Y or Cmd/Ctrl+Shift+Z | Redo |

## File structure

```
index.html            — entire application (HTML + CSS + JS)
field-ifaf.svg        — IFAF 11v11 field diagram
field-flag.svg        — Flag football field diagram
default-scenario.json — default positions loaded on startup
privacy.html          — privacy policy
media/                — SVG toolbar icons + logo
fonts/                — Circular Std (woff/woff2)
grass-tile.png        — background texture
```

## Contributing

If you find a bug or have a feature request, please open an issue.
If you find RefBoard useful, you can [buy me a coffee on Ko-fi](https://ko-fi.com/wfazackerley).
