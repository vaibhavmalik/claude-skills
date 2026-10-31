# claude-skills

Claude Code skills for the unglamorous half of shipping: getting a pull request merged, and writing
the description a stranger can review it from. Each skill is a plugin in this marketplace, built from
rules that earned their place on real PRs.

| Plugin | What it does |
|---|---|
| [babysit-pr](plugins/babysit-pr) | Drives a PR to merge: updates the branch, answers every review comment (bot or human), fixes CI, re-requests reviewers, and optionally watches until auto-merge lands it. |
| [pr-description](plugins/pr-description) | Writes a PR body for an engineer who knows nothing about the task, then has a fresh subagent fact-check every claim against the diff before posting. |

The two compose: when a `babysit-pr` cycle converges after changing the PR, it calls `pr-description`
to re-true the body, so the description never describes a PR that no longer exists.

## Install

```text
/plugin marketplace add vaibhavmalik/claude-skills
/plugin install babysit-pr@claude-skills
/plugin install pr-description@claude-skills
```

Or copy a skill folder straight into a project (`.claude/skills/<name>/`) or your home directory
(`~/.claude/skills/<name>/`); each skill is self-contained.

## Use

```text
/babysit-pr                      # full cycle on the current branch's PR
/babysit-pr 123 --until-merged   # and keep it mergeable until it lands
/babysit-pr comments 123         # only answer review comments
/pr-description                  # write or rewrite the PR body
```

Run `/babysit-pr --help` for every sub-command and flag.

## Requirements

[Claude Code](https://claude.ai/code), an authenticated [GitHub CLI](https://cli.github.com/), and a
repository with a GitHub remote.

## Where these come from

Both skills were developed and used daily on a Rails backend and its surrounding services, then
stripped of anything specific to that setup. The READMEs in each plugin record what was measured and
why each rule exists. Contributions are welcome; please include the failure a change fixes.

## License

MIT
