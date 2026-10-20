# Harbor — build prompt

Status: sendable. Implement one phase per session.

Familiarity anchor (not a spec, not a brand to copy): local markdown vaults with wiki links. Keep "a folder of notes I can link and search." Do not copy names, icons, themes, plugin APIs, or graph visuals from any shipping notes app.

## What this prompt is teaching

The vault is the filesystem. The moment you add a database as the source of truth, you have signed up for sync and corruption stories. Backlinks are a query, not a feature you store. A graph view is the usual way this product explodes.

## Product

Harbor is a Windows desktop tool that opens a folder of markdown files, edits one note at a time, and follows `[[wiki links]]` between filenames.

v1 ships when a user can create a note, link to another note, click the link, come back, and see the first note listed under the second note's backlinks, after a restart, with no account.

v1 is a failure if it can draw a graph or install a plugin, but cannot resolve a link or survive a restart without a hidden database.

## v1 user and job

- User: one person writing notes on their own machine.
- Situation: they want files they can read without Harbor, in a folder they picked.
- Done looks like: "My notes are markdown files. I can link them and find them."
- Not the user: a team collaborating in realtime, or someone publishing a wiki to the web.

## Core loop

1. User opens a vault folder (or creates an empty one).
2. User creates a note and types.
3. User inserts a link to another note by name.
4. User clicks the link and lands on that note.
5. User opens backlinks on the target and sees the source note.
6. User quits, reopens the vault, and the note body on disk matches what they typed.

Time box: under three minutes the second time, including relaunch.

## Non-goals

- No graph view, no canvas, no daily-note calendar UI. Reason: quality trap. A "today" button that creates `YYYY-MM-DD.md` is allowed in phase 3 because it adds no noun.
- No plugins, themes, or custom CSS. Reason: different product.
- No sync, no git UI, no accounts, no publish. Reason: solved elsewhere or different product. The folder can sit in whatever sync the user already uses. Harbor must not run git.
- No WYSIWYG editing. Reason: quality trap. The editor is a textarea. Preview is rendered markdown beside it.
- No image paste, attachments folder, or transclusion (`![[note]]`). Reason: different product. Phase 1 links are navigation only.
- No fuzzy search and no semantic search. Reason: quality trap. Search is case-insensitive substring.
- No tags, YAML frontmatter features, or properties panel. Reason: different product. If the user types frontmatter, it is body text.

## Rejected alternatives

- SQLite as the source of truth, with export to markdown. Killed because the job is "files I can read without this app." The markdown file is the source. An in-memory index is rebuilt on open.
- CodeMirror or Monaco in v1. Killed. A textarea is enough until the checks pass. Do not add an editor framework in passing.
- Electron. Killed. Stack is Tauri 2.
- Case-sensitive link resolution. Killed. Windows cannot have `Note.md` and `note.md` as two files. Resolution is case-insensitive. Creating a note uses the typed name, but if a case-insensitive stem match exists, open that file instead of creating a second one.
- Wikilink aliases and heading anchors (`[[Note#Heading|label]]`). Killed for v1. The grammar is `[[Stem]]` only. A stem is the filename without `.md`. No path separators in a stem.

## Frozen decisions

- A vault is a single directory the user picked. Harbor does not walk parent directories. It does read nested folders. Ignore directories named `.git` and `node_modules`.
- A note is a file ending in `.md`, case-insensitive. Other files are listed in a side list labeled "Files" but are not notes and cannot be linked. Do not render them.
- Note identity for links: the stem, compared case-insensitively, unique within the vault. If two files in different folders share a stem, the link is ambiguous. Do not guess.
- Create note: ask for a stem, reject empty, reject `.` `..` and any of `\ / : * ? " < > |`, max 80 characters. Write `stem.md` into the vault root, not into the currently open note's folder. If the file exists, open it instead.
- Save: explicit `Ctrl+S` and also save when switching notes. Debounced autosave at 1 second after the last keystroke. Write UTF-8 without BOM. Do not convert the user's line endings: preserve `\r\n` if the file had any, otherwise `\n`. New files use `\r\n` on Windows.
- Delete: moves the file to the Windows recycle bin if the Tauri API can do that. If it cannot, refuse with "Could not move to recycle bin. Nothing was deleted." Do not unlink permanently.
- Preview: GitHub-flavored markdown via `marked`, sanitized via `dompurify`. Wiki links are not markdown. Render them in preview as buttons, not as `<a href>` to the filesystem.
- Search: substring, case-insensitive, over stem and body, max 100 results. Show stem and the first matching line. No ranking beyond "stem matches first, then body."

## Data model

Source of truth: the files. The index is memory only.

### Note index entry

| Field | Type | Notes |
| --- | --- | --- |
| absolute_path | string | |
| stem | string | Filename without extension. |
| stem_key | string | `stem` lowercased. |
| folder_relative | string | Forward slashes, `""` for vault root. |
| mtime | number | Epoch ms, used to refresh one entry. |

Derived, do not store: backlinks, outgoing links, search scores, preview HTML.

Outgoing links are computed by scanning the body for `[[` + stem + `]]`. The stem inside may contain spaces. It must not contain `]` or a newline. Unclosed `[[` is not a link.

Backlinks for note A: every note whose outgoing links contain A's `stem_key`. Ambiguous keys still backlink to every file with that key; the UI says the name is ambiguous.

### Open buffer

| Field | Type | Notes |
| --- | --- | --- |
| absolute_path | string | |
| text | string | |
| dirty | bool | |
| loaded_mtime | number | For the conflict rule. |

## Screens

1. **No vault.** One button: "Open folder." Empty state is this screen. Error: "Could not open that folder."
2. **Vault.** Left column: filter box, note list (stem and relative folder), New note, Today (phase 3). Main: stem as the title (not editable in v1; rename is phase 3), textarea, preview. Right column: outgoing links and backlinks. Primary action: type in the textarea.

Empty vault: "No notes. Create one." in the list.
Loading: "Reading vault…" with the count of markdown files seen.
Error: banner "Could not read {path}." The rest of the index stays.

Not screens: graph, settings, publish, daily calendar, word count dashboard.

## Interaction spec

### Open a wiki link

- Success: unique `stem_key` match opens that note, saving the current buffer first. The preview button and a `Ctrl+Click` on the `[[stem]]` text in the textarea both do this. (Ctrl+Click is required because a plain click in the textarea places the caret.)
- Cancel: if the current note is dirty and save fails, do not navigate. Message: "Could not save. Stayed on this note."
- Conflict: zero matches. Preview shows the link text and a Create button. Create writes `{stem}.md` in the vault root and opens it. Ambiguous match (two files, same stem_key): do not navigate. Show "Ambiguous: {path1} and {path2}." and list both as buttons.

### Save vs disk change

- Success: write the buffer, refresh mtime, clear dirty.
- Cancel: no cancel control on autosave. Quitting with a failed save shows "Could not save {stem}. Quit anyway?" with Stay and Quit. Quit discards the dirty buffer. Stay returns to the note.
- Conflict: if `loaded_mtime` does not match the file on disk at save time, do not write. Message: "This note changed on disk. Reload or Overwrite." Reload replaces the buffer. Overwrite writes anyway and updates `loaded_mtime`.

## Edge cases

| Case | Required behavior |
| --- | --- |
| Stem collision on create, different case | Open the existing file. Do not create `Note.md` beside `note.md`. |
| Two folders, both have `Log.md` | Link `[[Log]]` is ambiguous. UI lists both paths. |
| `[[Missing]]` | Not an error banner. Preview offers Create. |
| File renamed outside Harbor | On next focus of the window, rescan. If the open path is gone, show "This note is no longer on disk." and do not recreate it unless the user creates it. |
| Note body contains `[[a]]` and also the letters `[[a` | Only the closed form is a link. |
| Binary-ish `.md` with NUL | Skip it in the index. Do not crash the scan. |
| Vault with 5,000 markdown files | Index them. The list must stay usable (filter the rendered rows). Do not add search infrastructure beyond the substring scan. |
| User links with a path `[[folder/Note]]` | Not a link. The grammar does not allow `/`. Show as plain text. |

## Visual constraints

- 14px system UI font in the textarea. 13px in the lists. Preview uses the same font, not a "prose" theme from a library.
- 8px scale. One accent for the current note in the list and for link buttons.
- Icons: Lucide only, for new, search, and trash. No note-type icons.
- Dense list. No cards around notes. No cover images.
- Split: list 240px, editor and preview share the rest 50/50, links column 240px. Do not make it responsive-web. A minimum window of 960px is fine; below that, horizontal scroll is acceptable.

## Technical constraints

- Stack: Tauri 2, React, TypeScript strict, Vite. Package manager: npm. Windows only.
- Markdown: `marked` and `dompurify`. Do not add a second markdown parser. Do not enable raw HTML in markdown; pass the option that ignores HTML, and still sanitize.
- Watcher: rescan the open note's mtime when the window gains focus, and rescan the vault file list at that time if a previous scan finished. Do not add a native file watcher in v1 unless focus-rescan misses edits made while the window is focused from an external save the user expects to see. If you add a watcher, record why in `DECISIONS.md`. No sync engine.
- Do not add SQLite, search engines, or graph layout libraries.
- Record exact versions in the project README. Do not upgrade mid-phase.

## Phases and acceptance checks

### Phase 1 — Open vault, edit, save, survive restart

In scope: open folder, list notes, textarea, explicit save, autosave, quit confirmation, disk-conflict prompt.

Out of scope: preview, wiki links, search, rename, delete, today button.

Checks:

1. Empty folder: empty state text is "No notes. Create one." Create `Hello`, type `one`, quit, relaunch, open vault: body is `one`.
2. External edit while the note is open and dirty: save shows "This note changed on disk. Reload or Overwrite." Reload shows the external text. A second run, choosing Overwrite, keeps Harbor's text.
3. Failed navigation does not apply yet (no links). Confirm no hidden database file is created in the vault.

### Phase 2 — Links, preview, backlinks

In scope: preview pane, `[[stem]]` grammar, create-from-missing-link, ambiguous links, backlinks column, Ctrl+Click in the textarea.

Checks:

1. Note A contains `[[B]]`. Click in preview. B opens. B's backlinks lists A.
2. Restart. Backlinks still lists A, with no index file in the vault.
3. Two files named `Log.md` in different folders. `[[Log]]` shows "Ambiguous:" and both paths, and does not navigate until a path is chosen.
4. `[[No Such]]` shows Create. Create writes `No Such.md` in the vault root and opens it.
5. Preview does not execute HTML that was typed into the note (`<script>` or an equivalent). It shows as text or is stripped. It must not run.

### Phase 3 — Search, rename, delete, today

In scope: substring search, rename stem (writes a new filename, deletes the old via recycle-bin rule if rename API is not a true rename — prefer a filesystem rename in the same volume), delete to recycle bin, Today button creates `{local YYYY-MM-DD}.md` or opens it if present.

Checks:

1. Search `xyz` finds a note with `xyz` in the body and shows that line. It does not find `xy`.
2. Rename `A` to `B`. The file on disk is `B.md`. Existing `[[A]]` links do not rewrite (frozen: no refactor). They become missing links. Backlinks from the old name disappear.
3. Delete sends the file to the recycle bin. If that cannot be done, the file remains and the refusal message is shown.
4. Today twice in one day opens the same file, no duplicate.

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
- Do not copy branding, assets, or text from any commercial notes app.
- Do not write an index, database, or config into the user's vault folder. App config goes in the Tauri app config directory.

## Open questions

None. If you find one, stop and report it. Do not guess a product answer.
