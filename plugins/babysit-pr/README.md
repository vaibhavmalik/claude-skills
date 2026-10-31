# babysit-pr

Drives a pull request to merge so you don't have to keep a tab open.

```
/babysit-pr                     # full cycle on the current branch's PR
/babysit-pr 123 --until-merged  # full cycle, then keep it mergeable until it lands
/babysit-pr comments 123        # only answer review comments
/babysit-pr checks 123          # only fix CI
/babysit-pr update 123 --rebase # only bring the branch up to date
```

## What a full cycle does

1. **Update** the branch against base (merge by default, `--rebase` for linear history). Conflicts are
   resolved with a recorded confidence level; `--safe` pauses on the low-confidence ones.
2. **Collect every piece of review feedback** in three batched API calls: line comments, PR-level
   comments and *review summaries* — the last one is what most tools miss, and it is where a reviewer
   who requests changes without line comments lives.
3. **Classify each item FIX or DEFER.** Bugs, security, style, types, unused code, clear test asks get
   fixed. Architecture, taste, false positives, anything over ~50 lines or outside the diff gets a
   reasoned reply, and real deferred work gets an issue filed (deduplicated across loops).
4. **Read existing CI failures and fix them locally**, then re-run the project's own static checks on
   the changed files before anything is pushed. Commands are discovered from the CI workflow and the
   project's manifests, never hard-coded.
5. **Push once**, then post every reply in-thread with a `[babysit-pr]` prefix and the commit SHA.
6. **Re-request review** from anyone whose latest state is `CHANGES_REQUESTED` and whose points were
   addressed. A pushed fix does not clear that state on GitHub; only a re-request does.
7. **Wait for CI**, fix what fails (max 3 rounds, one push per round), and loop back to step 1 while
   new feedback keeps arriving (max 5 outer iterations).
8. On convergence, **re-true the PR body** with the `pr-description` skill if it is installed.
9. With `--until-merged`: enable auto-merge, then watch `mergeStateStatus` and re-sync base every time
   the branch falls `BEHIND`, until the PR merges or something only a human can clear (required review,
   a failing required check, a repo with auto-merge disabled).

Merged PRs are handled too: the branch is reset from base and the fixes go out as a follow-up PR.

## Design choices worth knowing

- **Idempotent.** Threads already answered with `[babysit-pr]` are skipped, re-requests are guarded
  against the reviewer list, deferral issues are searched before they are filed.
- **One push per iteration.** Fixes from branch update, comments and CI are batched so CI runs once.
- **Never `git add -A`.** Specific files only; never force-pushes an open PR unless `--rebase` asked.
- **Stops on human gates.** Required approvals, requested changes it deferred, auto-merge disabled,
  a 60-minute watch backstop: each prints a reason and a summary, never spins.
- **Composable.** `SKILL.md` parses arguments and routes; `update.md`, `comments.md`, `checks.md`,
  `orchestrator.md` and `until-merged.md` each own one job and are read only when needed.

Several rules carry a "measured" note. Each one came from a real failure in daily use, for example a
multi-page comment fetch that silently dropped page two, or a reviewer's change request that sat
three days because it had no line comments.

## Requirements

Claude Code, an authenticated GitHub CLI (`gh`), and a repository with a GitHub remote.
