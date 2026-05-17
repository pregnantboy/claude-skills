---
name: plan-writer
description: Write an exhaustive implementation plan to a dated markdown file under ./docs/plans/ after brainstorming, before any code is written. Use this whenever the user says "write the plan", "save the plan", "let's plan this out", "capture this plan", "document the plan", or signals that brainstorming is converging and it's time to commit the plan to disk. Also use proactively at the end of any planning or design discussion, even if the user doesn't explicitly ask, so the plan survives a /clear and can be resumed in a fresh session. Do NOT use for writing code, executing a plan, or general documentation.
---

# Plan Writer

Writes a self-contained implementation plan to disk so the next Claude session (after /clear) can execute it without re-reading the planning conversation.

## When to invoke

- User says "write the plan", "save the plan", "let's commit this to a plan file", "document this", "capture what we discussed".
- Brainstorming has converged. Approach is decided. Files-to-touch are roughly known. No code has been written yet.
- User mentions they're about to /clear or start fresh.

If brainstorming is still open (decisions not made, alternatives not weighed), do not invoke. Push back: "We haven't decided X yet, want to settle that first?"

## Output location

`./docs/plans/YYYY-MM-DD-[slug].md`

Slug is kebab-case, derived from the feature name. Example: `2026-05-17-mrf-password-protected-links.md`. Date is today's date.

Create the `./docs/plans/` directory if it doesn't exist.

## Plan template

Write the file using this exact structure. Section headers are mandatory. Content under each section is mandatory unless explicitly marked optional. Replace square-bracket placeholders with real content.

```markdown
# [Feature name]

**Status:** Not started
**Created:** YYYY-MM-DD
**Last updated:** YYYY-MM-DD

## Resume from here

[One paragraph. What's the current state? What's the next concrete action? Update this after every phase completes. A fresh session should read only this paragraph to know what to do next.]

## Context

[Why this work exists. What problem it solves. Who asked for it or what triggered it. Link to any tickets, Slack threads, prior plans. Two to four paragraphs.]

## Goals

[Bulleted list of what success looks like. Concrete and verifiable.]

## Non-goals

[Bulleted list of what is explicitly out of scope. This is as important as goals. Prevents scope creep mid-execution.]

## Decisions made

[For each major decision: the decision, two to three alternatives considered, why the chosen one won. This is the section that justifies the plan.]

- **[Decision topic]:** Chose X over Y and Z because [reason].
- **[Decision topic]:** Chose A over B because [reason].

## Files touched

[Every file that will be created, modified, or deleted. Path + one-line description of what changes.]

- `path/to/file.ts` (modify) - [what changes]
- `path/to/new-file.ts` (create) - [what it does]
- `path/to/old-file.ts` (delete) - [why]

## Phases

[Break the work into phases. Each phase produces a verifiable result. Use checkboxes. Each phase has: what, files touched in this phase, verification step.]

### Phase 1: [Name]

- [ ] [Task]
- [ ] [Task]

Files: `[path]`, `[path]`
Verify: [how to confirm this phase is done, e.g. "tests in X pass", "endpoint returns 200 with payload Y"]

### Phase 2: [Name]

- [ ] [Task]
- [ ] [Task]

Files: `[path]`
Verify: [how]

## Edge cases considered

[Bulleted list. Each edge case and how the plan handles it. If unhandled, say so.]

## Verification

[How the whole thing gets tested end to end. What command to run. What output proves it works.]

## Open questions

[Anything unresolved. If none, write "None."]
```

## Filling in the template

- **Exhaustive on the decision surface, not on prose.** Every decision gets its rationale. Every file gets listed. Every edge case gets named. But don't pad sections with filler.
- **Self-contained.** Someone (or fresh-context Claude) reading only this file should be able to execute. Don't reference "what we discussed earlier".
- **Concrete file paths.** Not "the auth module", but `src/auth/session.ts`.
- **Verifiable phases.** Each phase ends with something that can be checked. If you can't write a verify step, the phase is too vague.
- **Decisions over discussion.** If brainstorming weighed three approaches, capture all three under "Decisions made" with the rationale. Don't dump raw discussion.

## After writing

1. Confirm the file path back to the user.
2. Suggest: "You can /clear now and start a fresh session with: `Read ./docs/plans/[filename] and execute phase 1.`"
3. Do not start executing. This skill writes plans, it does not execute them.
