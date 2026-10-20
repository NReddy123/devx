# Prompt anatomy

A build prompt is a decision record that an agent happens to execute. It is not a pitch, a brainstorm, or a chat starter.

Each brief in [prompts/](../prompts/) uses the same headings. Keep them in this order. Agents skim. Order is a form of priority.

Copy [prompt-template.md](prompt-template.md) when you start a new one. Do not write freeform.

## How to read Lens against this list

[Lens](../prompts/desktop/lens-file-compare.md) is the fully worked version of the file-compare job. Use it as the reference implementation of this anatomy.

| Heading | What it is for | What goes wrong if you skip it |
| --- | --- | --- |
| What this prompt is teaching | For you, not the agent. Names the design lesson. | You collect prompts and learn nothing. |
| Product | One paragraph. Name, job, platform. | The agent writes a landing page for a platform. |
| v1 user and job | Who, situation, success in their words. | Features arrive for imaginary teammates. |
| Core loop | Numbered, timed, ends in an outcome. | The app grows a shell and no path. |
| Non-goals | Refusals with reasons. | Silence becomes a roadmap. |
| Rejected alternatives | Options you considered and killed, including technical ones. | The agent re-proposes them as improvements. |
| Frozen decisions | Short list of choices that are not up for debate this phase. | Every file re-decides line endings, IDs, undo. |
| Data model | Nouns, fields, identity, derived values. | UI-first code with nowhere honest to put state. |
| Screens | The few surfaces, and what is not a screen. | A route per idea. |
| Interaction spec | The hard behavior, including refuse/cancel/conflict. | Happy-path demo, corrupt files on the first real use. |
| Edge cases | Table. Input, required behavior. | "Should be fine" bugs you cannot file precisely. |
| Visual constraints | Numbers and prohibitions. | Perpetual restyling. |
| Technical constraints | Stack, libraries, what not to swap, performance bar. | Framework tourism. |
| Phases and acceptance checks | Vertical slices, pass/fail. | Big-bang failure. |
| Agent operating rules | How to behave when the spec is silent or the chat disagrees. | Helpful scope creep. |
| Open questions | Should be empty. If not, the prompt is not sendable. | The agent guesses, you call it a direction change. |

## Rules for the prose

- Write behavior, not praise. "Fast" fails. "A 20,000-file pair must become interactive within 2 seconds of metadata compare, content hashes excluded" can fail a test.
- Use the product's nouns and no synonyms. Pick "hunk" and stay there.
- Prefer "refuse and show this message" over "handle gracefully."
- Put numbers on caps. Max file size, max shapes, max title length. Unbounded is how toys fall over and how agents build platforms.
- Specify empty, loading, and error for every screen that fetches or scans.
- One phase = one session. The prompt may contain later phases so the model is shaped correctly, but the agent is told to stop.
- Do not include implementation code. Algorithms belong only where the choice matters ("use the `diff` package, Myers, do not write one").
- Do not paste screenshots of commercial products. Describe layout in words: regions, order, density.
- Keep the prompt shorter than the app. If a section is speculative, it is a non-goal.

## What "standalone" means

Assume the agent has not read this playbook. Repeat the operating rules inside the prompt. A prompt that only works if the agent remembers last week's chat is not a prompt. It is a dependency.

## What not to put in

- Market analysis, competitor matrices, "unique value propositions."
- User stories for v3.
- A design system with tokens for components you do not have.
- Motivational language ("delight the user," "world-class").
- Options. "Either Electron or Tauri" is not a decision. Pick one.
- Secret keys, accounts, or instructions to copy proprietary source.

## A finished prompt is quiet

Read it once as the agent. Highlight any sentence that allows two different products. Rewrite those sentences until only one product remains.

Then run [prompt-scorecard.md](prompt-scorecard.md).
