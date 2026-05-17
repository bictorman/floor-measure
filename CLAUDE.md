# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file web app for measuring real-world distances on a scaled floor-plan image. **Everything lives in `index.html`** (HTML + CSS + JS in one file). No build system, no dependencies, no server, no tests.

To run: open `index.html` in a browser. To "deploy": host the one file.

## Architecture

### State and persistence

A single `state` object (see `defaultState()` near the top of the `<script>`) holds:

- `imageDataUrl`, `imageW`, `imageH` — the loaded image (image is embedded as a data URL, not a separate asset)
- `lines: [{id, x1, y1, x2, y2, realLength?}]` — endpoints in **image-pixel coordinates**
- `calibrationLineId`, `pixelsPerMeter` — scale reference
- `zoom`, `panX`, `panY`, `rotation` (0/90/180/270) — view state

Persistence has **two paths that are deliberately split**:

1. **`localStorage`** (`STORAGE_KEY = 'floorplan-measure-v1'`): the entire `state` is debounce-saved (150ms) by `save()`. Image is included as a data URL, so very large images may blow the ~5–10MB localStorage cap (handled — warns but keeps working in-session).
2. **Project files** (`.fpm.json`): `saveProject()` / `loadProject()` write/read the same shape wrapped with `{format: "floorplan-measure", version: 1, savedAt, state}`. Drag-and-drop on the canvas accepts both images and project files; `looksLikeProject()` dispatches.
3. **Undo/redo** stack (`undoStack`/`redoStack`) snapshots **only data state** — `{lines, calibrationLineId, pixelsPerMeter}`. View changes (zoom/pan/rotation) and the image itself are *not* historied. Limit 100, with `COALESCE_MS = 1000` so consecutive arrow-key nudges merge into one step via `commit(coalesceKey)`.

**Rule: every code path that mutates data state must call `commit(...)` *before* mutating, then `save()` + `render()` after.** Search for existing `commit()` calls to see the pattern (e.g., `removeLine`, `setCalibration`, `nudgeSelected`, drag end in mouse handlers).

### Three coordinate spaces

This is the part of the codebase most likely to bite you:

- **Image-pixel coords** — what's stored in `line.{x1,y1,x2,y2}`. Independent of zoom/pan/rotation.
- **World coords** — image-pixel coords after rotation has been applied and the rotated bbox shifted to (0,0). The `<svg>` and `<img>` both sit inside `#world`, which gets a single CSS transform.
- **Screen coords** — pixels in the viewport.

The CSS transform on `#world` is composed in this order (`applyTransform()`):
`translate(panX, panY) scale(zoom) translate(tx, ty) rotate(rotation)`

where `(tx, ty)` from `rotationOffset()` keeps the rotated bbox's top-left at (0,0). Conversions:

- `imageToWorld(px, py)` — apply rotation + offset
- `screenToWorld(sx, sy)` — full inverse (used by mouse handlers; returns image-pixel coords despite the name)

If you add features that touch coordinates (new shape types, snapping, etc.), use these helpers — do **not** reimplement the math inline.

### Rendering

`render()` rebuilds the SVG from scratch on every call (clears all children, re-appends). Cheap because line counts are small. Called after any state change.

To keep handles/labels visually constant across zoom levels, **stroke widths, handle radii, and font sizes are multiplied by `1/zoom`** before being set on SVG elements. Labels also get an inverse SVG rotation (`drawLabel`) so they stay upright when the world is rotated.

### PNG export is a separate renderer

`exportPNG()` + `drawLineCanvas()` reimplement the SVG render path in `<canvas>` 2D context. If you change how lines, handles, or labels look in SVG, you must mirror the change in the canvas path or exports will diverge from the screen. The export applies rotation to the lines/handles but keeps label text axis-aligned (matching the on-screen behavior).

### Calibration semantics (don't "fix" these)

- The **first line drawn** triggers a modal prompting for its real length; that sets `pixelsPerMeter`.
- **Reshaping** the calibration line (dragging an endpoint) preserves its declared `realLength` and recomputes `pixelsPerMeter` — so all other displayed lengths rescale live. See `recomputeCalibrationIfNeeded()`.
- **Translating** the calibration line (dragging the body) leaves `pixelsPerMeter` alone.
- **Deleting** the calibration line clears `pixelsPerMeter` rather than auto-promoting another line. Auto-promotion would silently change every other displayed length, which is worse than showing "no scale."

### Input model

- Drawing is click-click (two separate clicks), not drag — `handleDrawClick` tracks `drawingLine` between clicks.
- `mousemove` does hit-testing (`hitTest`) every frame to set the cursor (`updateHoverCursor`): endpoint handle → resize cursor, line body → move cursor.
- Pan: `Shift`+drag, middle-click drag, or the `Pan` toggle (`panModeToggle`).
- The `drag` global tracks an in-flight drag (endpoint reshape, body translate, or pan). Undo/redo is blocked while `drag` is set.

## Conventions worth keeping

- Pure vanilla JS in a single IIFE. No frameworks, no bundler, no network requests. Adding a dependency means breaking the "just open the file" promise — get explicit approval first.
- Metric only. `formatLength()` switches mm/cm/m based on magnitude; the calibration prompt assumes meters.
- The README's "Implementation notes" section captures the design rationale users have read — keep it in sync with code changes.
