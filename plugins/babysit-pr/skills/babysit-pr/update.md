# Sub-Command: update

Update the PR branch against its base branch. Default strategy is merge; use `--rebase` for rebase.

**Assumes init is complete**: PR number, metadata, branch checkout are done by SKILL.md.

**Orchestrator mode**: When called from the orchestrator (full cycle), push is deferred — the orchestrator batches all changes into a single push. Skip the push step and return control to the orchestrator.

## Execution

### 1. Fetch latest base branch

```bash
git fetch origin BASE_REF_NAME
```

### 2. Check if update is needed

```bash
git rev-list --count HEAD..origin/BASE_REF_NAME
```

If 0: print "Already up to date with BASE_REF_NAME." and exit.

### 3. Route by strategy

| Flag | Strategy |
|------|----------|
| (default) | Merge |
| `--rebase` | Rebase |

## Merge Strategy (default)

### 1. Merge

```bash
git merge origin/BASE_REF_NAME
```

### 2. Handle conflicts

If merge conflicts occur:

> **Merge vs rebase conflicts**: Unlike rebase (which stops at each conflicting commit), merge presents ALL conflicts at once in the working tree. Resolve everything in a single pass — no `--continue` loop.

#### Default mode (no --safe flag)

For each conflicted file:
1. Read the conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`)
2. Read the full file context and the PR diff to understand intent
3. Resolve the conflict by editing the file
4. `git add <file>`

After resolving ALL conflicted files, complete the merge:
```bash
git commit
```
(Git auto-populates the merge commit message.)

**Decision tracking**: For each conflict, record:
- File path
- What happened (e.g., "both sides modified imports", "overlapping changes")
- What you chose and why
- Confidence level: **high** (mechanical — schema regen, import ordering, adjacent additions) or **low** (overlapping logic, semantic changes)

#### Safe mode (--safe flag)

Same as default, except for **low confidence** conflicts:
- Present both sides to the user
- Show your suggested resolution
- Wait for user approval before proceeding
- If user rejects, ask for their preferred resolution

### 3. Push

```bash
git push origin HEAD_REF_NAME
```

No force-push needed — merge commits push normally.

## Rebase Strategy (--rebase)

### 1. Safety check

Before rebasing, check for unpushed commits from other authors:
```bash
git log --format='%an' origin/HEAD_REF_NAME..HEAD | sort -u
```
If any author other than the current git user appears, warn and stop:
"Branch has unpushed commits from other authors. Aborting rebase to avoid losing their work."

### 2. Rebase

```bash
git rebase origin/BASE_REF_NAME
```

### 3. Handle conflicts

If the rebase stops due to conflicts:

#### Default mode (no --safe flag)

For each conflicted file:
1. Read the conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`)
2. Read the full file context and the PR diff to understand intent
3. Resolve the conflict by editing the file
4. `git add <file>`

After resolving all files in the current step: `git rebase --continue`

Repeat until rebase completes. If rebase has multiple conflict steps, resolve each one.

**Decision tracking**: For each conflict, record:
- File path
- What happened (e.g., "both sides added imports", "overlapping method changes")
- What you chose and why
- Confidence level: **high** (mechanical — schema regen, import ordering, adjacent additions) or **low** (overlapping logic, semantic changes)

#### Safe mode (--safe flag)

Same as default, except for **low confidence** conflicts:
- Present both sides to the user
- Show your suggested resolution
- Wait for user approval before proceeding
- If user rejects, ask for their preferred resolution

### 4. Force-push

```bash
git push --force-with-lease origin HEAD_REF_NAME
```

If push fails (someone else pushed): stop and alert the user.

## Summary

**Merge:**
```
Branch updated via merge (N commits from BASE_REF_NAME). Use `--rebase` for linear history.
```

**Rebase:**
```
Branch rebased onto BASE_REF_NAME (N commits).
```

If conflicts were resolved, append:
```
N conflicts resolved automatically.
```

If any low-confidence resolutions were made:
```
## Conflicts Requiring Review
| File | What happened | Why it needs review |
|------|--------------|-------------------|
| path/to/file.rb | Merged overlapping method changes | Both sides modified `process_data` differently |
```
