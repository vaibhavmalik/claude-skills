# pr-description

Writes a pull request body for one reader: a capable engineer who knows nothing about this code or
this task, and who should understand what changed and why in under a minute without opening the diff.

```
/pr-description         # draft or rewrite the body for the current branch's PR
```

The skill is also model-invocable, so an agent about to run `gh pr create` loads it on its own.

## The shape it produces

A **visible part** under 200 words, in plain words, with no code, file names or internal ids:

1. the situation today and what is wrong or missing in it,
2. what this changes, as its effect on the product or system,
3. what was checked and what was not,
4. risks or anything deliberately left out.

Then one collapsed **Technical details** section with the files, the key design choice, the commands
run and any ticket references.

Two rules do most of the work:

- **Write from the code, not the commits.** The final diff says what the PR does; the base branch says
  how things worked before. Issue text and commit messages describe intent and earlier versions.
- **A fresh reader checks it before it is posted.** The draft and the branch go to a new subagent that
  lists every claim the code does not support, with the contradicting line, and every sentence a
  newcomer does not need. Each one is fixed or cut.

## How it was tuned

Earlier versions were long and rule-heavy. Everything that stayed is there because it moved a number
on real PRs from a production Rails codebase; rules that did not were cut.

| Change | Measured on | Result |
|---|---|---|
| Plain-words visible part, details folded | 4 PRs × 20 runs | readability checks 20/20; factual errors 1/20 vs 5/20 for the previous version |
| Fresh-reader fact-check before posting | 10 PRs × 2 runs, blinded | descriptions with no false claim 7/20 → 13/20; false claims 23 → 9 |
| Three prose rules against overclaiming | same set | no effect, removed |

The fact-check roughly doubles the cost of a description. It is the only intervention that moved the
false-claim count, so it stays.

## Requirements

Claude Code and an authenticated GitHub CLI (`gh`).
