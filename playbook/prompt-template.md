# {{Product name}} — build prompt

Status: draft. Do not hand to an agent until [prompt-scorecard.md](prompt-scorecard.md) is 10/10.

Familiarity anchor (not a spec, not a brand to copy): {{one known product, one sentence on which job you are keeping}}.

## What this prompt is teaching

{{One design lesson. For you. The agent may ignore this section.}}

## Product

{{Name}} is a {{desktop or local web}} tool that {{job in one sentence}}.

v1 ships when {{observable outcome}}.

v1 is a failure if it can {{impressive distraction}}, but cannot {{core outcome}}.

## v1 user and job

- User: {{who, singular}}.
- Situation: {{when they reach for this}}.
- Done looks like: {{their words, not yours}}.
- Not the user: {{who you are tempted to add}}.

## Core loop

1. {{action}}
2. {{response}}
3. {{action}}
4. {{outcome}}

Time box: a prepared user finishes this loop in {{N}} minutes the second time.

## Non-goals

- {{Refusal.}} Reason: {{different product | different user | solved elsewhere | quality trap | later, no new nouns}}.

## Rejected alternatives

- {{Option.}} Killed because {{reason}}. Do not re-propose it this phase.

## Frozen decisions

- {{Decision that must be identical in every file.}}

## Data model

### {{Noun}}

| Field | Type | Notes |
| --- | --- | --- |
| id | | User-visible identity: {{...}} |

Derived, do not store: {{...}}.

## Screens

1. {{Name}} — purpose, primary action, empty state text, error state.
2. {{Name}} — ...

Not screens: {{things that must not become routes}}.

## Interaction spec

### {{Hard interaction}}

- Success: {{...}}
- Cancel: {{no side effect, what is on screen}}.
- Conflict: {{refusal and the exact message, or the exact state}}.

## Edge cases

| Case | Required behavior |
| --- | --- |
| {{input}} | {{behavior}} |

## Visual constraints

- System UI font. 8px spacing scale. One accent. Dense tool layout, not a marketing page.
- Icons: Lucide only.
- No gradients, no glass, no custom illustration set.
- Every screen in this phase has empty, loading, and error states where data can be absent or fail.
- Same noun, same word, in UI and code.

## Technical constraints

- Stack: {{pick one}}. Package manager: npm.
- Use {{library}} for {{problem}}. Do not hand-roll it.
- Do not add services, auth, or native modules this phase does not need.
- Performance bar: {{measurement and fixture, or "not a concern below N"}}.
- Record exact versions in the project README once installed. Do not upgrade mid-phase.

## Phases and acceptance checks

### Phase 1 — {{name the slice}}

In scope: {{...}}

Out of scope: everything else in this file, even if described in detail so later phases do not invent new nouns.

Checks:

1. {{Given / when / then, runnable by hand.}}
2. {{Cancel or error check.}}

### Phase 2 — {{...}}

Checks:

1. {{...}}

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
- Do not copy branding, assets, or text from the familiarity-anchor product.

## Open questions

None. If you find one, stop and report it. Do not guess a product answer.
