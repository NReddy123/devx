# Prompt scorecard

Score the prompt before you send it. One point each. **Send only on 10.**

A 7 that you send anyway is how the file-compare project melted. The missing three points are the ones the agent will improvise, and you will call the improvisation "bad output" or "I changed my mind."

## The ten

1. **One user.** A stranger can name who is at the keyboard in v1, and what they are finishing. "Users" plural, with no primary, scores 0.
2. **Timed loop.** The core loop is 3–7 steps, ends in an outcome, and contains no step that is really a subsystem ("add auth," "build the engine").
3. **Failure sentence.** There is a sentence of the form "v1 fails if it can ___, but cannot ___."
4. **Non-goals with reasons.** At least five refusals, each with a reason class: different product, different user, solved elsewhere, quality trap, or later-without-new-nouns.
5. **Rejected alternatives.** At least three options you killed, including one technical option (a library, a framework, an algorithm, a storage choice). The agent will re-suggest these unless they are dead on the page.
6. **Data model.** Every stored noun has fields, a user-visible identity where the user sees one, and a note about what is derived and must not be stored.
7. **Hard interaction.** The riskiest action specifies success, cancel, and at least one conflict or invalid case, including the text or state the user sees.
8. **Frozen stack.** Framework, language, package manager, and the library choice that you refuse to hand-roll are named. There is no "or."
9. **Phase 1 checks.** The first phase is a vertical slice with pass/fail checks a human can run, including an empty or error path. "UI skeleton" scores 0.
10. **No open product questions.** The open-questions heading exists and says none. Defaults you do not care about are written as frozen decisions, not questions.

## Automatic fails, even if the ten pass

- The prompt says "clone &lt;commercial product&gt;" as the requirement, instead of as a one-line familiarity anchor.
- It tells the agent to copy proprietary source, assets, or branding.
- It says "and anything else that makes sense," "surprise me," or "use your judgment for the product."
- Two sections contradict each other. Fix the file. Do not leave the agent to pick a winner.
- Phase 1 cannot be demoed without phase 3.

## How to use a miss

Do not negotiate with yourself. A miss means add the missing decision, not "we'll figure it out in implementation."

If you cannot fill a point, the slice is too big. Cut the product until the point is easy. A small prompt that scores 10 beats a complete vision that scores 6.
