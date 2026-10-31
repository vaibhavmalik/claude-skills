# Phase: until-merged (watch loop)

Keep the PR current with its base and let auto-merge land it. This is the persistent tail that runs **after** the normal full cycle has converged (branch green, comments handled), when the `--until-merged` flag is set.

**Assumes**: init is complete AND the normal orchestrator cycle (Phases 1–5) has already run at least once, so CI is green and comments are handled at entry.

**Scope**: full cycle only. This file is never reached for a sub-command, a merged PR (follow-up mode), or when `--until-merged` is absent.

## Why this exists

Branch protection commonly requires "branch must be up to date with base before merging." With auto-merge enabled, GitHub will land the PR the moment it is simultaneously up-to-date, green, and approved — but GitHub does **not** re-sync the branch for you when someone else merges to master first. The PR then sits at `BEHIND` forever. This loop closes that gap: it watches the PR's merge-readiness and re-syncs master whenever the branch falls behind, until the PR actually merges.

## Step 0: Provenance gate (inherited PRs only)

If this session did **not** itself build and verify the PR it is about to drive to merge (the PR
predates the session, or another agent/human authored it), give it **one adversarial read before
enabling auto-merge**: read the actual diff (`gh pr diff` — never endorse from the PR body + CI
badge), dispatch one fresh reviewer subagent over it (a second model if you have one), and for any deletion run the
non-text reference sweep — tracked symlinks pointing into deleted paths
(`git ls-files -s | awk '$1==120000 {print $4}'` + `readlink`), plus config/JSONB references a text
grep misses. WHY: an inherited cleanup PR was once auto-merged on "config-only, green CI" and broke
a tracked symlink on master; the fix failed and the whole PR was reverted — the one-line sweep would
have caught it at this exact moment. PRs this session built and took through its own verify gates
skip Step 0.

## Step 1: Ensure auto-merge is enabled

Check current state:
```bash
gh pr view PR_NUMBER --json autoMergeRequest -q .autoMergeRequest
```

If non-null, print "Auto-merge already enabled." and go to Step 2.

If `null`, pick a merge method the repo allows (don't hardcode squash):
```bash
gh repo view --json squashMergeAllowed,mergeCommitAllowed,rebaseMergeAllowed
```
Prefer `--squash`; else `--merge`; else `--rebase`. Enable:
```bash
gh pr merge PR_NUMBER --auto --squash   # substitute --merge / --rebase per allowed methods
```
Print "Auto-merge enabled (<method>)."

**Handle enable failures:**
- If it fails because the PR is **already mergeable right now** (GitHub merges directly instead of queuing auto-merge), treat as success — fall through to Step 2, which will observe `MERGED`.
- If it fails because **auto-merge is disabled for the repo**, STOP and report: "Auto-merge is not enabled for this repo — enable it in repo settings, or merge manually. Cannot keep a branch current toward an auto-merge that can never fire."

## Step 2: Watch loop

Poll merge-readiness and act on it. **Backstop: 60 minutes total.** Track elapsed wall-clock from loop entry; if it exceeds 60 min, stop and report the last observed status (the loop is safely re-runnable).

Each iteration:

```bash
gh pr view PR_NUMBER --json state,mergeStateStatus,mergeable,reviewDecision
```

Route on `state` first, then `mergeStateStatus`:

| Signal | Action |
|--------|--------|
| `state = MERGED` | **Exit success.** The PR landed. |
| `state = CLOSED` | **Exit.** "PR closed without merge while watching." |
| `mergeStateStatus = BEHIND` | **Re-sync (core action).** Read `update.md` and follow it (default merge strategy, or rebase if `--rebase`). This merges base into the branch and pushes. Then loop again immediately (no wait) — CI will re-run; the next iterations will pass through `UNSTABLE`/`CLEAN`. This is expected work, NOT a blocker. |
| `mergeStateStatus = DIRTY` or `mergeable = CONFLICTING` | Conflicts with base. Read `update.md` and follow it. If it resolves cleanly, push and loop. If `--safe` is set and a low-confidence conflict is hit, **stop and report** (see Blockers). |
| `mergeStateStatus = UNSTABLE` | A check is failing. Read `checks.md` and follow it (it waits for CI, fixes, pushes, max 3 iterations). If it returns still-failing on a **required** check, **stop and report**. If it returns green, loop again. |
| `mergeStateStatus = BLOCKED` | Inspect `reviewDecision`: if `REVIEW_REQUIRED` or `CHANGES_REQUESTED` → **stop and report** (human gate — see Blockers). Otherwise the block is a failing required check → read `checks.md`; if still failing after its loop, **stop and report**. |
| `mergeStateStatus = CLEAN` or `HAS_HOOKS` | Ready; auto-merge will land it. Wait 60s, then loop. |
| `mergeStateStatus = UNKNOWN` | GitHub is still computing mergeability. Wait 30s, then loop. If `UNKNOWN` persists for 5 consecutive checks, stop and report. |

After each `update.md` re-sync, re-evaluate from the top of the loop — the merge may have introduced check failures that surface as `UNSTABLE` on the next pass.

## Blockers (stop and report)

Per design, the loop continues for soft, expected events (branch fell behind, transient `UNKNOWN`, fixable checks) but **stops** on anything only a human can clear:

| Blocker | Report |
|---------|--------|
| `reviewDecision = REVIEW_REQUIRED` | "Waiting on required review approval. All other merge conditions are met — the PR will auto-merge once approved." |
| `reviewDecision = CHANGES_REQUESTED` | "A reviewer requested changes. Address their feedback (re-run `/babysit-pr comments PR`), then re-run with `--until-merged`." |
| Required CI check still failing after `checks.md` exhausts 3 iterations | "Required check `<name>` is still failing after fix attempts. See: <link>." |
| `--safe` low-confidence conflict during a re-sync | "A merge conflict needs human judgment (see conflict table). Resolve it, push, then re-run with `--until-merged`." |
| 60-minute backstop exceeded | "Watch backstop (60 min) reached. Last status: `<mergeStateStatus>`. Re-run with `--until-merged` to resume." |
| Auto-merge disabled for repo (Step 1) | "Auto-merge is not enabled for this repo." |

On any blocker: print the reason, print the watch summary below, and exit. Do not keep polling.

## After the merge: goal preservation on revert

A merged PR can still die: if you observe (now or on a later run) a `Revert "…(#N)"` PR referencing
a PR this skill merged, the original goal is silently undone — file a follow-up issue carrying the
original definition-of-done and what the revert revealed must be done differently (e.g. "remove
.cursor cleanly: migrate the symlinked skill content into .claude/ first, then delete"). WHY: after
one such revert, the task's goal stayed dead for weeks because no artifact carried it forward.

## Summary

```
### Until-Merged Watch
Auto-merge: enabled (squash) / already enabled
Re-syncs:   N merges of BASE_REF_NAME during watch
Outcome:    Merged ✅  /  Stopped — <blocker>  /  Backstop reached
Watched:    ~M minutes
```

If re-syncs hit conflicts, append the conflict table from `update.md`.
