# Why prompts fail

You already know the symptom: vague prompts, direction changes, an agent that builds the wrong thing with confidence. The cause is usually one of five moves. All five showed up in the file-compare attempt.

## 1. The competitor's name was used as the spec

"Beyond Compare clone" feels specific. It is not. That product is decades of decisions: folder compare, text rules, binary rules, hex, 3-way merge, FTP, sync, picture compare, version control integration, saved sessions, alignment overrides.

An agent hears the name and samples a random subset. You see the subset and remember a different subset. That is not the agent being stupid. The prompt named a catalog, not a job.

**Replace the name with a job.** "Show me how two folders differ and let me copy a text hunk from one file to the other." Someone who has used a compare tool should recognize that job. They should not receive the catalog.

Lens is that rewrite. Read it after this file.

## 2. Nothing was refused

Agents fill silence. If you do not list non-goals, the empty space becomes features: themes, plugins, accounts, command palettes, a settings page with twelve toggles, a hand-rolled diff "so we control it."

Every non-goal needs a reason. "No 3-way merge" is weak. "No 3-way merge, because that needs a base plus two sides and a different conflict model, and v1 is two sides only" is a decision. The reason stops you from casually adding it on Thursday.

## 3. The build was horizontal

Typical thrash:

1. Ask for the whole app.
2. Agent scaffolds routes, a design system, and empty screens.
3. You react to the screens and change the product.
4. The data model, which nobody wrote down, no longer fits.
5. Repeat.

A phase is a vertical slice: one path a human can perform, including the failure path. Lens phase 1 is not "the shell." It is "open two text files and see the diff." If that is wrong, you find out before anyone builds folder sync.

## 4. Product questions were disguised as implementation details

These are product questions, even though they sound technical:

- Is a line-ending change a difference?
- If the file changed on disk, do we overwrite it?
- Are `Notes.md` and `notes.md` the same note on Windows?
- Does a cancelled booking free the slot immediately?
- Is undo a stack of commands or a snapshot?

If you do not answer them, the agent will, and it will answer differently in each file. You will then "change direction" to undo an answer you never gave. That feels like a prompting problem. It is an undecided design.

Write the answer in the prompt. If you truly do not care, write the default and say the agent must not reopen it.

## 5. Direction changes arrived as chat nudges

"Actually make it more like a hex editor."
"Can we also sync to S3?"
"Scrap the table, I want a tree."

Each of those throws away work because the previous prompt allowed it. A nudge feels cheap. It is a new product.

Rule: if the change adds a noun, a screen, or a new meaning for an existing field, edit the prompt file first, then tell the agent the file changed. If you cannot write the change cleanly in the spec, you do not understand it yet. Do not make the agent discover that for you.

## What a good session feels like

Boring. The agent asks almost nothing. The arguments you have are about whether a check passed, not about what the product is. When you are surprised, the surprise is a bug against the spec, and you can point at the paragraph.

If a session is exciting and full of ideas, the prompt was not done.

## A bad prompt, for contrast

Do not send this:

> Build a Beyond Compare clone with Tauri and React. It should look modern and professional. Support folder compare, file diff, and merge. Make the UX smooth. Add whatever features make sense.

Why it fails, line by line:

| Sentence | Damage |
| --- | --- |
| "Beyond Compare clone" | Imports an unbounded catalog and a trademark you should not ship. |
| "look modern and professional" | No constraint. Invites a redesign every time the agent shows a screen. |
| "folder compare, file diff, and merge" | Three products. Merge contradicts two-pane compare. |
| "UX smooth" | Not testable. |
| "whatever features make sense" | Permission to invent, which you will then reject. |

The fix is not longer adjectives. The fix is decisions. The rest of this playbook is how to make them before the session, not during it.
