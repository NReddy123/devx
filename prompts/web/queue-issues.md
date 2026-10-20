# Queue — build prompt

Status: sendable. Implement one phase per session.

Familiarity anchor (not a spec, not a brand to copy): single-team issue trackers with a fixed workflow and a keyboard-first list. Keep "capture work and move it through states." Do not copy names, icons, or marketing from any shipping tracker.

## What this prompt is teaching

Workflow customization is a second product. Freeze the statuses. A board, if you ever add one, is a view over those statuses, not a reason to add a workflow engine now. Also: web does not mean accounts, hosting, or a design system.

## Product

Queue is a local web app for one person to record units of work and move them through a fixed set of statuses.

v1 ships when a user can create an issue, find it in a filtered list, change its status from the keyboard, restart the process, and see the same issues.

v1 is a failure if it has customizable workflows, projects-with-permissions, or a notification system, but cannot create an issue and show it after restart.

## v1 user and job

- User: one maintainer tracking their own work on one machine.
- Situation: they want a list that is more structured than a text file and less machinery than a hosted tracker.
- Done looks like: "I filed it, I can see what is in doing, and it is still there tomorrow."
- Not the user: a company with roles, SLAs, and incoming email.

## Core loop

1. User opens the app and hits `c`.
2. User types a title and saves.
3. The new issue appears at the top of the list with a human id and status `inbox`.
4. User moves selection with `j` / `k` and sets status with a number key.
5. User restarts the server and the issue is still there with that status.

Time box: under one minute to file three issues and park one in `doing`.

## Non-goals

- No accounts, no multi-user, no assignees, no comments-from-other-people. Reason: different user. A comment entity is allowed in phase 3 as a local note, because the field list is frozen now and phase 1 must not block it. Do not build comments in phase 1.
- No custom statuses, no workflow editor, no per-project columns. Reason: different product. Statuses are the frozen enum below.
- No email, notifications, GitHub sync, or incoming webhooks. Reason: different product.
- No estimates, story points, cycles, or roadmaps. Reason: different product.
- No rich text. Description is markdown source in a textarea, previewed with the same rules as Harbor (`marked` + `dompurify`, no raw HTML). Reason: quality trap to build an editor.
- No board view in v1. Reason: later, no new nouns. A board would group by the existing status field. Do not add it "to look complete."

## Rejected alternatives

- Hosted SaaS with login. Killed. v1 is one local server and one browser.
- UUID as the visible id. Killed. Users cite ids out loud. Visible id is `{KEY}-{n}`.
- Custom status strings. Killed. The enum is the product boundary that keeps a future board from becoming a redesign.
- A client-only JSON file in `localStorage`. Killed. Restart-the-browser is not the same as restart-the-app, and the prompt's check is restart the server. SQLite is the store.
- React Query, Redux, or a client cache library. Killed for v1. Fetch on mutation and on load. The list is small.

## Frozen decisions

- One database file, `queue.sqlite`, in the process working directory. No migration framework. On boot, run `CREATE TABLE IF NOT EXISTS` for the tables below. Destructive schema changes are not allowed after phase 1 without a written migration in the prompt. Phase 3's comment table must be created in phase 1 as an empty table so later work is an insert, not a reshape. That is intentional. Do not also create tables you do not have fields for.
- Status enum, exact set and order: `inbox`, `ready`, `doing`, `done`, `cancelled`.
- Priority enum: `none`, `low`, `medium`, `high`, `urgent`. Default `none`.
- Project has a `key` of 2 to 5 uppercase ASCII letters, unique. Issue number `n` starts at 1 per project and never reuses, including after delete. v1 has delete. Deleting an issue does not compact numbers.
- Default project on first boot: key `GEN`, name `General`. Do not show a project wizard.
- Title required, 1 to 120 characters, trimmed. Description optional, max 20,000 characters.
- Labels: zero to eight per issue, each 1 to 24 characters, letters numbers spaces hyphen, stored lowercase. No label admin screen. Labels are created by typing them on the issue.
- Sort: `updated_at` descending. Filter is AND across status, priority, and a substring on title. Filters live in the query string so refresh keeps them.
- Keyboard (when not typing in a field): `c` create, `j` next, `k` previous, Enter open, Escape close detail, `1` inbox, `2` ready, `3` doing, `4` done, `5` cancelled. Keys do not fire while an input, textarea, or select is focused.

## Data model

### Project

| Field | Type | Notes |
| --- | --- | --- |
| id | integer | Internal. |
| key | string | User-visible, immutable after create. |
| name | string | |
| next_number | integer | The number that will be assigned to the next issue. |

### Issue

| Field | Type | Notes |
| --- | --- | --- |
| id | integer | Internal. |
| project_id | integer | |
| number | integer | Immutable. Visible id is `{key}-{number}`. |
| title | string | |
| description | string | Default empty. |
| status | status enum | Default `inbox`. |
| priority | priority enum | Default `none`. |
| created_at | string | ISO 8601 UTC. |
| updated_at | string | Set on every mutating request. |

### Issue label

| Field | Type | Notes |
| --- | --- | --- |
| issue_id | integer | |
| label | string | Lowercase. Unique per issue. |

### Comment

Created in phase 1. Unused until phase 3.

| Field | Type | Notes |
| --- | --- | --- |
| id | integer | |
| issue_id | integer | |
| body | string | Max 10,000 characters. |
| created_at | string | ISO 8601 UTC. No edit, no delete in v1. |

Derived, do not store: filters, selection index, preview HTML.

## Screens

1. **List.** Top: project name, filter chips for the five statuses, priority select, title search. Body: virtualized-enough list (windowing not required under 500 rows; do not add a virtualization library). Each row: visible id, title, status, priority, labels. Empty: "No issues. Press c to create one." if the database has zero issues. If filters exclude everything: "No issues match these filters." with a clear-filters control. Loading: "Loading issues…" Error: "Could not load issues." and a Retry button.
2. **Create / detail.** A panel over the list, not a separate site. Fields: title, description, status, priority, labels. Primary button: Save. Escape closes and discards unsaved create. For an existing issue, Escape closes and leaves the last save intact; unsaved edits prompt "Discard changes?" with Discard and Keep editing.

Not screens: board, admin, login, settings, reports, project switcher beyond a single select if a second project exists (phase 4).

## Interaction spec

### Create issue

- Success: `c` opens the panel with title focused, status `inbox`, priority `none`. Save with a valid title inserts the row, increments `next_number`, closes the panel, selects the new row, shows it under current filters if it matches, otherwise shows "Saved {id}, hidden by filters."
- Cancel: Escape on a new issue discards. Nothing inserted. `next_number` unchanged.
- Conflict: empty title does not submit. Inline message: "Title is required." Duplicate visible ids cannot happen if `next_number` is updated in the same transaction as the insert. If the transaction fails, show "Could not save." and do not increment.

### Set status from the list

- Success: number key updates the selected issue, sets `updated_at`, re-sorts the list, keeps selection on that issue even if it moves.
- Cancel: no toggle. Pressing the key for the current status is a no-op, not an error.
- Conflict: if the row was deleted (single user, so only if a second window deleted it), show "That issue no longer exists." and refresh the list. Do not recreate it.

## Edge cases

| Case | Required behavior |
| --- | --- |
| Title of 121 characters | Refuse. "Title must be 120 characters or fewer." Do not truncate. |
| Label `Bug` and `bug` | One label, `bug`. |
| Ninth label | Refuse. "An issue can have 8 labels." |
| Filter `doing` + create status inbox | Issue is saved and the toast says it is hidden by filters. It is not silently retitled into the active filter. |
| Two browser tabs | Last write wins. No CRDT, no presence. Acceptable. Do not build locking. |
| Delete issue (phase 4) | Row gone, comments gone, number not reused, list refreshes. |
| Server down | List error state with Retry. Do not cache a stale list and pretend it saved. |

## Visual constraints

- Light background, 13px system UI font, 8px scale, one accent for the selected row and the primary button.
- Status and priority are text labels, not large colored pills that dominate the title. A 8px dot is allowed. Color is secondary to the word.
- Icons: Lucide only, sparse. The row does not need an icon.
- No landing page, no pricing, no empty-state illustration, no app chrome that looks like a marketing site.
- The list is the product. The detail panel is at most 480px wide.

## Technical constraints

- Stack: one npm repo. Backend: Hono, Node, TypeScript, `better-sqlite3`. Frontend: Vite, React, TypeScript strict. Package manager: npm.
- Bind to `127.0.0.1` only, port 4177. Do not add auth to "compensate" for a public bind. Do not bind `0.0.0.0`.
- No ORM. Write SQL. Transactions for create (insert + increment) and for delete (issue + labels + comments).
- No Docker, no CI setup, no component library. Style with one CSS file.
- Markdown preview dependencies: `marked` and `dompurify` only, used in phase 2.
- Seed nothing except the General project. No demo issues.
- Record exact versions in the project README. Do not upgrade mid-phase.

## Phases and acceptance checks

### Phase 1 — Create, list, status, restart

In scope: server, schema including the empty comment table, General project, list, create panel, status keys, filters in the query string, error and empty states.

Out of scope: description preview, comments UI, extra projects, delete.

Checks:

1. `c`, title `First`, save. Row `GEN-1`, status `inbox`. Restart the server process. Reload the browser. The row is still `GEN-1`.
2. Select it, press `3`. Status is `doing`. It remains selected.
3. Escape during create does not insert. `GEN-2` is the next id after a cancelled create plus a real save, not `GEN-3`.
4. Empty database list message is "No issues. Press c to create one." Filters that match nothing use the other message.
5. Stop the server. The browser shows "Could not load issues." and does not claim a save succeeded.

### Phase 2 — Description, priority, labels, search

In scope: those fields, preview, title substring filter combined with status.

Checks:

1. Description containing `<script>` does not execute in the preview.
2. Label `Bug` stored and shown as `bug`. A second `bug` does not duplicate.
3. Query string `?status=doing&q=pay` keeps both filters after reload.

### Phase 3 — Comments

In scope: append-only comments on the detail panel. No edit, no delete.

Checks:

1. A comment survives restart and shows under the issue, oldest first.
2. Empty comment does not submit. Message: "Comment is empty."

### Phase 4 — Second project and delete

In scope: create project (key + name), switch project via a select (the list shows one project at a time), delete issue with a confirm dialog "Delete GEN-4? This cannot be undone."

Checks:

1. Project `API` issues start at `API-1`. `GEN` numbering is unchanged.
2. Delete removes the issue and its comments. The next issue in that project does not reuse the number.
3. Confirm dialog Cancel deletes nothing.

## Agent operating rules

You are implementing a frozen spec.

- Read this file before writing code. If a chat message conflicts with this file, stop and ask which wins. Do not silently follow the chat.
- Implement the named phase only.
- Do not add features, screens, settings, dependencies, or abstractions this file does not require.
- If you want to add something, write it in `OUT-OF-SCOPE.md` and do not build it.
- If the spec is silent, choose the simplest behavior that preserves the data model and record it in `DECISIONS.md`. Do not invent a product decision.
- If the spec is contradictory, stop on that point only.
- A phase is done only when its checks pass, including empty, error, and cancel paths.
- No unwired placeholder UI. No demo issues left behind.
- Do not copy branding, assets, or text from any commercial tracker.
- Do not add authentication, a public bind, or telemetry.

## Open questions

None. If you find one, stop and report it. Do not guess a product answer.
