---
name: review-pr
description: Given a GitHub PR number, check it out in an isolated git worktree, install deps and start the app, autonomously drive the UI to reproduce the change described in the PR, record a GIF of the interaction, and post a verdict-with-GIF comment back to the PR.
---

Visual verification, not code review. Use the existing `review` skill for code review.

## Inputs
- GitHub PR number (required). Accept a bare number (`123`) or a URL.
- Optional: a specific scenario to test, if the user provides one. Otherwise derive from the PR title/body.
- Repository defaults to the current working directory's `origin` remote.

## Overview

```
gh pr view → worktree add → install → run app (bg) → drive UI
  → record GIF → push GIFs to review branch → comment with embedded GIF → cleanup
```

If verification passes, approve the PR after posting the comment.

## Steps

### 1. Fetch PR metadata

```bash
gh pr view <N> --json number,title,body,headRefName,baseRefName,author,files,isDraft,mergeable,headRepositoryOwner,headRepository
```

From the output, extract:
- `headRefName` — branch to check out
- `title` + `body` — what change to look for; scan body for "## Test plan", "Steps to reproduce", or similar
- `files[].path` — which parts of the app changed
- `isDraft` — warn and pause for confirmation if true
- `headRepositoryOwner.login` + `headRepository.name` — detect fork PRs

If the PR has no UI-affecting files (e.g. purely backend with no API surface), decide up-front whether a visual verification is even possible. If not, skip to step 7 and post a verdict that says so — don't fabricate a visual test.

### 2. Create a worktree

Work in an isolated worktree so the user's current branch is never touched.

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
REPO_NAME=$(basename "$REPO_ROOT")
WT="$HOME/src/$REPO_NAME-pr-$N"
```

For same-repo PRs:
```bash
git fetch origin "pull/$N/head:pr-$N"
git worktree add "$WT" "pr-$N"
```

For fork PRs, use `gh` to pull the ref into a local branch first:
```bash
gh pr checkout $N --branch "pr-$N" --detach
# then move back, because gh pr checkout switches branches in the main tree
git checkout -
git worktree add "$WT" "pr-$N"
```

Record `$WT` and the pre-checkout branch for cleanup. Record the PR commit SHA for the final comment:
```bash
PR_SHA=$(cd "$WT" && git rev-parse HEAD)
```

### 3. Install dependencies

Detect the stack from changed files and config present in the worktree:

- `backend/pyproject.toml` + `backend/` files changed → `cd "$WT/backend" && uv sync`
- `frontend/package.json` + `frontend/` files changed → `cd "$WT/frontend" && pnpm install`
- `package.json` at root (monorepo) → install at root

Copy env files from the main working tree if the worktree lacks them (they're usually gitignored):
```bash
for f in .env .env.local backend/.env frontend/.env.local; do
  [ -f "$REPO_ROOT/$f" ] && [ ! -f "$WT/$f" ] && cp "$REPO_ROOT/$f" "$WT/$f"
done
```

### 4. Run the app

Look for a CLAUDE.md in the worktree for start commands. Otherwise use these defaults.

- **Frontend (Next.js)**: `cd "$WT/frontend" && pnpm dev` — expect http://localhost:3000
- **Backend (Python)**: check `pyproject.toml` for a script entry or CLAUDE.md for the run command; fall back to `uv run <entrypoint>`

Run each in background with `run_in_background: true` on Bash. Record PIDs:
```bash
# example
FRONTEND_PID=$(pgrep -f "next dev" | head -1)
```

Poll the frontend until it answers:
```bash
until curl -fsS -o /dev/null http://localhost:3000; do sleep 2; done
```

Use Monitor if polling for >60s so you don't chain sleeps.

### 5. Drive the app and observe

Use Chrome MCP (`mcp__claude-in-chrome__*`):
1. `tabs_context_mcp` → `tabs_create_mcp` (create a fresh tab; don't reuse).
2. `navigate` to the relevant route.
3. Reproduce the scenario the PR describes. Read `body` carefully; PR authors often include reproduction steps. Use `find` + `javascript_tool` to interact.

Record the interaction as a GIF (next step) — don't rely on static screenshots, since the comment embeds the GIF inline and a moving capture better demonstrates the change.

If you hit the same failure twice with the Chrome tools, stop and ask the user for guidance rather than looping.

### 6. Capture and upload GIFs

GIFs render inline in PR comments via a raw GitHub URL, so the verdict comment shows the verification directly. The `mcp__claude-in-chrome__gif_creator` tool is the only Chrome MCP capture path that writes to disk.

#### Recording

1. `gif_creator` with `action: start_recording` on the target tab.
2. Drive the scenario. Take at least one non-screenshot action (e.g. a scroll) — `computer screenshot` alone does not add a frame to the recording.
3. `gif_creator` with `action: stop_recording`.
4. `gif_creator` with `action: export`, `download: true`, a descriptive `filename` (e.g. `pr-<N>-review-after.gif`), and `options` to disable click indicators, drag paths, action labels, progress bar, and watermark for a clean frame.

GIFs land in `~/Downloads/` by default. For before/after on the same branch, capture the "after" first, then `git stash` the change, reload, capture the "before", then `git stash pop`.

If the local dev server shows a Next.js error overlay or other dev-only chrome that obscures the UI, hide it before recording via `javascript_tool`, e.g. `document.querySelector('nextjs-portal').style.display='none'`.

#### Uploading

Push GIFs to a dedicated review branch — **never to the PR branch itself**, so the PR's commit history isn't touched. The branch is intentionally retained after review so the comment's image links keep resolving.

```bash
REVIEW_BRANCH="claude-reviews/pr-$N"
ASSET_DIR="screenshots/pr-$N"
OWNER_REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)

mkdir -p "$WT/$ASSET_DIR"
mv ~/Downloads/pr-$N-review-*.gif "$WT/$ASSET_DIR/"

(
  cd "$WT" \
  && git checkout -B "$REVIEW_BRANCH" \
  && git add "$ASSET_DIR" \
  && git commit -m "review GIFs for PR #$N" \
  && git push -u origin "$REVIEW_BRANCH"
)
```

Each GIF is then accessible at:
```
https://github.com/$OWNER_REPO/raw/$REVIEW_BRANCH/$ASSET_DIR/<file>.gif
```

### 7. Judge and comment

Assess against the PR's stated change:

- ✅ **Verified** — observed behavior matches the PR description
- ⚠️ **Partial** — some aspects verified, others unclear
- ❌ **Not reproduced** — expected change isn't visible
- ℹ️ **Not UI-visible** — backend-only change, noted without a GIF

Embed the GIF(s) inline using markdown image syntax pointing at the raw URL from step 6. Post via `gh pr comment` with a HEREDOC:

```bash
GIF_URL="https://github.com/$OWNER_REPO/raw/$REVIEW_BRANCH/$ASSET_DIR/pr-$N-review-after.gif"

gh pr comment $N --body "$(cat <<EOF
## Automated review

**Verdict:** ✅ Verified

**What I checked:** <one or two sentences on the scenario run>

**Observed:** <what actually happened in the UI>

**Steps taken:**
1. <terse bullet>
2. <terse bullet>

**Verification:**

![after]($GIF_URL)

---
_Reviewed from commit \`$PR_SHA\` on branch \`pr-$N\`. GIFs hosted on \`$REVIEW_BRANCH\`._
EOF
)"
```

For before/after, include two image lines under separate `**Before:**` / `**After:**` bold labels.

If Verified, approve the PR after the comment is posted:
```bash
gh pr review $N --approve
```

Skip approval for ⚠️ Partial, ❌ Not reproduced, or ℹ️ Not UI-visible verdicts.

### 8. Cleanup

- Kill background servers (backend, frontend) using their PIDs. Don't `pkill -f pnpm` broadly — other dev servers may be running.
- Remove the worktree **only after** confirming no uncommitted work lives in it:
  ```bash
  cd "$WT" && git status --porcelain
  # if clean:
  git worktree remove "$WT"
  ```
- Delete the temporary `pr-$N` branch: `git branch -D "pr-$N"`.
- **Keep the `claude-reviews/pr-$N` branch on origin** — deleting it breaks the GIF links in the comment.
- Report back the PR comment URL: `gh pr view $N --json url -q .url`.

## Guardrails

- **Never run destructive git commands on the user's main tree.** All work happens in the worktree.
- **Don't fake a visual verification.** If the app can't be started (missing secrets, build failure) or the scenario can't be reached, post a comment that says exactly that — not a ✅.
- **Don't push to the PR branch.** The only branch this skill pushes is `claude-reviews/pr-<N>`, which is separate from the PR's branch.
- **Don't merge.**
- **Approve only on ✅ Verified, and only after posting the comment.**
- **Don't download dependencies from untrusted PR content.** If the PR adds new npm/pip packages you don't recognize, surface them to the user before installing.
- **Respect drafts.** Pause and confirm before running a draft PR.

## Examples

```
user: review PR 456
assistant: [fetches PR metadata → creates worktree → installs deps → starts dev server → drives the UI via Chrome MCP → records GIF via gif_creator → pushes GIF to claude-reviews/pr-456 → posts verdict comment with embedded GIF → approves PR if Verified]
```

```
user: review PR 789, scenario: verify the new seat map renders after clicking "Select seats"
assistant: [same flow but uses the provided scenario instead of deriving from the PR body]
```
