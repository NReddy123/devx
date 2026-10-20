# Ink — build prompt

Status: sendable. Implement one phase per session.

Familiarity anchor (not a spec, not a brand to copy): a small infinite whiteboard used for boxes and arrows. Keep "draw a few shapes, pan around, find them tomorrow." Do not copy names, icons, or asset libraries from any shipping whiteboard.

## What this prompt is teaching

Multiplayer is not a feature you add later by sprinkling a library on a UI. It changes the data model. v1 is one document, one user, snapshots for undo. The scene is a list of shapes, not a painting app.

## Product

Ink is a local web app that edits one diagram: shapes on an infinite surface, saved as a JSON file.

v1 ships when a user can draw a rectangle and an arrow, pan to it after reload, undo the arrow, and still see the rectangle.

v1 is a failure if it can collaborate in realtime or export a slide deck, but cannot hit-test the topmost shape or round-trip the file.

## v1 user and job

- User: one person sketching a system on their own machine.
- Situation: they would otherwise drag boxes around in a slide they do not want to share.
- Done looks like: "I drew the picture, I reopened the file, the picture was the same."
- Not the user: a workshop of ten people on one board, or an illustrator painting.

## Core loop

1. User opens Ink. A blank document is up.
2. User draws a rectangle and an arrow between two points.
3. User pans away and zooms, then pans back.
4. User saves, reloads the page, and opens the same file.
5. The shapes are where they were. Undo removes the last change.

Time box: under three minutes to redraw a three-box sketch from memory the second time.

## Non-goals

- No realtime multiplayer, no cursors, no accounts, no sharing links. Reason: different product. Would replace snapshots and the file format.
- No freehand pen, no images, no sticky-note fonts, no frames, no layers panel. Reason: the shape set is frozen. More shapes are a new prompt.
- No slide mode, no presentation clicker, no minigame physics. Reason: different product.
- No automatic connector routing that jumps around obstacles. Reason: quality trap. An arrow is a straight segment between two points the user set.
- No component library and no templates. Reason: quality trap.
- No PNG export in phase 1. Reason: later, no new nouns. Phase 3 may rasterize the current viewport. The document format does not change.

## Rejected alternatives

- CRDT or Yjs "so we can add multiplayer later." Killed. It dominates the project and the v1 user does not have a second cursor. Do not add it.
- Canvas as the source of truth. Killed. The source of truth is the shape list. Render with SVG. Canvas is allowed only for phase 3 raster export.
- Command-pattern undo with inverse operations for every tool. Killed for v1. Undo pushes a snapshot of the shape list, capped at 100. Documents are small.
- Electron / Tauri. Killed. This is a local web app. A file is opened and saved through the browser file APIs the checks name. No custom server.
- Infinite history in the JSON file. Killed. The file stores the current scene only, not the undo stack.

## Frozen decisions

- One document open at a time. New document discards the in-memory scene. If dirty, confirm "Discard unsaved changes?" Discard / Cancel.
- Camera: `x`, `y` are the world coordinates at the viewport center. `zoom` is a number from 0.1 to 4, default 1. Wheel zooms toward the cursor. Space+drag or middle-mouse pans. Arrow keys do not pan (they nudge the selection, below).
- World origin is the center of a new document. Positive x right, positive y down, in SVG units. Store numbers as finite floats. On save, round to 2 decimal places.
- Shape set: `rect`, `ellipse`, `arrow`, `text`. No groups.
- z-order is array order. The last shape is on top. Hit test walks from the end and returns the first hit.
- Selection: click selects one. Shift-click toggles one. Drag on empty space rubber-bands and selects every shape whose bounds intersect the band, replacing the selection unless Shift is held. Delete and Backspace delete the selection. Arrow keys nudge 1 world unit, Shift+arrow nudges 10.
- Resize: selected `rect` and `ellipse` show eight handles. Minimum size 8 by 8. Arrow has two endpoint handles, no bounding-box rotate. No rotation handle on any shape. Text has no resize; the box grows with the text up to 400 width, then wraps.
- Colors are frozen: fill `#F7F7F4`, stroke `#202020`, stroke width 2, text 16px system UI. No color picker, no fonts.
- Dirty flag sets on any scene change and clears on successful save or on load. The tab title prefix is `• ` when dirty.
- Autosave is not in v1. Save is `Ctrl+S` and a button. The browser may still warn on `beforeunload` when dirty.

## Data model

### Document

| Field | Type | Notes |
| --- | --- | --- |
| version | number | Always `1`. Reject other versions with "This file is not an Ink document." |
| shapes | Shape[] | |

Camera is not saved. On open, fit the viewport to the shape bounds with 48px padding, or reset to origin if there are no shapes.

### Shape

| Field | Type | Notes |
| --- | --- | --- |
| id | string | Internal, `crypto.randomUUID()`. Not shown. |
| kind | `rect` \| `ellipse` \| `arrow` \| `text` | |
| x | number | World. For arrow, start x. |
| y | number | For arrow, start y. |
| w | number | For arrow, end minus start x. May be negative. |
| h | number | For arrow, end minus start y. May be negative. |
| text | string or null | `text` shapes only. Max 500 characters. |

Derived, do not store: selection, camera, undo stack, hover.

Bounds for hit testing: for rect and text, the axis-aligned box `(x, y, w, h)` with w/h positive in storage (normalize on commit so x,y is top-left and w,h > 0). For ellipse, the same box. For arrow, a hit if the pointer is within 6 screen pixels of the segment (convert the tolerance to world units using zoom).

## Screens

1. **Board.** Full viewport SVG. Top bar: New, Open, Save, Undo, Redo, zoom percentage, and the tool name. Tools are buttons: Select, Rect, Ellipse, Arrow, Text. Empty document: no illustration; the board is blank and the bar reads "No shapes." Error: a banner for file errors, board still visible. Loading: only "Opening…" while reading a file the user picked.

Not screens: login, share dialog, layers, templates gallery, presentation mode.

## Interaction spec

### Draw a shape

- Success: Rect, ellipse, and arrow are a drag. On mouse-up, if both |w| and |h| are under 4, discard the shape (mis-click). Otherwise commit, select it, and switch the tool back to Select. Text is a click that inserts a text shape with text `Text` and selects it for editing. Enter commits the edit, Escape cancels the edit and restores the previous text (for a new text shape, Escape deletes it).
- Cancel: Esc during a drag discards the in-progress shape and does not push undo. Undo after a committed shape restores the previous snapshot.
- Conflict: 2,000 shapes already in the document. Further creates are refused. Banner: "This document is at the 2000 shape limit." Deletes still work.

### Save and open

- Success: Save uses the File System Access API `showSaveFilePicker` when available, suggesting `diagram.ink.json`. If the API is missing, download `diagram.ink.json`. Open uses `showOpenFilePicker` or an `<input type="file">` fallback. After save, dirty clears.
- Cancel: user dismisses the picker. Document stays dirty. No error banner.
- Conflict: JSON parse failure or `version !== 1`. Do not replace the current document. Banner: "This file is not an Ink document."

## Edge cases

| Case | Required behavior |
| --- | --- |
| Overlapping shapes, click | The later shape in the array is selected. |
| Rubber-band that touches a shape edge | Shape is selected. Touching bounds is enough. |
| Zoom at 0.1 and 4 | Further wheel in that direction does nothing. Do not clamp by resetting the camera. |
| File with an extra unknown field on a shape | Ignore the field on load. Do not fail. Round-trip drops it. |
| File missing `version` | "This file is not an Ink document." |
| Text of 501 characters | Input refuses the 501st character. |
| Reload with dirty document and no save | Browser `beforeunload` prompt. If the user proceeds, changes are gone. That is acceptable. Do not add silent `localStorage` backup. |

## Visual constraints

- Board background `#F7F7F4`. Shapes use the frozen colors. Selection stroke is 2px `#1D4ED8`, handles are 8px squares.
- 13px system UI font in the bar. No logo, no watermark in the SVG.
- Icons: Lucide only in the bar.
- No gradients, no dotted "cute" grid. A grid is allowed: 32px world units, 1px line at 20% black, non-scaling-stroke so it does not thicken when zoomed. If the grid is hard, skip it. Do not spend the phase on the grid.
- No minimap.

## Technical constraints

- Stack: Vite, React, TypeScript strict. Package manager: npm. No backend.
- Render shapes as SVG elements. Do not add a canvas library, a diagram library, or a state library.
- Persist only through the save/open actions above. MIME/extension: `.ink.json`.
- Undo stack in memory only, max 100 snapshots of the shapes array. Coalesce typing in a text edit into one snapshot on commit, not one per keystroke.
- Record exact versions in the project README. Do not upgrade mid-phase.

## Phases and acceptance checks

### Phase 1 — Draw, pan, zoom, hit-test, undo

In scope: the four tools, select, delete, nudge, camera, undo/redo, shape limit. Save can be a download of the JSON so the file format is real; Open can be a file input.

Checks:

1. Draw two overlapping rects. Click the overlap. The later rect is selected. Delete removes only that rect.
2. Undo restores it. Redo removes it again.
3. Pan and zoom. Reload is not required in this check. The shapes stay in world position (select one, note coordinates in a temporary debug readout if needed, pan away, come back, coordinates unchanged). Remove any debug readout before calling the phase done.
4. Drag a 2 by 2 rect. It is not added. Undo count unchanged.
5. Esc mid-drag does not add a shape.

### Phase 2 — File round-trip

In scope: dirty flag, beforeunload, save picker or download, open, version rejection, camera fit on open.

Checks:

1. Draw a rect and an arrow. Save. Reload the page (confirm discard). Open the file. Both shapes are back, within 0.01 of the saved coordinates.
2. Open a text file that is not JSON. Current document, if any, is unchanged. Banner: "This file is not an Ink document."
3. Dirty document, close tab: the browser warns. Cancel stays on the page.

### Phase 3 — Viewport PNG

In scope: Export PNG button downloads the current viewport, including the grid if you drew one, excluding the toolbar. Shapes clipped to the viewport.

Checks:

1. Export does not change the document or the dirty flag.
2. The PNG dimensions match the board element's pixel size, not the whole monitor.

## Agent operating rules

You are implementing a frozen spec.

- Read this file before writing code. If a chat message conflicts with this file, stop and ask which wins. Do not silently follow the chat.
- Implement the named phase only.
- Do not add features, screens, settings, dependencies, or abstractions this file does not require.
- If you want to add something, write it in `OUT-OF-SCOPE.md` and do not build it.
- If the spec is silent, choose the simplest behavior that preserves the data model and record it in `DECISIONS.md`. Do not invent a product decision.
- If the spec is contradictory, stop on that point only.
- A phase is done only when its checks pass, including empty, error, and cancel paths.
- No unwired placeholder UI. No sample diagram left in the initial state.
- Do not copy branding, assets, or text from any commercial whiteboard.
- Do not add networking, accounts, or a sync endpoint.

## Open questions

None. If you find one, stop and report it. Do not guess a product answer.
