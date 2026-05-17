---
name: plan-executor
description: Execute a plan file from ./docs/plans/ phase by phase, updating checkboxes and the "Resume from here" section as work progresses. Use this whenever the user says "execute the plan", "run the plan", "start the plan", "implement the plan", "continue the plan", "resume the plan", or references a plan file in ./docs/plans/. Also use when the user starts a fresh session asking to read a plan file from ./docs/plans/ and start work. Do NOT use for writing or brainstorming plans, only for executing them.
---

# Plan Executor

Executes a plan file written by plan-writer. Keeps the plan file as the source of truth for progress so work can be resumed across sessions.

## When to invoke

- User says "execute the plan", "run the plan", "start the plan", "continue", "resume".
- User points at a file in `./docs/plans/`.
- User starts a fresh session after /clear asking to read a plan file from ./docs/plans/ and start.

## Startup sequence

1. **Locate the plan.** If the user named a file, read it. If they said "the plan" without naming one, list `./docs/plans/` and ask which one (or pick the most recent if only one is in-progress).
2. **Read the entire plan file.** Especially the "Resume from here" section. That paragraph tells you where to start.
3. **Confirm scope with the user before any code changes.** State: "I'll start with phase [N]: [name]. Files I'll touch: [list]. Verify step: [step]. Proceed?" Wait for confirmation.
4. **Mark the plan status.** Update `**Status:**` line to "In progress" if not already.

## Execution loop

For each phase, in order:

1. Read the phase's tasks and files.
2. Do the work. Use subagents (general-purpose Task tool) for read-heavy investigation: "go check how X is currently implemented across the codebase and summarize". Do not use subagents for the actual code changes, those happen in the main session.
3. After each task within a phase, update the checkbox in the plan file from unchecked to checked.
4. After the phase is done, run the phase's verify step. If it fails, stop and surface the failure to the user.
5. Update "Resume from here" to describe what's done and what's next. One paragraph. Update "Last updated" to today's date.
6. Pause and ask the user before starting the next phase, unless they've said "do the whole plan without stopping".

## Resume behavior

If the plan file has any checked boxes on entry, you are resuming, not starting fresh. In this case:

1. Read "Resume from here" to understand where the previous session left off.
2. Identify the next unchecked task.
3. Confirm with the user: "Last session left off at [X]. Next up is [Y]. Continue from there?"
4. Proceed.

## Updating the plan as you go

The plan file is a living document during execution. Update it for:

- **Checkbox state.** Every completed task gets ticked. Do this in the same message as the work, not in a batch at the end.
- **"Resume from here".** After each phase, rewrite this paragraph. It's the one thing a fresh session will read first.
- **Deviations.** If you had to deviate from the plan (e.g., a file you thought existed didn't, or a decision turned out wrong), add a `## Deviations` section at the bottom with date and what changed. Do not silently change the plan body.
- **Open questions resolved.** If something in "Open questions" got answered during execution, move it to "Decisions made" with the resolution.

## Hard rules

- **Do not re-plan during execution.** If you discover the plan is wrong, stop and surface it to the user. Let them decide whether to update the plan or proceed. Do not silently take a different approach.
- **Do not skip the verify step.** Each phase has a verify step for a reason. Run it.
- **Do not execute multiple phases in one go without checkpointing.** Update the plan file between phases. Otherwise a crash loses progress.
- **If context is getting full,** stop, update "Resume from here" with maximum detail, and tell the user: "Context is filling. Suggest /clear and resume with: `Read ./docs/plans/[file] and continue from where it left off`."

## When the plan is done

1. All checkboxes ticked.
2. Final verify step passes.
3. Update `**Status:**` to "Complete".
4. Update "Resume from here" to "Done. [One-line summary of what was shipped]."
5. Tell the user it's done and ask if they want anything else (commit message, PR description, follow-up tasks).
