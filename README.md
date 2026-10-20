# Devx prompt gym

This workspace is a set of worked examples for writing prompts that an agent can build from without you changing direction every twenty minutes.

It is not a starter repo for six apps. Nothing here is implemented on purpose. The artifact is the prompt.

## The point

"Build a Beyond Compare clone" is not a design. It is a name. The agent then invents a product, you disagree with the invention, and both of you thrash.

A usable prompt names a job, freezes a slice of that job, and refuses the rest. The apps below are familiarity anchors only. The products in [prompts/](prompts/) are original, smaller tools that keep the part people actually hire those apps for.

Do not copy trademarks, icons, marketing copy, or assets from the reference products into anything you build.

## Read in this order

1. [playbook/01-why-prompts-fail.md](playbook/01-why-prompts-fail.md) — the failure you already lived through, named precisely.
2. [playbook/02-design-before-you-prompt.md](playbook/02-design-before-you-prompt.md) — the questions to answer before you open an agent session.
3. [playbook/03-prompt-anatomy.md](playbook/03-prompt-anatomy.md) — how those answers become a file an agent cannot wiggle out of.
4. [playbook/04-working-with-agents.md](playbook/04-working-with-agents.md) — how to run the file without undoing it in chat.
5. [playbook/prompt-scorecard.md](playbook/prompt-scorecard.md) — gate. If a prompt scores under 10, do not send it.

Then read one desktop brief and one web brief all the way through. Start with [prompts/desktop/lens-file-compare.md](prompts/desktop/lens-file-compare.md). It is the file-compare lesson, rewritten as a spec.

## The briefs

Desktop:

| Prompt | Hired to | Reference, not a spec |
| --- | --- | --- |
| [Lens](prompts/desktop/lens-file-compare.md) | See two folders or two text files differ, and copy the change you want | Two-pane compare tools |
| [Snip](prompts/desktop/snip-screenshot.md) | Grab a region, mark it up, put it on the clipboard | Region screenshot tools |
| [Harbor](prompts/desktop/harbor-notes.md) | Write markdown notes in a folder and follow links between them | Local markdown vaults |

Web, all runnable on localhost, none of them a SaaS:

| Prompt | Hired to | Reference, not a spec |
| --- | --- | --- |
| [Queue](prompts/web/queue-issues.md) | File a unit of work and move it through a fixed workflow | Issue trackers |
| [Ink](prompts/web/ink-whiteboard.md) | Sketch a diagram on an infinite surface and find it tomorrow | Lightweight whiteboards |
| [Slot](prompts/web/slot-booking.md) | Offer bookable times without double-booking or timezone lies | Scheduling pages |

## How to practice

1. Pick a small tool you wish existed. Not a famous one. Something you would use this week.
2. Copy [playbook/prompt-template.md](playbook/prompt-template.md) to `prompts/mine/<name>.md`.
3. Fill every section. Delete no heading. If a heading does not apply, write why.
4. Score it. Fix it until the scorecard passes.
5. Only then paste **one phase** to an agent, with the non-goals and frozen decisions still attached.

When you want to change direction, edit the markdown first. Chat is not the spec.
