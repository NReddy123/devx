# Design before you prompt

Do this on paper or in the template. Do not do it in the agent chat. The chat is where undecided design goes to multiply.

You are not trying to imagine every screen. You are trying to make the next hour of implementation unsurprising.

## Answer these, in order

If you skip one, the agent will answer it badly.

### 1. One user, one job

Not a persona poster. One person, this week.

- Who is holding the keyboard in v1?
- What are they trying to finish?
- What do they use today, and what is intolerable about it?
- What will they do in the five minutes after your tool works?

If you have two users with different jobs (guest and host, author and reader), name both, and say which one the core loop belongs to. Slot has two. Lens has one. That is a real difference, not a template quirk.

### 2. The core loop, as steps you could time

Write 3–7 steps. If you need more, you have several products.

Each step is a user action or a system response, not a subsystem. "Index the vault" is not a step. "Type a link, click it, land on that note" is.

The loop is done when the user has the outcome, not when the UI is "set up."

### 3. The failure sentence

Finish this: "v1 is a failure if it can ___, but cannot ___."

Lens: "v1 is a failure if it can theme the window, but cannot show a correct text diff and copy one hunk."

This sentence is the priority filter for every later idea, including your own.

### 4. Non-goals, each with a reason

Aim for a list longer than the feature list. Reasons are one of:

- **Different product.** Needs new nouns. Example: 3-way merge.
- **Different user.** Example: team permissions on a single-user tool.
- **Solved well enough elsewhere.** Example: git, when the user already has git.
- **Quality trap.** Example: hand-rolled diff, fuzzy search, realtime multiplayer. You will spend the project here and never ship the loop.
- **Later, and the model already allows it.** Example: a board view on top of a fixed status field. Say so, so you do not "design ahead" by building it now.

### 5. Nouns, then fields

List the nouns you will store. For each, the fields another programmer could create a table from. Name the identity the user sees (`Q-14`, not a UUID). Name what is immutable.

If you cannot name the fields, you do not have a feature yet. You have a wish.

Also write what is derived and must not be stored. Harbor backlinks are derived from scanning note bodies. Storing them creates a cache invalidation project.

### 6. The hardest interaction, including the unhappy path

Every app in this workspace has one interaction that will be wrong if you only say "support X."

- Lens: copy a hunk while the target file may have changed on disk.
- Snip: annotate in image pixels, then flatten only on copy or save.
- Harbor: resolve `[[note]]` when two stems collide, on a case-insensitive disk.
- Queue: status change is a field update, not a workflow engine.
- Ink: hit-test the topmost shape; undo restores a snapshot.
- Slot: a slot offered in the guest's zone must still be free in UTC when the booking commits.

Write the rule, the refusal, and what the user sees. "Handle errors gracefully" is not a rule.

### 7. Frozen technical choices

Pick the stack in the prompt. Include the thing you will not let the agent swap (package manager, diff library, SVG versus canvas, no auth).

You are allowed to be wrong. You are not allowed to leave it open. An open stack becomes a research project.

Choose the boring option that can finish the current phase. Move work to Rust, a queue, or a service only behind a measurement you wrote down.

### 8. Phase 1 as a manual test

Phase 1 is the smallest slice where the core loop is visible, even if ugly and narrow.

Write checks as actions:

- Given these two files, the hunk list is this.
- Given cancel, no file is written.
- Given an empty vault, the empty state names the next action.

"Works well" is not a check.

## How to know you are ready to write the prompt

You can explain the v1 in two minutes without saying the reference product's name, except once, as a familiarity anchor. You can say what you cut and why. You can name the first click-through.

If you are still excited about optional features, they are not cut hard enough. Put them in non-goals until the loop exists.

## A note on taste

Visual taste is not the same as design. Design here means: what exists, what it means, what happens when it fails, what you will not build.

Taste still matters, but it is a constraint, not a mood. "Dense, system font, one accent, 8px grid, Lucide icons, no gradients" will be implemented. "Make it stunning" will be renegotiated every session.

Look at real tools you respect and steal **constraints**, not screenshots:

- Dense information, not marketing whitespace, for tools used for hours.
- One primary action per screen.
- Keyboard path for the core loop if the user is a developer.
- Empty states that say what to do, not an illustration and a slogan.
- The same word for the same noun everywhere. Do not say "issue" in the UI and "ticket" in the API.

## Reference products are anchors

Use a famous app to calibrate ("the feeling of a two-pane compare"). Then write your slice so the agent never needs that app installed, logged into, or screenshotted.

If a sentence only makes sense if the agent has used the reference product, delete it and write the behavior.
