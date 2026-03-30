---
name: babysit-pr
description: |
  Automates the PR review-fix-push cycle: waits for CI checks, handles ALL review comments
  (bots and humans), fixes or defers them, pushes changes, replies to threads, and repeats
  until clean. Works on both open and merged PRs — for merged PRs, resets the same branch
  from base and creates a new follow-up PR. Use when a PR has pending reviews or the user
  wants hands-free PR babysitting.
argument-hint: "[PR_NUMBER]"
---

# babysit-pr

Automate the PR review-fix-push cycle. Waits for CI, processes ALL review comments (bot and human), fixes code or defers, pushes, replies, and loops until clean.

## Initialization

1. **Determine PR number**:
   - If argument provided, use it directly
   - Otherwise, detect from current branch: `gh pr view --json number -q .number`
   - If no PR found, abort with message

2. **Get PR metadata**:
   ```bash
   gh pr view PR_NUMBER --json number,title,headRefName,baseRefName,url,author,state
   ```

3. **Check PR state and set mode**:
   - If `OPEN`: proceed normally (**live mode**)
   - If `MERGED`: switch to **follow-up mode** (see "Follow-Up Mode for Merged PRs" below)
   - If `CLOSED` (not merged): exit with message: "PR #NUMBER is closed without merge. Nothing to babysit."

4. **Handle dirty working directory**: Before switching branches, stash any uncommitted changes:
   ```bash
   git stash --include-untracked
   ```
   Pop the stash after checkout if it was created. Check `git stash list` to verify.

5. **Ensure on correct branch**:
   - **Live mode**: `git checkout HEAD_REF_NAME && git pull --rebase origin HEAD_REF_NAME`
   - **Follow-up mode**: See setup below

6. **Announce**: Print PR title, URL, branch, and mode (live / follow-up). Begin loop.

## Follow-Up Mode for Merged PRs

When the PR is already merged, reuse the **same branch name** — reset it to the base branch and apply fixes there.

### Setup
```bash
# Checkout base branch and update
git checkout BASE_REF_NAME
git pull origin BASE_REF_NAME

# Reset the original PR branch to base (safe — PR is already merged)
git checkout HEAD_REF_NAME       # e.g. fix/review-feedback
git reset --hard origin/BASE_REF_NAME
```

This gives a clean branch matching the current base, reusing the original branch name for the follow-up PR.

### Comment Processing
- Process comments the same as live mode (Phase B -> C)
- All fixes go on the reset `HEAD_REF_NAME` branch

### Reply Format (Follow-Up Mode)

**For FIX**:
```
[babysit-pr] Will be addressed in follow-up PR #{FOLLOW_UP_PR_NUMBER}.

<Brief explanation of what will be changed and why>
```

**For DEFER**:
```
[babysit-pr] Deferred — <category>.

<Explanation>
```

### After Processing All Comments
If any fixes were made:
1. Commit and push the branch (force push is OK here — the PR was already merged, branch is being reused):
   ```bash
   git push --force-with-lease origin HEAD_REF_NAME
   ```
2. Create a follow-up PR from the same branch:
   ```bash
   gh pr create --title "fix: address review comments from #NUMBER" \
     --body "Follow-up to #NUMBER. Addresses review comments:

   - <bullet for each fix>

   Original PR: #NUMBER"
   ```
3. Post replies on the **original PR** with the follow-up PR number
4. Print the follow-up PR URL in the summary

## Core Loop (max 5 iterations)

Track iteration count. Exit if iteration > 5 with summary of remaining issues.

**At the start of each iteration** (live mode only), check PR state:
```bash
gh pr view PR_NUMBER --json state -q .state
```
If state changed to `MERGED` during the loop: switch to follow-up mode for any remaining comments. If `CLOSED` (not merged), exit gracefully.

### Phase A: Wait for CI Checks

**Live mode only.** Skip this phase in follow-up mode.

> **CRITICAL**: You MUST NOT proceed to Phase B until ALL CI checks have either passed or failed. Never say "while waiting, I'll collect comments" — Phase B does not start until Phase A is fully complete. Bots (e.g. automated review bots) post new review comments AFTER CI runs complete, so collecting comments before CI finishes will miss them.

**Always use polling** — do NOT use `--watch` as it can hang or be interrupted:

```bash
# Poll every 30s until all checks reach a terminal state (pass/fail/skipped)
gh pr checks PR_NUMBER --json name,state,bucket
```

Poll loop:
1. Fetch checks with the command above
2. If any check has `state` of `IN_PROGRESS` or `QUEUED` (bucket `pending`): wait 30 seconds, repeat
3. If all checks have `state` of `SUCCESS`, `SKIPPED`, or terminal: proceed to Phase B
4. If any check has `state` of `FAILURE` or `ERROR`: read failing logs, fix, commit, push, restart Phase A
5. Maximum wait: 15 minutes. If checks still pending after 15 min, stop and report to user — do NOT proceed.

Note: `conclusion` is NOT a valid field — only `name`, `state`, `link`, `bucket`, `completedAt`, `description`, `event`, `startedAt`, `workflow` are available.

### Phase B: Collect Unresolved Review Comments

Process **ALL review comments** — from bots and humans alike. Do NOT filter by author type.

Fetch all comments in **two batch API calls**, then filter and cross-reference in memory. Do NOT make per-comment API calls for reply checks.

#### Step 1: Batch-fetch all review comments

```bash
gh api repos/{owner}/{repo}/pulls/PR_NUMBER/comments --paginate --jq '[.[] | {id, body, path, line, user_login: .user.login, user_type: .user.type, in_reply_to_id: .in_reply_to_id, created_at}]'
```

**Important**:
- Use `--jq` to extract only needed fields and produce a single valid JSON array. This avoids the concatenated-JSON problem that `--paginate` causes without `--jq`.
- Do NOT attempt to strip HTML comment markers (`<!-- -->`) inside `--jq` — `gh` escapes `!` as `\!` in jq strings, causing parse errors. Instead, fetch the raw `body` and strip bot noise (automated fix links, bot IDs, `<!-- DESCRIPTION START/END -->` blocks) in your own code after fetching.

#### Step 2: Batch-fetch all issue comments

```bash
gh api repos/{owner}/{repo}/issues/PR_NUMBER/comments --paginate --jq '[.[] | {id, body, user_login: .user.login, user_type: .user.type, created_at}]'
```

#### Step 3: Identify actionable comments

Keep all comments that contain review feedback — from **any author** (bot or human). Skip:
- Comments that are purely informational (CI status reports, merge notifications)
- Comments from the PR author themselves (self-comments are usually notes, not review requests)
- Comments that are clearly just acknowledgments ("LGTM", "Looks good", thumbs up)

#### Step 4: Filter out already-resolved comments

Using the batch-fetched data (no additional API calls):

- **For review comments**: Build a map of `in_reply_to_id -> [reply comments]`. A comment is resolved if any reply in its thread contains `[babysit-pr]` in the body.
- **For issue comments**: Scan all issue comments after the target comment. If any subsequent comment contains `[babysit-pr]` and references the comment (by quoting or by proximity), consider it resolved.

#### Step 5: Exit check

If no unresolved comments remain, exit loop with success summary.

### Phase C: Process Each Comment

For each unresolved comment:

#### C1: Gather Context

- Read the file referenced in the comment (`path` field for review comments)
- Read the PR diff for that file: `gh pr diff PR_NUMBER -- FILE_PATH`
- Parse the comment body to understand what's being requested

#### C2: Classify — FIX or DEFER

**FIX** (actionable, clear, safe):
- Bug reports (null checks, off-by-one, missing error handling)
- Security issues (injection, XSS, secrets exposure)
- Style/convention violations (naming, formatting, imports)
- Straightforward performance issues (N+1, unnecessary allocations)
- Missing types or type errors
- Unused imports/variables
- Test improvements with clear direction
- Transaction safety (wrapping related operations in a transaction)

**DEFER** (subjective, risky, or out of scope):
- Architectural suggestions ("consider redesigning this module")
- Subjective preferences without project convention backing
- False positives (reviewer misunderstood the code)
- Changes requiring >50 lines of modification across multiple files
- Suggestions that contradict project conventions (check CLAUDE.md)
- Performance suggestions requiring benchmarking to validate
- Suggestions about code not in the PR diff

#### C3: Apply Fix or Draft Deferral

**If FIX**:
- Make the code change using Edit tool
- Keep changes minimal and focused on what the comment requests
- NEVER modify files outside the PR diff (live mode) or outside the files referenced by comments (follow-up mode)
- If the fix would touch >50 lines, STOP and ask the user for confirmation before proceeding
- Stage and note the fix for the commit in Phase D

**If DEFER**:
- Do not modify any code
- Prepare a deferral reply explaining why

#### C4: Queue Reply for Comment Thread

Queue replies — do NOT post them yet. They are posted in Phase D after the commit (so FIX replies can include the commit SHA).

**For review comments** (code-line comments), reply in-thread using `in_reply_to`:
```bash
gh api repos/{owner}/{repo}/pulls/PR_NUMBER/comments \
  -X POST -f body="REPLY_BODY" -F in_reply_to=COMMENT_ID
```
**Note**: Do NOT use the `/pulls/comments/COMMENT_ID/replies` endpoint — it returns 404. Use `/pulls/PR_NUMBER/comments` with the `in_reply_to` field instead.

**For issue comments** (PR-level), reply on the PR:
```bash
gh pr comment PR_NUMBER --body "REPLY_BODY"
```

### Phase D: Commit, Push, Reply, and Loop

1. **If any fixes were made**:
   ```bash
   git add <specific files that were changed>
   git commit -m "fix: address review comments

   - <bullet point for each fix made>"
   ```
   - NEVER use `git add -A` or `git add .`
   - **Live mode**: Push normally: `git push origin BRANCH_NAME`
   - **Follow-up mode**: Force push is OK (branch was reset from merged PR): `git push --force-with-lease origin BRANCH_NAME`

2. **Post queued FIX replies** with the actual commit SHA from the push

   **Live mode reply format**:
   ```
   [babysit-pr] Fixed in COMMIT_SHA.

   <Brief explanation of what was changed and why>
   ```

   **Follow-up mode reply format**:
   ```
   [babysit-pr] Will be addressed in follow-up PR #{FOLLOW_UP_PR_NUMBER}.

   <Brief explanation of what will be changed and why>
   ```
   Note: In follow-up mode, create the PR first (Phase D step 5), then post replies with the PR number.

3. **Post DEFER replies** (these don't need a commit SHA):
   ```
   [babysit-pr] Deferred — <category>.

   <Explanation>
   ```

4. **If fixes were made in live mode**: Increment iteration counter, go back to Phase A — **wait for the new CI run to complete before collecting comments again**. New bot comments (e.g. automated review bots) appear only after CI finishes, so skipping the wait will miss them.

5. **If fixes were made in follow-up mode**: Create the follow-up PR (if not already created), then exit:
   ```bash
   gh pr create --title "fix: address review comments from #NUMBER" \
     --body "$(cat <<'EOF'
   ## Summary
   Follow-up to #NUMBER. Addresses review comments.

   - <bullet for each fix>

   Original PR: #NUMBER
   EOF
   )"
   ```

6. **If only deferrals (no code changes)**: Exit loop

### Phase E: Exit Summary

Print a summary table:

```
## babysit-pr Summary

PR: #NUMBER — TITLE
Mode: Live / Follow-up
Follow-up PR: #FOLLOW_UP_NUMBER (if applicable)
Iterations: N
Status: Clean / Has remaining issues

| # | File | Author | Comment | Action | Details |
|---|------|--------|---------|--------|---------|
| 1 | path/to/file.rb:42 | reviewer-bot | "Missing null check" | Fixed (abc1234) | Added nil guard |
| 2 | path/to/file.rb:78 | @reviewer | "Consider redesign" | Deferred | Architectural, out of scope |
```

If there are remaining issues (hit max iterations), list them clearly.

## Safety Guardrails

1. **Max 5 iterations** — prevents infinite loops
2. **Never force push in live mode** — only force push in follow-up mode (branch reset from merged PR)
3. **Never modify unrelated files** — in live mode, check file is in `gh pr diff --name-only`; in follow-up mode, only modify files referenced by comments
4. **Confirm before large changes** — if a single fix requires >50 lines changed, ask user
5. **Stage specific files** — never `git add -A` or `git add .`
6. **Idempotent replies** — `[babysit-pr]` prefix prevents re-processing
7. **Replies posted on original PR** — even in follow-up mode, replies go on the original PR thread
8. **Stash before branch switch** — always stash uncommitted changes before checkout

## Edge Cases

- **PR merges mid-loop**: If PR state changes to `MERGED` during live mode, switch to follow-up mode for remaining comments
- **Dirty working directory**: Stash uncommitted changes before switching branches, pop after
- **Deleted remote branch**: If the original branch was deleted after merge, `git checkout` will use the local ref or create fresh from base — `git push --force-with-lease` will recreate it on remote
- **Deleted files in PR**: Skip comments on files that were deleted in the PR
- **Outdated diff positions**: If the comment references a line that no longer exists after a push, note it in the reply as resolved by recent changes
- **Push failures**: If push fails in live mode, do NOT force push — alert the user and stop. In follow-up mode, `--force-with-lease` failures mean someone else pushed to the branch — alert and stop.
- **Rate limiting**: If GitHub API rate-limits, wait and retry once. If still limited, stop and report
- **No review comments**: If Phase B finds zero comments, skip to summary and report "No review comments found"
- **Binary files**: Skip comments on binary files
- **Self-comments**: Skip comments from the PR author (unless they explicitly request a change)
