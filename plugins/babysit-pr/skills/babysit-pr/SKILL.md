---
name: babysit-pr
description: |
  Composable PR babysitting with sub-commands. Bare invocation runs the full cycle
  (update branch → comments → CI checks → loop). Sub-commands run independently:
  `update` updates branch against base, `checks` fixes CI failures, `comments`
  processes review comments. Works on open and merged PRs.
argument-hint: "[update|checks|comments] [PR_NUMBER] [--rebase] [--no-update] [--safe] [--until-merged] [--help]"
---

# babysit-pr

Composable PR babysitting. Run the full cycle or individual sub-commands.

## Help

If `--help` is present in the arguments, print the following and exit immediately — skip all initialization:

```
babysit-pr — PR babysitting with CI fixes and comment handling

Usage: /babysit-pr [sub-command] [PR_NUMBER] [flags]

Sub-commands:
  (none)     Full cycle: update branch → comments → CI checks
  update     Update PR branch against base
  comments   Process review comments
  checks     Wait for CI, fix failures

Flags:
  --rebase         Update via rebase instead of merge (default: merge)
  --no-update      Skip branch update even if behind base (full cycle only)
  --safe           Pause on low-confidence conflict resolutions
  --until-merged   After the cycle converges, enable auto-merge and keep the
                   branch current with base until the PR merges (full cycle only)
  --help           Show this help

Examples:
  /babysit-pr 123                 Full cycle on PR #123 (merge if behind)
  /babysit-pr 123 --rebase        Full cycle, rebase instead of merge
  /babysit-pr 123 --no-update     Full cycle, skip branch update
  /babysit-pr 123 --until-merged  Full cycle, then watch + re-sync base until merged
  /babysit-pr update 123          Update branch via merge
  /babysit-pr update --rebase     Update branch via rebase
  /babysit-pr checks 123          Fix CI failures on PR #123
  /babysit-pr comments            Process comments on current branch's PR
```

## Argument Parsing

Parse the argument string into these parts:
- **Sub-command**: `update`, `checks`, `comments`, or none (full cycle)
- **PR number**: any numeric token
- **Flags**: `--rebase`, `--no-update`, `--safe`, `--until-merged`, `--help`

Arguments can appear in any order. Examples:
- `123` -> full cycle on PR #123 (merge if behind)
- `update 123` -> update branch on PR #123 (merge by default)
- `update 123 --rebase` -> update branch via rebase
- `123 checks` -> checks on PR #123
- `123 --rebase` -> full cycle, rebase if behind
- `123 --no-update` -> full cycle, skip branch update
- `123 --until-merged` -> full cycle, then watch + re-sync base until the PR merges
- `update --safe` -> update with manual conflict approval
- `--help` -> print help and exit
- (empty) -> full cycle, PR from current branch

**Flag validation:**
- `--no-update` only applies to the full cycle (no sub-command). If used with a sub-command, warn and ignore.
- `--rebase` applies to `update` sub-command and full cycle.
- `--safe` applies to any update (merge or rebase) — pauses on low-confidence conflict resolutions.
- `--rebase` and `--no-update` are mutually exclusive. If both specified, warn and use `--no-update`.
- `--until-merged` only applies to the full cycle (no sub-command). If used with a sub-command, warn and ignore.
- `--until-merged` and `--no-update` are contradictory (the watch loop's job is to re-sync base). If both specified, warn and ignore `--no-update`.
- `--until-merged` is a no-op on a merged PR (follow-up mode has nothing to merge). If the PR is merged, warn and ignore.

## Initialization

Run these steps for ALL sub-commands before routing.

### 1. Determine PR number

- If a numeric argument was provided, use it directly
- Otherwise, detect from current branch: `gh pr view --json number -q .number`
- If no PR found, abort with message

### 2. Get PR metadata

```bash
gh pr view PR_NUMBER --json number,title,headRefName,baseRefName,url,author,state
```

### 3. Check PR state and route

| State | Sub-command | Action |
|-------|-----------|--------|
| OPEN | update | Proceed |
| OPEN | checks | Proceed |
| OPEN | comments | Proceed |
| OPEN | (none) | Proceed with orchestrator |
| MERGED | update | Exit: "PR #N is merged, nothing to update." |
| MERGED | checks | Exit: "PR #N is merged, nothing to check." |
| MERGED | comments | Proceed in **follow-up mode** |
| MERGED | (none) | Proceed with orchestrator (follow-up mode — runs only comments) |
| CLOSED | any | Exit: "PR #N is closed without merge. Nothing to babysit." |

### 4. Handle dirty working directory

```bash
git stash --include-untracked
```
Pop the stash after checkout if it was created. Check `git stash list` to verify.

### 5. Ensure on correct branch

- **Open PR**: `git checkout HEAD_REF_NAME && git pull --rebase origin HEAD_REF_NAME`
- **Merged PR (follow-up mode)**:
  ```bash
  git checkout BASE_REF_NAME
  git pull origin BASE_REF_NAME
  git checkout HEAD_REF_NAME
  git reset --hard origin/BASE_REF_NAME
  ```

### 6. Announce

Print PR title, URL, branch, and which sub-command is running.

## Routing

After init, read the appropriate sub-command file and follow it:

| Sub-command | File to read |
|------------|-------------|
| `update` | Read `update.md` and follow it |
| `comments` | Read `comments.md` and follow it |
| `checks` | Read `checks.md` and follow it |
| (none) | Read `orchestrator.md` and follow it |

**IMPORTANT**: Only read ONE file — the one matching the sub-command. Do not read all files.

## Preview deploys (applies to every sub-command that pushes)

When the repo has a preview-deploy bot (a `@bot deploy` PR comment or equivalent) and the user will
inspect the change on a preview URL, drive the deploy yourself and **verify what actually deployed**:

1. After the push, post the repo's deploy command as a PR comment.
2. Wait for the bot's confirmation and read the SHA it reports; **assert it equals the branch
   HEAD you just pushed** — if it doesn't, the bot deployed a stale tree; re-trigger and re-check
   (two burn-in deploys silently ran on stale SHAs and the user tested the wrong code).
3. Only then hand over the URL — pointed at an entity that actually exercises the change (pick one
   with data for the new path; an empty fixture shows nothing and reads as a bug).

Never send the user to a preview URL you haven't SHA-verified.
