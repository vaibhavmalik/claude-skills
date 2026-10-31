# Sub-Command: checks

Wait for CI checks, fix ALL failures locally, validate locally, push once, and loop. Max 3 iterations.

**Assumes init is complete**: PR number, metadata, branch checkout are done by SKILL.md.

**Orchestrator mode**: In the full cycle, the orchestrator reads existing CI failures in its Phase 1c (before pushing). This file is then used for Phase 4 — polling for fresh CI results after the orchestrator's single push, and fixing any remaining failures with its own push-and-retry loop (max 3 iterations).

## Execution (max 3 iterations)

### 1. Poll for CI completion

> **CRITICAL**: Always use polling — do NOT use `--watch` as it can hang or be interrupted.

```bash
gh pr checks PR_NUMBER --json name,state,bucket
```

Poll loop:
1. Fetch checks with the command above
2. If any check has `bucket` of `pending`: wait 30 seconds, repeat
3. If all checks have terminal state (`pass`, `fail`, `skipped`): proceed to step 2
4. Maximum wait: 15 minutes. If checks still pending after 15 min, stop and report to user.

Note: `conclusion` is NOT a valid field — only `name`, `state`, `link`, `bucket`, `completedAt`, `description`, `event`, `startedAt`, `workflow` are available.

### 2. Check results

If all checks passed or skipped: before exiting with the success summary, check whether the project
names a pre-merge verification step of its own (a `verify`/`ship` script in its manifests, or a step in
CLAUDE.md / CONTRIBUTING). If it does, run it — any fix commit pushed during this loop moved HEAD past
whatever review last looked at it, and CI-green fix commits nobody re-read are where regressions
cluster. If the project defines nothing, exit with the success summary.

If any check failed: proceed to step 3.

### 3. Read ALL failure logs

For **every** failing check (not just the first one):
- Get the check's `link` from the JSON output
- Read the failure logs to understand what failed
- Identify the root cause

Collect ALL failures before making any fixes.

### 4. Fix ALL failures

Fix every issue found in step 3, across **all** failing checks — do NOT fix one and push. The failing checks span whatever categories *this project* runs in CI (type-checking, linting, tests, build, etc.); fix them all.

Keep changes minimal and focused on fixing the failures.

### 5. Local validation (CRITICAL — do this BEFORE pushing)

After fixing all CI-reported failures, reproduce the relevant checks **locally** to catch cascading issues before pushing. **Discover the commands from the project — never hardcode a stack.** Derive them, in order of preference, from: the CI workflow definition (the failing job's own steps name the exact tool + invocation), the project's build/test manifests and script-runner targets (e.g. declared `lint`/`typecheck`/`test` scripts), or a documented command block in the README/CONTRIBUTING. Map each failing CI check to its local command.

- **Fast static checks (type-check, lint, and any cheap analysis the project runs) — always re-run** the project's own versions, scoped to the changed files where the tool supports it. These are cheap and catch cascades. Scope per the project's module/package layout when its tooling is per-package.
- **Test suite — only if CI's test check failed.** A full suite is too slow for the inner loop: run only the **specific failing tests/files** named in the CI logs to verify the fix before pushing.

Fix any new issues found. **Repeat local validation until all static checks are clean.** Do not push until they pass.

### 6. Commit and push (ONCE per iteration)

```bash
git add <specific files>
git commit -m "fix: <description of CI fixes>

- <bullet for each fix>"
git push origin HEAD_REF_NAME
```

Stage specific files — never `git add -A` or `git add .`.

**ONE push per iteration.** The whole point is to batch all fixes into a single push so CI only runs once.

### 7. Loop

Increment iteration counter. Go back to step 1 — wait for the new CI run.

## Summary

```
CI Checks: All passed after N iteration(s).
Fixes applied:
- Iteration 1: Fixed the lint, type-check, and test failures CI reported (name the specific checks/rules fixed)
```

Or if max iterations hit:
```
CI Checks: Still failing after 3 iterations.
Remaining failures:
- check-name: <description>
```
