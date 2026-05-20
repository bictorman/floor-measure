# Floor Plan Measure

A tiny single-file web app for measuring real-world distances on a floor plan (or any scaled image). Upload a picture, draw a line over a feature whose length you know, type that length in meters, and every line you draw afterwards is labeled with its computed length.

No build step, no dependencies, no server — just open `index.html` in a browser.

## Features

- **Upload** a floor plan image (drag-and-drop or file picker; PNG / JPG / WebP — anything the browser can render)
- **Click-click line drawing** for precision (no drag — you pick two points)
- **Edit existing lines**: drag an endpoint to reshape, or drag the body to reposition. Cursor changes on hover so you know what you'll grab
- **Nudge with arrow keys**: click a measurement in the sidebar to select it (it highlights both in the list and on the canvas), then use the arrow keys to nudge by 1 px (Shift+arrow for 10 px). `Delete` removes the selected line, `Esc` deselects
- **Undo / Redo**: every data change (drawing, dragging, nudging, deleting, calibration changes, clearing) is undoable. Consecutive arrow-key nudges within ~1 s collapse into one undo step. View changes (zoom/pan/rotation) are not historied
- **First line auto-calibrates**: after drawing the first line you're asked for its real length in meters, and that sets the scale for every subsequent measurement
- **Live length tooltip** while drawing — once calibrated, you see the length as you move the mouse
- **Smart units**: mm under 1 cm, cm under 1 m, otherwise meters with 2 decimals
- **Rotate** the whole thing (image + measurements) in 90° steps, keeping the visual center stationary
- **Zoom** with the mouse wheel (centered on cursor), `+` / `−` / `Fit` / `1:1` buttons, or keyboard
- **Pan** with `Shift`+drag, middle-click drag, or the `Pan` toggle button
- **Background opacity** slider — fade the floor plan down (useful when measurements clutter the image, or for tracing)
- **Stretch X / Y** sliders — scale the background image horizontally or vertically without touching the lines. Useful for calibrating distorted floor plans (e.g. phone photos with perspective squish) by visually re-aligning image features to known-correct measurements
- **Export PNG** of the floor plan with all measurements baked in — labels stay upright regardless of rotation
- **Save / Load project**: download a single `.fpm.json` file containing the image and every measurement, and load it back later (or on another machine). Drag-and-drop works too
- **Sidebar list** of every measurement, with per-row delete, a separate "Edit" on the scale-reference row, and **inline length editing** on every row — type a new length (e.g. `2.5`, `250cm`, `2500mm`) and the line is geometrically resized from its midpoint to match. This is *not* the same as setting the scale: the px-per-metre ratio is unchanged, so other measurements stay put
- **Area of closed loops**: when lines meet endpoint-to-endpoint to enclose a region, the area in square metres is shown inside (works for any polygon — rectangles, L-shapes, multiple disjoint rooms)
- **Persistent state** — image, lines, calibration, zoom/pan, and rotation are debounce-saved to `localStorage`, so closing the tab won't lose work
- **Reset** options: `Clear lines` removes measurements only; `Reset all` also drops the image

## Usage

1. Open `index.html` in any modern browser (Chrome/Firefox/Safari/Edge)
2. Drop a floor plan image onto the page (or use **Upload image**)
3. Click two points on a feature whose real length you know — say a 2 m kitchen counter
4. Type `2` when prompted — the scale reference is now set (highlighted green in the sidebar)
5. Click two points anywhere else — the length appears as a label and in the sidebar
6. Use **Export PNG** to download an annotated copy of the floor plan

## Keyboard shortcuts

| Key | Action |
|-----|--------|
| `F` | Fit image to view |
| `R` | Rotate 90° clockwise |
| `E` | Export PNG |
| `+` / `-` | Zoom in / out (centered) |
| `Arrow` keys | Nudge selected line 1 px (hold `Shift` for 10 px) |
| `Delete` / `Backspace` | Remove selected line |
| `Ctrl/Cmd` + `Z` | Undo |
| `Ctrl/Cmd` + `Shift` + `Z` / `Ctrl/Cmd` + `Y` | Redo |
| `Esc` | Deselect line / cancel half-drawn line |

## Implementation notes

- Single self-contained HTML file. No frameworks, no bundler, no network requests.
- The image is rendered as an `<img>` and measurements as an `<svg>` overlay; both sit inside a `#world` `<div>` that gets a single CSS transform combining pan, zoom, rotation, and a rotation-bbox offset. This keeps the image and the SVG perfectly aligned at any zoom/rotation.
- Line endpoints are stored in image-pixel coordinates. Stroke widths and font sizes are scaled by `1/zoom` so handles and labels stay constant on screen.
- Label text uses an SVG counter-rotation so it stays upright even when the world is rotated 90/180/270°. The PNG export does the equivalent in canvas: lines and handles rotate, label text does not.
- Persistence is debounced (150 ms). The image is stored as a data URL — `localStorage` is typically capped at ~5–10 MB, so very large images may fail to persist (the app still works for the current session; a warning is logged to the console).
- Project files are plain JSON with the image embedded as a base64 data URL. No zip / no external library. Shape: `{ format: "floorplan-measure", version: 1, savedAt, state: { imageDataUrl, imageW, imageH, lines, calibrationLineId, pixelsPerMeter, zoom, panX, panY, rotation, imageOpacity, imageScaleX, imageScaleY } }`. Base64 makes the file ~33% larger than the raw image, but the file is human-inspectable and the format is trivial to parse by other tools.
- Removing the calibration line deliberately *does not* auto-promote another line to the reference role. Doing so would silently change every other displayed length. Lines fall back to "no scale" until you set a new reference.
- Reshaping the calibration line (dragging an endpoint) keeps its declared real length and updates the px/m ratio — all other measurements rescale live. Translating it (dragging the body) leaves the ratio unchanged.
- Inline length editing (typing a new length in the sidebar) is deliberately distinct from calibration: it resizes the line geometrically from its midpoint, leaving the px/m ratio alone. For the calibration line, the line is resized the same way *and* its stored real length is updated so the ratio remains self-consistent — every other line's displayed length is unaffected.

## Limitations

- One line type (straight, two-point). No polylines, angles, or free-form annotations. Areas are computed automatically from closed loops of lines (no separate polygon tool).
- Area detection relies on endpoints landing close enough to count as the same point (fixed tolerance of 4 image pixels). There is no snap-to-endpoint while drawing, so for reliable loops you may need to zoom in and align endpoints manually.
- Metric only (millimetres / centimetres / metres). Adding imperial would be a small change to `formatLength` and the calibration prompt.
- Calibration is a single scalar (pixels-per-metre). Anisotropic distortion can be partially compensated with the **Stretch X / Y** sliders (which scale the background image without moving the lines), but there is no perspective / rotational skew correction — for an arbitrarily warped phone photo you'd want a proper homography tool, not this.
- Export is PNG. PDF export was considered but would either require a bundled library (jsPDF) or a print-to-PDF workaround; PNG keeps the file dependency-free.

## License

MIT.

## Credits

Built by Claude Opus 4.7 (Anthropic) at the request of the repo owner.
