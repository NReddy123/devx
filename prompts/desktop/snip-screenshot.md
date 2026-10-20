# Snip — build prompt

Status: sendable. Implement one phase per session.

Familiarity anchor (not a spec, not a brand to copy): region screenshot tools that annotate before copying. Keep "select a rectangle, mark it, put the image on the clipboard." Do not copy names, icons, hotkeys, or marketing from any shipping capture tool.

## What this prompt is teaching

The capture interaction and the annotation model are the product. History, OCR, video, and scrolling capture are how a small tool becomes a platform. Also: store annotations in image pixels, or undo and resize will lie.

## Product

Snip is a Windows desktop tool that grabs a rectangle from the primary monitor, lets the user draw a few marks on it, and copies a flattened PNG to the clipboard.

v1 ships when a global hotkey leads to a copied image that contains the region plus the marks, and Esc at any point before copy leaves the clipboard unchanged.

v1 is a failure if it can record video or OCR text, but cannot cancel cleanly or undo a mark.

## v1 user and job

- User: one person on Windows who needs to show someone a screen region, with one or two arrows or a box, several times a day.
- Situation: they are already in another app. Snip must get out of the way until the hotkey.
- Done looks like: "I hit the hotkey, drag a box, draw an arrow, copy, and paste into chat."
- Not the user: someone recording bugs as video, redacting documents for court, or capturing a scrolling web page.

## Core loop

1. User presses the hotkey (default `Ctrl+Shift+1`).
2. Snip hides its editor if open, dims the primary monitor, and waits for a drag.
3. User drags a rectangle and releases.
4. The editor opens on that bitmap, fit-to-window, with the arrow tool selected.
5. User draws zero or more marks and presses Copy (or `Ctrl+C`).
6. The flattened PNG is on the clipboard. The editor stays open.

Time box: under 20 seconds for a region and one arrow, the fifth time.

## Non-goals

- No scrolling capture, no multi-monitor pick, no window-picker by hovering windows. Reason: different product. Primary monitor, one rectangle.
- No video, GIF, or timed capture. Reason: different product.
- No OCR, no upload, no account, no share link. Reason: quality trap and different product. Copy and save are the exits.
- No numbered steps, blur brushes that stay live, speech bubbles, or sticker packs. Reason: the tool list is frozen below. More tools are a new prompt.
- No global "capture history cloud." Reason: different product. Local last 10 is in phase 3 and is files on disk, not a service.
- No plugin or hotkey chord editor beyond a single changeable hotkey. Reason: quality trap.

## Rejected alternatives

- Drawing marks by burning pixels immediately. Killed because undo, move, and cancel-before-copy get much harder. Marks stay vectors until flatten.
- HTML canvas as the only model. Killed for v1. The editor shows the bitmap as an image and draws marks as SVG in the same pixel coordinate space. Do not re-propose canvas unless SVG text measurement blocks phase 2, and record why in `DECISIONS.md` before switching.
- Electron. Killed because the stack is frozen to Tauri 2.
- A custom hotkey format (`Ctrl-Shift-S` strings the agent invented). Killed. Use Tauri's global shortcut string and the default above.

## Frozen decisions

- Primary monitor only. If capture APIs require a monitor id, use the primary. Do not offer a monitor picker.
- Minimum rectangle: 8 by 8 physical pixels. Smaller drags cancel the capture and do not open the editor.
- Coordinate space for marks: image pixels of the captured bitmap, origin top-left. Not CSS pixels of the editor window. Zoom in the editor does not change stored coordinates.
- Tools, exact set: `select`, `arrow`, `rect`, `text`, `blur`. No freehand pen.
- Undo is a stack of document snapshots (the mark list), capped at 50. Redo is required and is the inverse of that stack. Flatten does not add an undo step; Copy does not destroy the mark list.
- Blur is a mark, not a destructive edit, until flatten. On flatten, blur pixelates the bitmap under its rectangle (block size 10 image pixels) and is then baked in. Other marks render above the blurred bitmap.
- Flatten order: bitmap, then blur marks in z-order, then non-blur marks in z-order. z-order is list order. Later marks are on top.
- Default arrow and rect stroke: 3 image pixels, color `#E23B2F`. Text: 20 image pixels, same color, system UI font. No color picker in v1.
- Copy writes a PNG to the Windows clipboard and does not clear the editor.
- Save writes a PNG via the native save dialog, default directory the user's Pictures folder, default filename `snip-YYYYMMDD-HHmmss.png`.
- Esc on the overlay cancels capture. Esc in the editor does nothing (so it does not throw away marks). Close is explicit.

## Data model

### Capture

Not stored until phase 3.

| Field | Type | Notes |
| --- | --- | --- |
| bitmap_png | bytes | The raw region, no marks. |
| width | int | Image pixels. |
| height | int | Image pixels. |
| marks | Mark[] | |

User-visible identity: none until history exists. Do not show a UUID.

### Mark

| Field | Type | Notes |
| --- | --- | --- |
| id | string | Internal only. |
| kind | `arrow` \| `rect` \| `text` \| `blur` | |
| x | number | Image pixels. |
| y | number | |
| w | number | For arrow, w/h is the delta to the end point, and may be negative. |
| h | number | |
| text | string or null | Required for `text`, max 200 characters. Empty text marks are deleted on commit. |

Derived, do not store: the flattened PNG, the zoom level, the dim overlay.

## Screens

1. **Overlay.** Full primary monitor, dimmed, crosshair. Drag draws a 2px light rectangle. No toolbars. Esc cancels. Empty does not apply. Error: if capture fails, close the overlay and show a small always-on-top window: "Could not capture the screen." with a Close button. Do not open the editor.
2. **Editor.** Bitmap centered on a neutral background. Tool row: Select, Arrow, Rect, Text, Blur, Undo, Redo, Copy, Save. Zoom label as a percentage, plus fit. Status text for errors ("Copy failed."). Loading: "Capturing…" only between mouse-up and editor open. Empty: the editor is not shown without a bitmap.

Not screens: preferences beyond the hotkey on a single line in the editor footer, gallery (until phase 3), account, onboarding.

## Interaction spec

### Region drag

- Success: mouse-up with both sides ≥ 8 image pixels hides the overlay, captures that rectangle, opens the editor with the arrow tool selected and no marks.
- Cancel: Esc, or mouse-up below 8 pixels, or right-click. Clipboard unchanged. No file written. Overlay gone. Previous editor, if any, stays closed (do not restore a previous capture in v1; the new hotkey abandons the unsaved previous document). Abandoning a previous document is acceptable only before phase 3. In phase 3, a capture that is not copied or saved is still inserted into history when the editor opens, so abandon does not lose the bitmap.
- Conflict: capture API error uses the error window above. Clipboard unchanged.

### Draw and copy

- Success: arrow is a drag from start to end. Rect is a drag. Text is a click that places an input at that image point; Enter commits, Shift+Enter inserts a newline, Esc cancels that text mark only. Blur is a drag rectangle. Copy flattens and places PNG on the clipboard. Footer shows "Copied." for 2 seconds.
- Cancel: Undo returns the previous mark list. It does not restore a cancelled capture. Closing the editor window discards marks in memory. Phase 3 still has the bitmap in history from the moment the editor opened.
- Conflict: if clipboard write fails, marks remain, message "Copy failed." Do not retry silently in a loop.

## Edge cases

| Case | Required behavior |
| --- | --- |
| Hotkey already registered by another app | On launch, show the editor hidden and a dialog: "Hotkey Ctrl+Shift+1 is in use." Offer a single text field to set a replacement. Do not pick a random replacement. |
| DPI scaling | Capture and mark coordinates are physical image pixels of the bitmap you actually captured. The editor scales uniformly to fit. A mark's stored x/y does not change when the window resizes. |
| Zero marks, Copy | Copies the raw bitmap. Valid. |
| Text longer than 200 characters | Refuse the extra characters in the input. Do not truncate silently after commit. |
| Blur rectangle partly outside the bitmap | Clamp to bitmap bounds on commit. If the clamp has zero area, delete the mark. |
| User hits hotkey while overlay is already up | Ignore the second hotkey. |
| Save dialog cancelled | No file written. Editor unchanged. Message: none. |
| Very large region (over 20 megapixels) | Still allow. If the bitmap cannot be allocated, error window "Could not capture the screen." and no editor. |

## Visual constraints

- Overlay: black at 40% opacity, selection stroke 2px white. No logo on the overlay.
- Editor: 13px system UI font, 8px scale, one accent on the active tool. Neutral background `#1E1E1E` because screenshots are often white; the chrome around the bitmap must not be white.
- Icons: Lucide only, in the tool row.
- No gradients, no rounded marketing cards, no app name watermark on the exported PNG. The PNG contains pixels and marks only.

## Technical constraints

- Stack: Tauri 2, React, TypeScript strict, Vite. Package manager: npm. Windows only.
- Global shortcut via Tauri's global shortcut plugin. Clipboard image via a maintained Tauri 2 capability. Do not shell out to PowerShell to set the clipboard.
- Screen capture: use a maintained Tauri 2 plugin if one exists in the current stable line. If it does not, a small Rust command that captures the primary monitor rectangle is allowed. Do not build a cross-platform capture framework. Do not add OCR libraries.
- Flatten in the webview by drawing the bitmap and SVG marks to a canvas at export time only, at 1 image pixel = 1 canvas pixel. This is the one allowed use of canvas.
- History files in phase 3 live in the app config directory under `history/`, PNG plus a sidecar JSON of marks. Cap 10 captures. No database.
- Record exact versions in the project README. Do not upgrade mid-phase.

## Phases and acceptance checks

### Phase 1 — Capture region and copy unmarked PNG

In scope: hotkey, overlay, drag, cancel, editor showing the bitmap, Copy of the unmarked image, error window.

Out of scope: marks, save dialog, history, zoom beyond fit-to-window.

Checks:

1. Hotkey, drag a region, Copy, paste into Paint or another image app: the image matches the region, not the whole screen.
2. Esc during drag: overlay closes, clipboard is unchanged (test by copying a known image first).
3. Drag of 4 by 4 pixels: editor does not open, clipboard unchanged.
4. If capture is forced to fail (unplug nothing; simulate by unit-testing the error branch if a live failure is impractical), the error window reads "Could not capture the screen." and the editor does not open.

### Phase 2 — Marks, undo, flatten

In scope: the five tools, undo/redo, flatten rules, Save dialog, zoom fit plus 100%.

Checks:

1. Draw an arrow, undo, redo. Copy after undo does not contain the arrow. Copy after redo does.
2. Place text "hi", Copy, paste: the text is visible and the editor still has the text mark (copy does not bake the in-memory document).
3. Add a blur and an arrow on top of the same area. Copied PNG shows pixelation under the arrow, arrow not pixelated.
4. Resize the editor window. The arrow's tip stays on the same bitmap pixel (verify by copying before and after resize; pixels match).
5. Save, then Cancel in the dialog: no new file. Save, confirm: PNG exists and matches a Copy from the same document.

### Phase 3 — Local history

In scope: when the editor opens, write bitmap and empty mark sidecar into history, update sidecar when marks change (debounce 300 ms), list last 10 in a left column, click to reopen. New capture beyond 10 deletes the oldest files.

Checks:

1. Restart the app. The last capture is in the list and reopens with marks.
2. Eleventh capture removes the oldest files from disk, not only from the UI.
3. Hotkey still abandons the current editor view, but the previous capture remains in history.

## Agent operating rules

You are implementing a frozen spec.

- Read this file before writing code. If a chat message conflicts with this file, stop and ask which wins. Do not silently follow the chat.
- Implement the named phase only.
- Do not add features, screens, settings, dependencies, or abstractions this file does not require.
- If you want to add something, write it in `OUT-OF-SCOPE.md` and do not build it.
- If the spec is silent, choose the simplest behavior that preserves the data model and record it in `DECISIONS.md`. Do not invent a product decision.
- If the spec is contradictory, stop on that point only.
- A phase is done only when its checks pass, including empty, error, and cancel paths.
- No unwired placeholder UI. No mock data left behind in a phase that claims to be done.
- Do not copy branding, assets, or text from any commercial capture tool.
- Do not watermark, upload, or phone home with screenshot contents.

## Open questions

None. If you find one, stop and report it. Do not guess a product answer.
