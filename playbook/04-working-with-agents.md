# Working with agents without undoing the prompt

The prompt file is the product. The chat is a build log. When those two disagree, you get the thrash you are trying to quit.

## Session shape

1. Score the prompt. Under 10, do not start.
2. Tell the agent which phase is in scope. Paste the whole prompt anyway, or attach the file. Non-goals must be in context. A phase section alone will be "helpfully" expanded.
3. Ask for a short plan that maps to that phase's checks. Reject any plan item that is not a check.
4. Let it build. Do not co-design in the same thread.
5. Run the acceptance checks yourself. File mismatches as bugs against a paragraph, not as new ideas.
6. Stop. Commit. Later phases get a new session that starts from the updated prompt, not from memory.

If the agent asks a product question, the prompt failed. Answer it in the file, then continue. Do not answer only in chat. Chat answers evaporate and contradict the file next session.

## Direction changes

Allowed mid-phase:

- A check failed.
- The spec contradicts itself. Fix the file, name the winner, continue.
- A frozen technical choice is impossible (library abandoned, API missing). Pick the nearest option that preserves behavior, write it under rejected alternatives, continue.

Not allowed mid-phase:

- New nouns.
- New screens.
- "While you're in there."
- Restyling.
- Replacing the stack because another one is more interesting.
- Expanding a non-goal because the demo felt thin.

When you truly want a new noun, finish or abandon the phase, edit the prompt, and start a session whose first message is: "The spec file changed. Diff your plan against it. Do not keep behavior the file removed."

## How to give feedback

Weak, and it causes redesigns:

> This feels off. Make it cleaner. Maybe a sidebar? Also I saw a tool that uses cards.

Strong:

> Phase 1 check 2 failed. `fixtures/crlf-only` is marked content-different. The spec says line-ending-only pairs are `same` with a `line-endings` badge when the normalize toggle is on. Fix that. Do not change the folder view.

Point at the heading. Say what happened. Say what must not move.

## How to keep scope from leaking

Add this to your first message, every time:

> Implement phase N only. If you want to add something this file does not require, write it in `OUT-OF-SCOPE.md` and do not build it. I will delete that file unread if the checks pass.

You will still get creep. The rule gives you a place to reject it without a design argument.

## Reviewing the result

Do not review by scrolling the UI and reacting. Review in this order:

1. Acceptance checks, by hand, including cancel and conflict paths.
2. Non-goals: search the diff for nouns you refused (merge, plugin, auth, sync). If they appear, revert them even if they work.
3. Data model matches the prompt's fields. Extra columns are scope.
4. Empty, loading, and error states exist for the slice.
5. Visual constraints, only after behavior is right. A beautiful wrong diff is a failed phase.

## When the agent is stuck

Stuck means it cannot satisfy a written check, not that it has a better idea.

Ask for the smallest reproduction of the failed check. Do not ask for a rewrite. If the check was wrong, that is your correction to make in the file, explicitly.

## When you are stuck

If you cannot describe the next slice as checks, stop building. Go back to [02-design-before-you-prompt.md](02-design-before-you-prompt.md). Building is a bad way to discover what you meant. You already tried that.

## Practice drills

Do these before you prompt a real tool again.

1. Take an old prompt you sent. Score it. Rewrite only the non-goals and the core loop. Do not let yourself add features during the rewrite.
2. Read [Queue](../prompts/web/queue-issues.md) and write the phase 3 checks in your own words without looking. Compare. Where you were vaguer, that is the muscle.
3. Pick a tool you use daily. Write the failure sentence and ten non-goals. Stop there. Most of the skill is the refusal list.
4. After your next agent session, mark every direction change as either "spec bug" or "I got bored." Only the first kind is allowed to edit the prompt mid-build. Count the second kind. That count is the metric.

## What improvement looks like

- Prompts get longer in decisions and shorter in adjectives.
- Sessions get shorter.
- You delete agent code that "also supports" something.
- You can resume a project a week later from the markdown alone.

That is the game. The six briefs are examples of the target, not assignments to implement all at once.
