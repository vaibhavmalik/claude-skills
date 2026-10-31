# Orchestrator (Full Cycle)

Delegates to sub-command files, batches all changes into a single push per iteration. Max 5 outer iterations.

**Assumes init is complete**: PR number, metadata, branch checkout are done by SKILL.md.

## Merged PR Flow

If the PR is merged, skip branch update and checks. Run only comments in follow-up mode:

1. Read `comments.md` and follow it **in full, including its "Commit, Push, and Reply" section** —
   the orchestrator-mode deferral does not apply here, because a merged PR has no Phase 2/3 below;
   `comments.md` pushes the reset branch, opens the follow-up PR and posts the replies itself
2. Print summary and exit

`--until-merged` is a no-op here (nothing left to merge) — warn and ignore if set.

## Open PR Flow

```
for iteration in 1..5:
```

### Phase 1: Gather and fix everything locally (NO push yet)

#### 1a. Update branch (conditional)

Read `update.md` and follow it, with one override: **do NOT push.** The orchestrator handles push in Phase 2.

If `--no-update` flag is set, skip. Note for summary: "Branch is N commits behind base."

Track whether Phase 1a made changes (merge commit or rebased commits).

#### 1b. Process comments

Read `comments.md` and follow it, with two overrides:
- **Do NOT push** — the orchestrator handles push in Phase 2.
- **Do NOT post replies yet** — the orchestrator handles replies in Phase 3.

Just collect comments, classify them, and apply fixes locally.

#### 1c. Read existing CI failure logs

Check if CI has results from a previous push:
```bash
gh pr checks PR_NUMBER --json name,state,bucket,link
```

- If any checks have `bucket` of `fail`: read the failure logs (get `link`, read logs), identify specific errors, fix them locally. **If Phase 1a made changes** (merge/rebase), verify each CI failure still exists in the current code before fixing — the merge may have resolved some issues.
- If all checks are `pending`: skip — nothing to read yet.
- If all checks passed: skip — nothing to fix.

#### 1d. Local validation

Run local static checks on files changed in steps 1a-1c to catch cascading issues. **Discover the commands from the project — never hardcode a stack:** derive them from the CI workflow definition (the failing job's steps name the tool), the project's build/test manifests and script-runner targets, or a documented command block. Map each failing CI check to its local command.

- **Fast static checks (type-check, lint, cheap analysis) — always re-run** the project's own versions on the changed files; these are cheap and catch cascades.
- **Tests — only if CI's test check failed**, and then only the specific failing tests/files named in the CI logs (a full suite is too slow here).

Fix any new issues found. Repeat local validation until clean.

### Phase 2: Push once

If ANY changes were made in Phase 1 (branch update, comment fixes, CI fixes, local validation fixes):

```bash
git add <specific files>
git commit -m "fix: <adapt message to what was actually fixed>

- <bullet for each fix>"
```

Adapt the commit message to what happened:
- Comments only: `fix: address review comments`
- CI only: `fix: resolve CI failures (<the specific failing checks>)`
- Both: `fix: address review comments and CI failures`

Push strategy:
- If `--rebase` was used: `git push --force-with-lease origin HEAD_REF_NAME`
- Otherwise: `git push origin HEAD_REF_NAME`

If no changes were made and branch update didn't create a merge commit, skip push entirely.

Note: The branch update (merge commit or rebased commits) and the fix commit are separate commits — the merge/rebase is already done in 1a, the fix commit is added on top. Both go in the single push.

### Phase 3: Post comment replies

After the push, in this order:

1. **Run the "Re-request review" step of `comments.md` first** (step 3 under "Commit, Push, and Reply") — decide the re-requests (anyone whose latest review state is
   `CHANGES_REQUESTED` and whose points this pass addressed) and fold its citing comment into the
   queued replies. It belongs after the push, because a re-request before the fix is pushed asks the
   reviewer to look at unchanged code; and it belongs BEFORE step 2, because its comment is *queued*
   like the others — running it afterwards would leave that comment with no posting step at all.
   Honour its idempotence guard: this orchestrator loops up to 5 times and a re-request does not
   clear the reviewer's standing state, so an unguarded re-add spams them once per iteration.
2. **Post all queued replies** (FIX, DEFER, and **REVIEW-SUMMARY**) using the commit SHA from
   Phase 2, following the reply formats in `comments.md`.
3. **Apply the `gh pr edit --add-reviewer` calls** that step selected, so the re-request lands with its
   answer already visible.

### Phase 4: Wait for CI and fix

Read `checks.md` and follow it. This polls CI, reads failure logs, fixes locally with validation, and pushes — with its own internal loop (max 3 iterations).

If checks reports that CI still fails after max iterations:
- If branch is behind base AND `--no-update` was used, hint: "Branch is N commits behind base — this may be causing CI failures. Re-run without `--no-update` to merge base branch."
- Stop the orchestrator and report to user.

### Phase 5: Check PR state and loop decision

Check if PR state changed:
```bash
gh pr view PR_NUMBER --json state -q .state
```
- If `MERGED`: switch to merged flow
- If `CLOSED`: exit with message

Loop decision:
- If Phase 1 produced no fixes AND Phase 4 passed first try: **converged** — re-true the body (5a), then go to Phase 6 if `--until-merged` is set, otherwise exit clean.
- Otherwise: loop back to Phase 1 (new comments may have appeared from the CI run)

### Phase 5a: Re-true the PR body (only on convergence)

Run only when the cycle converged, and only if **any iteration of this run** changed the PR — a merge of
base, a review fix, or a CI fix. WHY here and not earlier: those three are exactly the changes
that make a body describe a PR that no longer exists, and a claim that *was* accurate and has
stopped being true is worse than no claim, because the reviewer acts on it. Skipping it when
nothing changed keeps a no-op run from rewriting a body for no reason.

**Invoke the `pr-description` skill** (it ships alongside this one in the same marketplace) and
let it rewrite whatever the body now gets wrong; it owns the rules for what goes stale. If that
skill is not installed, say so in one line and move on — a stale body is a reported outcome, never
a blocked loop.

### Phase 6: Watch until merged (only if `--until-merged`)

Reached only after the cycle converges (CI green, comments handled) AND `--until-merged` is set. This is the persistent tail that enables auto-merge and keeps the branch current with base until the PR actually lands.

Read `until-merged.md` and follow it. It enables auto-merge (if not already on), then watches the PR's merge-readiness and re-syncs base whenever the branch falls behind — looping until the PR merges or it hits a stop-and-report blocker.

When it returns, include its watch summary in the final summary below and exit. Do NOT loop back to Phase 1.

## Summary

```
## babysit-pr Summary

PR: #123 — TITLE
Branch: feature/xyz
Mode: Live / Follow-up
Iterations: N

### Branch Update
Merged base branch (3 commits). Use `--rebase` for linear history.

### Comments
| # | File | Author | Comment | Action | Details |
|---|------|--------|---------|--------|---------|
| 1 | path/file.rb:42 | cursor[bot] | "Missing null check" | Fixed (abc1234) | Added nil guard |
| 2 | path/file.rb:78 | @reviewer | "Consider redesign" | Deferred | Architectural |

### CI Checks
All passed after N iteration(s).
Fixes applied:
- Fixed the lint violation CI reported in <file>
- Fixed the type-check error CI reported in <file>
```

**Branch Update section variants:**
- `Merged base branch (N commits). Use --rebase for linear history.`
- `Rebased onto base branch (N commits).`
- `Branch is N commits behind base. Use --rebase or default merge to update.` (when `--no-update`)
- `Already up to date.`

**Until-Merged section** (only when `--until-merged` ran): append the watch summary returned by `until-merged.md` (auto-merge state, re-sync count, outcome, time watched).

If max outer iterations hit, list remaining issues.
