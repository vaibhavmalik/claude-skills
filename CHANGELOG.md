# Changelog

## 2026-10-31

### Repository
- Renamed `babysit-pr` → `claude-skills` and restructured as a marketplace with one plugin per skill
  under `plugins/`. Old install commands keep working through GitHub's redirect.

### babysit-pr 2.0.0
- Split into sub-commands: `update`, `comments`, `checks`, and a full-cycle orchestrator.
- `--until-merged`: enable auto-merge and keep the branch current with base until the PR lands.
- `--rebase`, `--no-update`, `--safe` (pause on low-confidence conflict resolutions), `--help`.
- Review *summaries* are now collected alongside line and PR-level comments; reviewers whose latest
  state is `CHANGES_REQUESTED` are re-requested after a fix, with an idempotence guard.
- Deferred work is filed as a deduplicated issue; declined suggestions are answered without one.
- Pagination fix: comment fetches use `--paginate --slurp` so multi-page PRs are read in full.
- Local re-run of the project's static checks before every push; commands discovered from CI config.
- Re-trues the PR body via `pr-description` on convergence.
- Preview-deploy SHA verification before handing over a URL.

### pr-description 1.0.0
- First public release.
