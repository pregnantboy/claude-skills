---
name: code-reviewer
description: Perform rigorous, context-aware code reviews on TypeScript and Node.js changes, focusing on security and correctness. Use this skill whenever the user asks to "review" code, a PR, a diff, or their current changes, or says "look over this", "check my code", "is this safe", or anything indicating they want another pair of eyes on code. Auto-detects the review target via `gh pr status` if no PR is specified, falls back to uncommitted changes, and reads project conventions from CLAUDE.md, AGENTS.md, and `.claude/` before reviewing. Produces severity-ranked findings as compact cards (terminal-friendly) and a verdict inline in chat.
---

# Code Reviewer

A rigorous, context-aware code review skill for TypeScript and Node.js. Prioritizes security and correctness, minimizes noise, and respects existing project conventions.

## When to Use

Trigger whenever the user wants code reviewed:
- "Review my PR" / "Review PR #123" / "Review this"
- "Check my changes" / "Look over this code"
- "Is this safe?" / "Any bugs here?"
- "Review the uncommitted changes"

## Core Workflow

The review is a disciplined 5-phase process. Do not skip phases. Do not reorder.

1. **Target Resolution** — figure out what to review
2. **Context Gathering** — load project conventions and PR metadata
3. **Mode Selection** — decide light vs heavy based on scope
4. **Analysis** — systematic review against checklists
5. **Report** — produce the findings cards and verdict inline

Never post anything to GitHub. Output only in the chat.

---

## Phase 1: Target Resolution

Determine what to review in this order:

1. **Explicit PR given** (e.g., "review PR #42", "review PR 42"): use `gh pr view 42 --json ...`
2. **Current branch has a PR**: run `gh pr status --json currentBranch -q .currentBranch` first. If a PR exists on the current branch, use it.
3. **No PR on current branch**: check for uncommitted changes with `git status --porcelain`. If there are changes, review them (staged + unstaged combined via `git diff HEAD`).
4. **No PR and no uncommitted changes**: ask the user what to review. Do not guess. Offer: the last commit, a diff range like `main..HEAD`, or a specific file.

If multiple options are ambiguous (e.g., there's both a PR and uncommitted changes), ask the user which one to review.

### Commands reference

```bash
# Detect current branch PR
gh pr status --json currentBranch -q .currentBranch

# Fetch PR metadata (do this in ONE call to save round-trips)
gh pr view <NUM> --json number,title,body,author,headRefName,baseRefName,files,reviews,comments,commits,labels,isDraft

# Get the diff
gh pr diff <NUM>

# For uncommitted changes
git status --porcelain
git diff HEAD               # staged + unstaged vs last commit
git diff --stat HEAD        # summary for size estimation

# For explicit ranges
git diff main..HEAD
```

---

## Phase 2: Context Gathering

### Always read (if present)

Check and read these files from the repo root in this order. Skip any that don't exist. These establish project conventions:

1. `CLAUDE.md` (root)
2. `AGENTS.md` (root)
3. `.claude/` — read all `*.md` files inside, shallow only (do not recurse into subdirs unless obvious relevance)
4. `README.md` — skim for stack info only, not for conventions
5. `package.json` — for stack detection (TypeScript? Next.js? Express? Fastify? Node version?)
6. Linter configs: `.eslintrc*`, `eslint.config.*`, `biome.json`, `.prettierrc*` — their *existence* matters. If a linter is configured, skip style/formatting findings entirely (the linter owns those).

### For PR reviews, also read

- **PR title and body** — understand intent
- **Existing review comments and review bodies** — do not re-flag what another reviewer already raised. If another reviewer flagged something and the author pushed back, note the thread but do not re-litigate.
- **Commit messages** — sometimes reveal intent the PR body misses
- **Linked issues** — if the PR body references `#123` or `fixes #123`, fetch with `gh issue view 123 --json title,body` for context

### Stack detection

From `package.json`, note:
- TypeScript (strict mode? from `tsconfig.json` if easy to check)
- Framework: Next.js, Express, Fastify, NestJS, Hono, etc.
- Runtime: Node, Bun, Deno
- DB layer: Prisma, Drizzle, Knex, raw pg/mysql — affects SQL injection analysis
- Auth: NextAuth, Clerk, custom JWT — affects authn/authz analysis
- Validation: Zod, Valibot, Yup, Joi — affects input validation analysis

This tunes the security review. Raw `pg.query(sql)` with string concat is a critical SQL injection finding; the same pattern with Drizzle's query builder is fine.

---

## Phase 3: Mode Selection

Two modes control how deeply to read:

### Light mode (default)
- Read only the diff
- Do not load full files
- Best for focused PRs under 500 lines or 10 files

### Heavy mode
- Read full content of every changed file
- Read test files for changed source files (if they exist)
- Read one-hop imports: for each changed file, identify its top-level imports from within the repo and read those too (one level deep, no recursion)
- Best for PRs with architectural changes, new modules, or security-sensitive code

### Selection rules

- If the user specifies a mode (`"do a heavy review"`, `"light review"`), honor it.
- Auto-pick heavy if ANY of: diff > 500 lines, files changed > 10, PR body mentions "security", "auth", "permission", "secret", "crypto", or any file under `auth/`, `security/`, `middleware/`, or `permissions/` is touched.
- Otherwise default to light.
- Always announce the chosen mode at the start of the review ("Reviewing in heavy mode because..." or "Reviewing in light mode...").

---

## Phase 4: Analysis

Work through these checklists against the diff. For heavy mode, also apply them against the broader file context where relevant.

### Priority order
1. **Security** (see `references/security-checklist.md`)
2. **Correctness** (logic, edge cases, race conditions, error handling)
3. **TypeScript/Node specifics** (see `references/typescript-nodejs.md`)
4. **API/contract changes** (breaking changes, backward compat)
5. **Tests** (does new logic have tests? are existing tests still valid?)
6. **Maintainability** (only if something is genuinely hard to follow)

Read `references/security-checklist.md` for every review. Read `references/typescript-nodejs.md` for every TypeScript/Node review (which is the default for this skill).

### Anti-patterns — do NOT flag these

Suppress these to reduce noise:

- **Style and formatting** if any linter config exists in the repo. The linter owns these.
- **Missing tests** if the change is a pure refactor (no behavior change) or a type-only change.
- **Issues already raised in existing PR review comments** — even if the author hasn't addressed them yet. Trust the other reviewer.
- **Preference-level naming disputes** unless the name is actively misleading (e.g., `isEnabled` that returns count).
- **"You could use X library"** suggestions when the existing code works fine.
- **Speculative concerns** — every finding must tie to a concrete line and a concrete failure mode. If you cannot describe how it breaks, do not flag it.
- **Nits beyond a cap** — emit at most 5 `Info` items per review. If there are more, pick the 5 most impactful.

### Severity definitions

| Severity | Meaning |
|----------|---------|
| **Critical** | Security vulnerability or data-loss bug. Must fix before merge. |
| **High** | Correctness bug likely to manifest in production, or a significant regression. Should fix before merge. |
| **Medium** | Bug in an edge case, missing error handling on a non-critical path, or meaningful maintainability issue. Fix before merge unless there's a reason not to. |
| **Low** | Minor correctness concern, weak test coverage on a secondary path, or mild code smell. Address when convenient. |
| **Info** | Nit. Style preference, alternative approach, educational note. Non-blocking. |

---

## Phase 5: Report

Output the review inline in chat. No files, no GitHub posting. The format is a vertical card layout designed to render well on narrow terminals where wide tables wrap badly. Use this exact structure:

### Severity and category icons

Each finding leads with two icons: one for severity, one for category. Use these exact mappings:

**Severity icons:**
- 🔴 Critical
- 🟠 High
- 🟡 Medium
- 🔵 Low
- ⚪ Info

**Category icons:**
- 🔒 Security
- 🐛 Correctness
- ⚡ Performance
- 🧩 Types
- 🧪 Tests
- 🔌 API
- 🛠️ Maintainability
- 🎨 Style

### Format

```markdown
## Code Review: <PR title or "uncommitted changes" or range>

**Mode:** <light|heavy>
**Scope:** <N files, M lines changed>
**Context:** <brief note on what conventions were loaded, e.g., "Loaded CLAUDE.md and 2 files from .claude/">

### Findings

#### 1. 🔴 Critical · 🔒 Security
**Where:** `src/auth/login.ts:42`
**Issue:** Password compared with `==` allowing timing attacks.
**Fix:** Use `crypto.timingSafeEqual`.

#### 2. 🟠 High · 🐛 Correctness
**Where:** `src/api/users.ts:87`
**Issue:** `await` missing on async `updateUser()`, caller sees stale data.
**Fix:** Add `await` before the call.

#### 3. 🟡 Medium · 🧪 Tests
**Where:** `src/lib/parser.ts:120-145`
**Issue:** New branch for malformed input has no test coverage.
**Fix:** Add a case in `parser.test.ts` covering the malformed path.

### Summary

**Verdict:** <Approve | Comment | Request Changes>

**Top priorities:**
1. <most important thing to fix>
2. <second>
3. <third, omit if fewer than 3 issues>

**Done well:** <1-2 specific things, only if genuinely true. Do not invent praise. Skip this line if there's nothing genuine to say.>
```

### Rules for the findings cards

- Order findings by severity (Critical first), then by location within the same severity.
- Number cards sequentially across the whole list (`#### 1.`, `#### 2.`, ...). Numbering does not reset per severity.
- The card heading must be exactly `#### N. <severity icon> <Severity> · <category icon> <Category>`. Use a middle dot (·) as the separator.
- `Where` is `file:line` (or `file:start-end` for multi-line). Always use the path as it appears in the diff. Wrap it in backticks.
- `Issue` is one sentence describing the concrete problem. Name the failure mode.
- `Fix` is one sentence (or a short code snippet) that is actionable, not vague.
- For findings that need a code snippet, place a fenced code block on its own line after `**Fix:**` rather than inline.
- If there are no findings, write `✅ No issues found.` below the Findings header and skip the cards entirely.
- Keep each card tight. If you find yourself writing more than three lines of prose for `Issue`, you are probably restating the same point. Cut it down.

### Rules for the verdict

- **Request Changes**: any Critical or High finding.
- **Comment**: only Medium/Low/Info findings.
- **Approve**: no findings, or only Info findings with substantive praise warranted.

### Tone

- Ask questions for ambiguous situations: "What happens if `items` is empty here?" rather than "This will crash."
- Suggest, don't command: "Consider extracting this" rather than "Extract this."
- Do not hedge on genuine bugs. A null-deref is a null-deref. State it plainly.
- Do not invent praise. If nothing stands out positively, omit the "Done well" line.

---

## Edge cases

- **Draft PR**: review anyway, but lead the verdict with a note that the PR is draft.
- **Empty diff**: tell the user there's nothing to review.
- **Binary files in diff** (images, lockfiles): skip them, note skipped at the bottom.
- **Lockfile changes** (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`): do not review line-by-line. Note if new dependencies were added, and flag any that look unfamiliar or suspicious as a single Info item.
- **Generated files**: if a file header says "DO NOT EDIT" or the path matches `*.generated.*`, `dist/`, `build/`, skip it.
- **Very large PR** (>2000 lines): review in heavy mode, and add a Low-severity suggestion that the PR be split in future.

## What not to do

- Do not post to GitHub. Output stays in chat.
- Do not run `gh pr review`, `gh pr comment`, or any write operation against the PR.
- Do not modify files. This skill is read-only.
- Do not run tests, linters, or builds unless the user explicitly asks. The review is static.
- Do not invent issues to pad the table. A short, accurate review beats a long, noisy one.