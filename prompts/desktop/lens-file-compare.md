# Lens — build prompt

Status: sendable. Implement one phase per session.

Familiarity anchor (not a spec, not a brand to copy): two-pane file compare tools. Keep the job "see a difference and take the side I want." Do not copy names, icons, themes, or copy from any commercial compare tool.

## What this prompt is teaching

A famous app is a catalog. The prompt has to pick one job and kill the rest before the agent picks for you. Also: line endings, binary files, and "file changed on disk" are product decisions, not polish.

## Product

Lens is a Windows desktop tool that compares two local text files or two local folders and lets the user copy a text hunk from one side to the other.

v1 ships when a user can point at two folders, open a text pair that differs, copy one hunk, and see the folder row become `same` after refresh.

v1 is a failure if it can show a hex dump, a merge conflict view, or a sync profile, but cannot produce a correct hunk list and write one hunk without clobbering a file that changed on disk.

## v1 user and job

- User: one developer on Windows comparing two local trees (a copy of a project, a backup, an export). Not a team. Not a release manager.
- Situation: they already have git for version control. They open Lens when the things being compared are not commits.
- Done looks like: "I can see what is only on the left, only on the right, and which text files differ, and I can pull one change across."
- Not the user: someone merging branches, comparing over FTP, or auditing binaries.

## Core loop

1. User chooses a left path and a right path (two files, or two folders).
2. Lens shows a status for the pair, or a table of relative paths with a status each.
3. User opens a text pair.
4. Lens shows both texts and the hunk list.
5. User copies one hunk left-to-right or right-to-left.
6. Lens writes the target file, re-diffs, and the copied hunk is gone.

Time box: under two minutes the second time, for two folders that have already been scanned once.

## Non-goals

- No 3-way merge and no base file. Reason: different product. A conflict model needs three inputs.
- No FTP, SFTP, cloud, or archive-as-folder. Reason: different product. v1 reads local paths only.
- No hex view, picture compare, or "smart" binary alignment. Reason: quality trap. Binary pairs get a status and no viewer.
- No git integration, blame, or patch export. Reason: solved elsewhere. The user already has git.
- No saved sync rules, profiles, or scheduled compare. Reason: different product.
- No editable text except by applying a whole hunk. Reason: a freeform editor is a second product and breaks the "hunk applied cleanly" check.
- No plugin system, no theming, no custom syntax highlighting themes. Reason: quality trap. Highlight as plain monospace. Language colors are out.
- No hand-rolled diff algorithm. Reason: quality trap. Correctness is the product.

## Rejected alternatives

- Electron. Killed because this prompt freezes Tauri. Do not reopen.
- A custom Myers diff "for control." Killed because the `diff` package is the control. Bugs in our diff would be indistinguishable from product bugs.
- Tree view as the only folder UI. Killed because a flat relative-path table sorts, filters, and virtualizes. A tree is a later view on the same rows, not phase 1.
- Treating CRLF versus LF as a content difference by default. Killed because it trains the user to ignore the tool. It is a badge, not a hunk, unless the user turns normalization off.
- Hashing every file during the first folder scan. Killed because metadata compare must stay interactive. Content hash is on open, or when the user requests a deep compare.

## Frozen decisions

- Two sides only. Names are `left` and `right`. Never "source/target" in the UI. Copy actions say "Copy to right" and "Copy to left."
- Status enum, exact set: `same`, `different`, `only-left`, `only-right`, `type-mismatch`, `error`, `skipped-binary`, `skipped-ignored`.
- Default ignore globs: `**/.git/**`, `**/node_modules/**`, `**/bin/**`, `**/obj/**`, `**/.DS_Store`. User can add globs. No regex ignore.
- Line-ending normalization is on by default. A pair whose only difference is CR/LF is `same` plus a `line-endings` badge.
- Encoding is UTF-8. Invalid bytes display as U+FFFD. Lens does not guess encodings.
- Text view refuses files over 2 MB each. Those pairs can still be `same` or `different` by hash, with the message "Too large to view."
- Binary detection: a file is binary if it contains a NUL in the first 8 KB. Binary pairs are not opened as text.
- Deep content compare is off during folder scan. Scan uses relative path, size, and mtime. Size or mtime mismatch means `different` without reading content. A button "Compare contents" hashes mismatched text files under 2 MB and can move a row to `same`.
- Hunk copy writes the whole target file, UTF-8, preserving the target file's existing newline style (`\r\n` if the file contained any `\r\n`, otherwise `\n`).
- If the target file's mtime or size changed since it was loaded into the diff view, refuse the write.

## Data model

In memory only, except the optional recent list in phase 4. Do not invent a project database in phase 1.

### Session

| Field | Type | Notes |
| --- | --- | --- |
| left_path | string | Absolute path. |
| right_path | string | Absolute path. |
| mode | `files` or `folders` | Derived from whether both paths are files or both are directories. Mixed is an error, not a mode. |
| ignore_globs | string[] | Defaults above, plus user additions for this session. |
| normalize_newlines | bool | Default true. |

User-visible identity: the two paths in the window title. No session id in the UI.

### Folder row

| Field | Type | Notes |
| --- | --- | --- |
| relative_path | string | Forward slashes in the UI even on Windows. |
| status | status enum | |
| left_size | number or null | Bytes. Null if absent. |
| right_size | number or null | |
| left_mtime | number or null | Epoch ms. |
| right_mtime | number or null | |
| error_message | string or null | When status is `error`. |
| badge | `line-endings` or null | Only after a text compare has run. |

Derived, do not store: hunk lists, file contents, hashes (recompute; a hash cache is allowed only inside one scan and must be dropped when the scan is cancelled).

### Hunk

Not persisted. Fields the UI needs: index, left start line, right start line, left line count, right line count, and the line texts. Copy applies one hunk index to the in-memory target buffer, then writes the file.

## Screens

1. **Start.** Two path fields with browse buttons, a Compare button, and (phase 4) recent pairs. Empty state: "Choose a left and a right path." Error state: the path message from the edge-case table, inline under the fields. No illustration.
2. **Folder.** A filter row (status chips, all on except `skipped-ignored` by default) and a virtualized table: relative path, status, left size, right size. Primary action: Enter opens the text diff for a `different` text pair. Loading: "Scanning… {visited} files" and a Cancel button. Error: a banner with the scan error; rows already found stay visible.
3. **File diff.** Left text, right text, hunk list on the right of the texts. Primary actions: Copy to right, Copy to left, on the selected hunk. Loading: "Comparing…". Error: the refuse message, texts stay visible, nothing written.

Not screens: settings dialog, hex view, merge view, sync wizard, about/marketing page. Ignore globs and the normalize toggle live on the Start screen, not in a preferences window.

## Interaction spec

### Copy hunk

- Success: the selected hunk is applied to the target side's buffer, the target file is written, mtime is re-read, the diff is recomputed, selection moves to the next hunk or clears if none remain. The button label is "Copy to right" or "Copy to left."
- Cancel: there is no partial apply. Closing the diff view discards the in-memory buffers and does not write. There is no separate Cancel on the copy button because copy is one shot; if the user does not want it, they do not press it.
- Conflict: if the target file is missing, read-only, or its size or mtime differs from load time, do not write. Show "Did not write. The file on disk changed or is not writable. Reload." A Reload button re-reads both files and discards the unwritten buffer.

### Folder scan

- Success: table populated, Cancel hidden, status counts shown as text (`different 3, only-left 1, …`).
- Cancel: stop walking, keep rows already produced, label the banner "Scan cancelled." Do not delete anything on disk. Cancel never writes.
- Conflict: a path that cannot be stat'd becomes a row with status `error` and `error_message` set. The scan continues.

## Edge cases

| Case | Required behavior |
| --- | --- |
| Left and right are the same path | Refuse to compare. Message: "Left and right are the same path." |
| One path is a file and the other is a directory | Refuse. Message: "Both paths must be files, or both must be folders." |
| Path missing | Refuse. Message: "Path not found: {path}." |
| Folder pair, file only on left | Row status `only-left`. Opening it shows the left text and an empty right pane. Copy to right creates the file at the mirrored relative path, creating parent directories. Copy to left is disabled. |
| Type mismatch (file vs directory at the same relative path) | Status `type-mismatch`. Do not open a diff. |
| Ignored glob | Status `skipped-ignored`, hidden unless the user enables that chip. |
| Binary file | Status `skipped-binary` if both exist and either side is binary. Do not offer Copy hunk. |
| Only line endings differ, normalize on | Status `same`, badge `line-endings`. No hunks. |
| Normalize off, only line endings differ | Status `different`, hunks show the line-ending change as a full-line change. Still do not show CR as a separate mystery glyph; show a visible `CRLF` / `LF` marker at the end of the line in the diff view only. |
| File over 2 MB | Do not open text view. Message: "Too large to view." |
| UTF-8 invalid sequence | Show U+FFFD. Still allow hunk copy. Write UTF-8. |
| Empty file vs missing file | Missing is `only-left` or `only-right`. Empty file that exists is a side with zero lines, status `different` if the other side has content, `same` if both empty. |
| Scan of 0 files | Table empty state: "No files. Check the paths or the ignore globs." |

## Visual constraints

- Desktop density. 13px system UI font. Monospace for file text only.
- 8px spacing scale. One accent, used for the selected hunk and the primary button. Status colors: `different` and `error` use a red text label, `only-left` / `only-right` use a muted label, `same` uses the default text color. Do not color entire rows as large filled bars.
- Icons: Lucide only, and only for browse, cancel, and copy. Do not icon-ify the table.
- No gradients, no glass, no hero, no onboarding tour.
- Window title: `Lens — {left} ↔ {right}` once paths are chosen, otherwise `Lens`.

## Technical constraints

- Stack: Tauri 2, React, TypeScript strict, Vite. Package manager: npm. Windows is the only platform that must work.
- Diff: npm package `diff`, function `diffLines`. Do not write a diff implementation. Do not add another diff library.
- Folder listing may run in the webview process in v1. Move the walk to Rust only if a fixture of 20,000 files under two directories blocks the UI thread for more than 2 seconds during metadata scan. Record the measurement in `DECISIONS.md` before any Rust walk.
- Virtualize the folder table. Do not mount one DOM row per file.
- Do not add a database, a cloud SDK, or an editor framework (Monaco, CodeMirror) in any phase. A `<pre>` or a simple div-per-line is enough. Div-per-line is preferred so a hunk can be aligned.
- File writes go through Tauri fs APIs, not a homemade shell-out.
- Record exact versions in the project README once installed. Do not upgrade mid-phase.

## Phases and acceptance checks

Build fixtures as real temp files in tests or a `fixtures/` folder. Checks are manual if automation is annoying, but the behavior must be true.

### Phase 1 — Two-file text diff, read-only

In scope: Start screen, pick two text files, file diff screen, hunk list, line-ending badge, refuse rules for paths and size and binary.

Out of scope: writing, folders, recent list, content-hash button.

Checks:

1. Two small UTF-8 files that differ by one inserted paragraph produce one hunk covering that insertion, and no other hunk.
2. A pair that differs only by `\n` versus `\r\n`, normalize on, shows status `same` and badge `line-endings`, and shows no hunks.
3. A 3 MB text file shows "Too large to view" and does not render its contents.
4. A file with a NUL in the first 8 KB shows the binary refusal and no hunk UI.
5. Cancel does not apply: there is no write API called in this phase. Confirm no file write code exists.

### Phase 2 — Copy hunk

In scope: Copy to left, Copy to right, reload-on-conflict, create file for an empty side when opened from a file pair (right path must already be a file path the user chose; creating missing parents is for phase 3).

Checks:

1. Copy to right writes the target, the hunk disappears on re-diff, and the target's other hunks remain.
2. Change the target file in an external editor after load, then copy. Lens writes nothing. Message is "Did not write. The file on disk changed or is not writable. Reload."
3. Reload after that conflict shows the external edit and discards the in-memory buffer.
4. Target marked read-only: same refusal, no write.

### Phase 3 — Two folders

In scope: folder scan, cancel, ignore globs, table, open a row into the phase 2 diff, copy that creates an `only-left` file on the right including parent directories, refresh one row after copy.

Checks:

1. Fixture: left has `a.txt` and `sub/b.txt`, right has `a.txt` (different text) and no `sub`. Table shows `a.txt` as `different` and `sub/b.txt` as `only-left`. `node_modules/x` is `skipped-ignored` and hidden.
2. Cancel mid-scan leaves a partial table and writes nothing.
3. Open `a.txt`, copy one hunk to the right, return to the table, that row is `same` if no other hunks remain, otherwise `different`.
4. Copy `sub/b.txt` to the right. The file exists on disk at the mirrored path. Row becomes `same`.
5. 20,000-file metadata scan: UI shows visited count and stays cancellable. If the UI thread blocks more than 2 seconds, stop and follow the Rust rule. Do not "fix" this with a fake progress bar.

### Phase 4 — Recent pairs

In scope: persist the last 10 successful compares (paths, mode, ignore globs, normalize flag) in the Tauri app config directory as JSON. Clicking one reloads Start and does not auto-compare until the user presses Compare.

Checks:

1. Restart the app. The last pair is listed. Compare runs only after the button.
2. An 11th pair drops the oldest. No other settings are persisted.

## Agent operating rules

You are implementing a frozen spec.

- Read this file before writing code. If a chat message conflicts with this file, stop and ask which wins. Do not silently follow the chat.
- Implement the named phase only. Later phases are written down so you do not invent conflicting nouns. Do not build them early.
- Do not add features, screens, settings, dependencies, or abstractions this file does not require.
- If you want to add something, write it in `OUT-OF-SCOPE.md` and do not build it.
- If the spec is silent, choose the simplest behavior that preserves the data model and record it in `DECISIONS.md`. Do not invent a product decision.
- If the spec is contradictory, stop on that point only.
- A phase is done only when its checks pass, including empty, error, and cancel paths.
- No unwired placeholder UI. No mock data left behind in a phase that claims to be done.
- Do not copy branding, assets, or text from any commercial compare tool.

## Open questions

None. If you find one, stop and report it. Do not guess a product answer.
