---
name: fix-github-issues
description: Find open GitHub Issues labeled "claude", design and implement a fix, test it, capture screenshots for UI changes, then open a PR. Optionally posts the proposed solution as a comment on the issue, and adds the "fixed" label to the issue once the PR is open.
---

Iterates the open issue queue tagged for Claude to act on. For each issue: understand → design → implement → test → screenshot (if UI) → open PR → label.

## Inputs
- Optional: a specific issue number. If omitted, pick the oldest open issue with the `claude` label.
- Repository defaults to the current working directory's `origin` remote.

## Process

### 1. Find candidate issues
```
gh issue list --label claude --state open --json number,title,labels,body,author,createdAt --limit 20
```
- If the user supplied an issue number, fetch just that one with `gh issue view <N> --json number,title,body,labels,comments,author`.
- Otherwise pick the oldest open issue and announce which one you're tackling.
- If the issue is already linked to an open PR (check with `gh issue view <N>`), stop and report — don't duplicate work.

### 2. Read the issue thoroughly
- Read the body AND all comments — clarifications and reproduction steps often live in the comment thread.
- Note any explicit requests in the issue body:
  - "Please describe your approach first" / "comment with your plan" → post the design as an issue comment before coding (step 4).
  - Reproduction steps, screenshots, expected vs actual behavior.
  - Acceptance criteria.

### 3. Design the solution
- Locate the relevant code. Use Grep/Glob/Read or spawn an `Explore` agent for anything broader than 3 queries.
- Form a complete plan: files to change, behavior changes, test strategy, any migrations or config implications.
- Identify the test surface — unit, integration, or manual UI verification.

### 4. (Conditional) Comment the design on the issue
If the issue requests a design proposal, a plan, an approach discussion, or asks Claude to "explain first" / "describe what you'll do":
```
gh issue comment <N> --body "$(cat <<'EOF'
## Proposed approach
<design summary>

## Files to change
- path/to/file.ts — <what changes>

## Test plan
- <how this will be verified>
EOF
)"
```
If the issue does NOT request this, skip — go straight to implementation.

### 5. Create a worktree and implement

Create an isolated worktree so the user's main checkout stays put:
```
git worktree add -b fix/issue-<N>-<short-slug> ../<repo>-<N> main
```
If the project has env files in a secrets dir (check CLAUDE.md), copy them into the worktree's `frontend/.env` / `backend/.env` and install deps (`pnpm install` / `uv sync` / `pnpm exec prisma generate` etc.) per CLAUDE.md's "first-time setup" section.

Follow the instructions from CLAUDE.md for running the app and testing it.
Add unit tests only for new backend code.

### 6. Test
- Run the project's CI suite (per CLAUDE.md if present).
- If a test fails, check whether it also fails on `main` with the same env. Env-sensitive pre-existing failures should be documented in the PR body, not blindly "fixed" on the side.

### 7. Capture screenshots (UI changes only)
Per CLAUDE.md's "Frontend PR Screenshots" section:
- Use `mcp__claude-in-chrome__gif_creator` to record before/after GIFs at the target viewport.
- Save them to a `screenshots/` folder at the repo root, commit them, and reference them from the PR body with **absolute raw GitHub URLs** (`https://github.com/<owner>/<repo>/raw/<branch>/screenshots/<name>.gif`). Relative paths don't render inside PR descriptions.
- If the feature requires specific app state (a trip with flights, a populated dashboard, an admin-only view), don't give up on the first attempt. Try multiple existing records, seed fresh state, or navigate to a different path before falling back to "couldn't reproduce."

### 8. Push and open the PR
```
git push -u origin fix/issue-<N>-<short-slug>
gh pr create --title "Fix #<N>: <short title>" --body "$(cat <<'EOF'
## Summary
<1-3 bullets>

Fixes #<N>

## Test plan
- [x] <test that was run>
- [ ] <manual verification>

## Before
![before](https://github.com/<owner>/<repo>/raw/fix/issue-<N>-<short-slug>/screenshots/<name>-before.gif)

## After
![after](https://github.com/<owner>/<repo>/raw/fix/issue-<N>-<short-slug>/screenshots/<name>-after.gif)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

### 9. Label the issue as "fixed"
Per CLAUDE.md: after the PR is open, add the `fixed` label to the **issue** (not the PR) so the triage state reflects that the fix is in review:
```
gh issue edit <N> --add-label fixed
```
If the `fixed` label doesn't exist on the repo, create it first:
```
gh label create fixed --color 99adc1 --description "Claude has written code and created a PR" || true
```

### 10. Report back
Tell the user:
- Issue number tackled and one-line summary
- PR URL
- Whether tests passed (and any pre-existing failures surfaced)
- Whether design was commented on the issue (if applicable)
- Confirmation that the `fixed` label is on the issue

## Safety / scope guards
- Don't merge the PR — only open it. Merging is the user's call.
- Don't close the issue manually — `Fixes #N` in the PR body handles that on merge.
- Don't push to `main` directly. Always work on a `fix/issue-<N>-*` branch.
- If the issue is ambiguous, post a clarifying question as an issue comment instead of guessing — and stop.
- If the test suite is broken on `main` before your changes, surface that to the user and stop rather than committing on top of red.
- Never use `--no-verify` to bypass hooks. Fix the underlying problem.

## Looping
This skill is intended to be run repeatedly (e.g. via `/loop`) to drain the backlog. Each invocation handles ONE issue and exits. If no open `claude`-labeled issues exist, report "queue empty" and stop.
