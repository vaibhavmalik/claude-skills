# babysit-pr

A [Claude Code](https://claude.ai/code) skill that automates the PR review-fix-push cycle.

## What it does

`babysit-pr` watches your pull request, processes all review comments (from bots and humans), fixes what it can, defers what it can't, pushes changes, replies to threads, and loops until your PR is clean.

### Features

- **CI monitoring** — waits for all checks to pass/fail before processing comments
- **Automatic fixes** — applies bug fixes, style corrections, type fixes, and more
- **Smart deferrals** — defers architectural suggestions, subjective preferences, and risky changes
- **Threaded replies** — replies to each comment thread with `[babysit-pr]` prefix for traceability
- **Merged PR support** — creates follow-up PRs for already-merged PRs
- **Idempotent** — won't re-process comments it's already addressed
- **Safe** — never force pushes on open PRs, confirms before large changes, max 5 iterations

### How it classifies comments

| Action | Examples |
|--------|---------|
| **FIX** | Bug reports, security issues, style violations, missing types, unused imports, test improvements |
| **DEFER** | Architectural suggestions, subjective preferences, changes >50 lines, suggestions outside PR diff |

## Installation

### As a Claude Code plugin

```bash
# Add as a plugin marketplace
/plugin marketplace add vaibhavmalik/babysit-pr
/plugin install babysit-pr
```

### Manual (copy to your project)

Copy `skills/babysit-pr/SKILL.md` into your project's `.claude/skills/babysit-pr/SKILL.md`.

### Personal (all your projects)

Copy `skills/babysit-pr/SKILL.md` to `~/.claude/skills/babysit-pr/SKILL.md`.

## Usage

```
/babysit-pr           # Auto-detect PR from current branch
/babysit-pr 123       # Babysit PR #123
```

## Requirements

- [Claude Code](https://claude.ai/code)
- [GitHub CLI (`gh`)](https://cli.github.com/) — authenticated
- Git repository with a GitHub remote

## License

MIT
