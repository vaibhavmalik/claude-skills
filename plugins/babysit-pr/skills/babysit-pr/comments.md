# Sub-Command: comments

Process unresolved review comments. Single pass — no loop.

**Assumes init is complete**: PR number, metadata, branch checkout are done by SKILL.md.

**Orchestrator mode**: When called from the orchestrator (full cycle), push and reply posting are deferred — the orchestrator batches all changes into a single push, then posts replies afterward. Skip the "Commit, Push, and Reply" section and return control to the orchestrator with the queued fixes and replies.

## CI Warning

Before processing, check if CI is still running:
```bash
gh pr checks PR_NUMBER --json name,state,bucket
```

If any check has `bucket` of `pending`, print:
> "CI is still in progress — bot review comments may not have been posted yet. Consider running `checks` first."

Proceed anyway.

## Collect Unresolved Review Comments

Process **ALL review comments** — from bots and humans alike. Do NOT filter by author type.

Fetch everything in **three batch API calls** — line comments, PR-level comments, and **reviews** —
then filter and cross-reference in memory. Do NOT make per-comment API calls for reply checks.

### Step 1: Batch-fetch all review comments

```bash
gh api repos/{owner}/{repo}/pulls/PR_NUMBER/comments --paginate --slurp \
  | jq '[.[][] | {id, body, path, line, user_login: .user.login, user_type: .user.type, in_reply_to_id: .in_reply_to_id, created_at}]'
```

**Important**:
- **`--paginate --slurp | jq`, never `--paginate --jq`.** With `--jq` the filter runs PER PAGE, so a multi-page PR emits concatenated arrays (`[...][...]`) and a naive parse silently keeps only page 1 — dropping exactly the comments this skill exists to answer. (`gh` rejects `--slurp` together with `--jq`, hence the pipe. Measured on a real multi-page PR.)
- Do NOT attempt to strip HTML comment markers (`<!-- -->`) inside `--jq` — `gh` escapes `!` as `\!` in jq strings, causing parse errors. Fetch the raw `body` and strip bot noise in your own code after fetching.

### Step 2: Batch-fetch all issue comments

```bash
gh api repos/{owner}/{repo}/issues/PR_NUMBER/comments --paginate --slurp \
  | jq '[.[][] | {id, body, user_login: .user.login, user_type: .user.type, created_at}]'
```

### Step 3: Batch-fetch all REVIEWS

Steps 1 and 2 fetch line comments and PR-level comments. Neither returns a **review** — the object a
reviewer creates when they submit, carrying its own summary `body` and a `state`. A reviewer who
writes their whole review in that summary and leaves no line comments is invisible to Steps 1–2. That
is not hypothetical: one run reported *"I checked CI on my last pass but not for new threads"* and a
review sat three days until a second human escalated.

```bash
# --slurp + external jq, NOT --paginate --jq: with --jq the filter runs PER PAGE, so a PR with more
# than one page emits concatenated arrays ([...][...]) rather than one. (gh rejects --slurp WITH
# --jq, hence the pipe.) Measured on a real multi-page PR.
gh api repos/{owner}/{repo}/pulls/PR_NUMBER/reviews --paginate --slurp \
  | jq '[.[][] | {id, state, body, user_login: .user.login, user_type: .user.type, submitted_at}]'
```

Two distinct signals come out of this, and you need both:

- **Summary bodies.** A non-empty `body` on a `COMMENTED` or `CHANGES_REQUESTED` review is review
  feedback — treat it exactly like an issue comment in Steps 4–6. (An empty body is normal: the
  reviewer put everything in line comments, which Step 1 already has.)
- **Per-reviewer state.** A reviewer's effective state is their latest review **among
  `APPROVED` / `CHANGES_REQUESTED` / `DISMISSED` only**. `COMMENTED` and `PENDING` do NOT change it —
  that is GitHub's rule, and getting it wrong reproduces the exact gap this step exists to close: a
  reviewer who requests changes and later adds any comment-only review would compute as `COMMENTED`,
  never get re-requested, and leave the PR merge-blocked with nobody asked to look.
  (Still harvest `COMMENTED` **bodies** — they are feedback. Just don't let them set state.)
  ```bash
  gh api repos/{owner}/{repo}/pulls/PR_NUMBER/reviews --paginate --slurp \
    | jq '[.[][] | select(.state=="APPROVED" or .state=="CHANGES_REQUESTED" or .state=="DISMISSED")]
          | group_by(.user.login) | map(max_by(.submitted_at) | {user: .user.login, state, submitted_at})'
  ```
  **Cross-check against `reviewDecision`** below: if it reads `CHANGES_REQUESTED` and no reviewer
  computes to that state, your computation is wrong — do not exit clean on it.

Also record the aggregate, which is what the PR page shows:
```bash
gh pr view PR_NUMBER --json reviewDecision,reviewRequests
```

### Step 4: Identify actionable comments

Keep all comments that contain review feedback — from **any author** (bot or human). Skip:
- Comments that are purely informational (CI status reports, merge notifications)
- Comments from the PR author themselves (self-comments are usually notes, not review requests)
- Comments that are clearly just acknowledgments ("LGTM", "Looks good", thumbs up)

### Step 5: Filter out already-resolved comments

Using the batch-fetched data (no additional API calls):

- **For review comments**: Build a map of `in_reply_to_id -> [reply comments]`. A comment is resolved if any reply in its thread contains `[babysit-pr]` in the body.
- **For issue comments**: Scan all issue comments after the target comment. If any subsequent comment contains `[babysit-pr]` and references the comment (by quoting or by proximity), consider it resolved.
- **For review summary bodies**: a review has no `in_reply_to` to hang a reply from, so it is resolved only when a later issue comment contains `[babysit-pr]` and cites the review (quote a line of it, or name the reviewer and `submitted_at`), **or** when that same reviewer has submitted a newer `APPROVED`/`CHANGES_REQUESTED`/`DISMISSED`
  review (a later `COMMENTED` is more feedback, not an answer to the earlier body). Absent both, it is unresolved — an unanswerable thread must not read as an answered one.

### Step 6: Early exit, and what "done" means

**Early exit, decided now:** if no unresolved comments remain **and** no reviewer's latest state is
`CHANGES_REQUESTED`, print "No unresolved comments" and exit.

**Terminal state, judged after the processing and reply steps below:** the pass is complete when every
reviewer whose latest state is `CHANGES_REQUESTED` is accounted for by the rule that matches their case:

- **You fixed anything they raised** → their login MUST be in `reviewRequests`. A reply alone does
  not count: the whole point of the "Re-request review" step is that a pushed fix leaves their review
  standing until they are asked again, so finishing on "I replied" reinstates the defect this step
  exists to remove.
- **Everything they raised was deferred or already answered in an earlier pass** → that step forbids
  re-requesting, so a reply is the terminal state here.

A standing `CHANGES_REQUESTED` is work even when every line comment is answered — it is what keeps
the PR unmergeable. But nothing in this skill can clear it: only that reviewer submitting a new
review, or a dismissal, can. So "answered and re-requested" IS the terminal state.

There is also a state where a re-request is **forbidden** and must not be waited on: every point that
reviewer raised was deferred, or was already answered in an earlier pass. Report it as
`blocked on <reviewer> (deferred — awaiting their read)` rather than looping on a condition nothing in
this pass can satisfy.

Report either as `blocked on <reviewer>` and exit — never loop waiting for a state you cannot change.

## Process Each Comment

For each unresolved comment:

### Gather Context

- Read the file referenced in the comment (`path` field for review comments)
- Read the PR diff for that file: `gh pr diff PR_NUMBER -- FILE_PATH`
- Parse the comment body to understand what's being requested

### Classify — FIX or DEFER

**FIX** (actionable, clear, safe):
- Bug reports (null checks, off-by-one, missing error handling)
- Security issues (injection, XSS, secrets exposure)
- Style/convention violations (naming, formatting, imports)
- Straightforward performance issues (N+1, unnecessary allocations)
- Missing types or type errors
- Unused imports/variables
- Test improvements with clear direction
- Transaction safety (wrapping related operations in a transaction)

**DEFER** (subjective, risky, or out of scope):
- Architectural suggestions ("consider redesigning this module")
- Subjective preferences without project convention backing
- False positives (reviewer misunderstood the code)
- Changes requiring >50 lines of modification across multiple files
- Suggestions that contradict project conventions (check CLAUDE.md)
- Performance suggestions requiring benchmarking to validate
- Suggestions about code not in the PR diff

### Apply Fix or Draft Deferral

**If FIX**:
- Make the code change using Edit tool
- Keep changes minimal and focused on what the comment requests
- NEVER modify files outside the PR diff (live mode) or outside the files referenced by comments (follow-up mode)
- If the fix would touch >50 lines, STOP and ask the user for confirmation before proceeding
- Stage and note the fix for the commit

**If DEFER**:
- Do not modify any code
- Prepare a deferral reply explaining why
- **File it ONLY if it is real work you are scoping out of this PR** — architectural, >50 lines,
  needs benchmarking, or about code outside the diff. `gh issue create` referencing this PR, and put
  the issue number in the reply: a deferral reply alone is a note in a thread that closes with the PR.
  File it with the recipe below — the body goes through the shared formatter, not composed by hand:

```bash
# RECIPE: deferral-issue
# The PR number is a SHELL VARIABLE, not a placeholder to eyeball. Unset, this line stops the whole
# recipe with a message; a placeholder left unsubstituted would file an issue reading "Deferred from
# #PR_NUMBER", which no validator can catch because it is a perfectly good string.
PR=PR_NUMBER   # the number from init. Set it in THIS shell call: shell state does not persist between tool calls.
PR="${PR:?set PR to the pull request number this review is on}"

# DEDUPE FIRST. babysit-pr is built to loop — the full cycle re-runs `comments` up to five times, and a
# later run of the skill walks the same unresolved threads again — so a second pass files the same
# issue again. The opening phrase of the body below is the search key; keep them identical.
gh issue list --search "Deferred from #$PR" --state all --json number,title

# Scratch in a throwaway directory, NOT in the repo: filing an issue must not leave an untracked
# file in someone's tree. `gh issue create` files against whatever repo cwd resolves to — the PR's
# repo when this skill runs. The `&&` are load-bearing: `> file` truncates before the command that
# fills it runs, so an unchained `gh issue create` files an empty issue if the write fails.
D="$(mktemp -d)" &&
cat > "$D/deferral.md" <<EOF
Deferred from #$PR during review. Pick this up separately — #$PR is not blocked on it.

<the reviewer's point, restated as work, with path:line>

<details><summary><b>Why it was deferred rather than fixed in #$PR</b></summary>

<which DEFER category it fell into, and the reviewer's own words for it>

</details>
EOF
gh issue create --title "<the deferred work, stated as work>" --body-file "$D/deferral.md" &&
rm -rf "$D"
```

- **Never file a rejection.** A false positive, a subjective preference with no convention behind it,
  or a suggestion that contradicts project conventions is *declined*, not deferred — filing it turns
  a correctly-rejected item back into open work. Reply, and leave it closed.

### Queue Reply

Queue replies — do NOT post them yet. They are posted after the commit.

## Commit, Push, and Reply

### 1. Commit and push (if any fixes were made)

```bash
git add <specific files that were changed>
git commit -m "fix: address review comments

- <bullet point for each fix made>"
```

- **Live mode**: `git push origin HEAD_REF_NAME`
- **Follow-up mode**: `git push --force-with-lease origin HEAD_REF_NAME`

Never use `git add -A` or `git add .`.

### 2. Post FIX replies

**For review comments** (code-line comments), reply in-thread:
```bash
gh api repos/{owner}/{repo}/pulls/PR_NUMBER/comments \
  -X POST -f body="REPLY_BODY" -F in_reply_to=COMMENT_ID
```
Do NOT use `/pulls/comments/COMMENT_ID/replies` — it returns 404.

**For issue comments** (PR-level):
```bash
gh pr comment PR_NUMBER --body "REPLY_BODY"
```

**Live mode reply format**:
```
[babysit-pr] Fixed in COMMIT_SHA.

<Brief explanation of what was changed and why>
```

**Reply format for a REVIEW SUMMARY body** (any mode) — it must carry the citation, because a review
has no thread to reply into and Step 5 resolves it by that citation alone:
```
[babysit-pr] Fixed in COMMIT_SHA — re: @LOGIN's review of SUBMITTED_AT.

<Brief explanation of what was changed and why>
```

**Follow-up mode reply format**:
```
[babysit-pr] Will be addressed in follow-up PR #FOLLOW_UP_PR_NUMBER.

<Brief explanation of what will be changed and why>
```

### 3. Re-request review from anyone whose latest state is CHANGES_REQUESTED

A pushed fix does **not** clear a `CHANGES_REQUESTED` review. GitHub leaves it standing until that
reviewer is asked again, so the PR keeps reading `REVIEW_REQUIRED` with nobody aware they are being
waited on. One PR sat green for weeks still reading `REVIEW_REQUIRED` while the bug it fixed kept firing in
production.

Queue this like any other reply — it is posted after the push, alongside them. (In orchestrator mode
the orchestrator posts it in its Phase 3; see `orchestrator.md`.)

For each reviewer whose latest review state is `CHANGES_REQUESTED` **and** whose points you actually
addressed in this pass:

```bash
# IDEMPOTENCE GUARD — check before adding. /pulls/{n}/reviews is append-only, so their latest state
# stays CHANGES_REQUESTED until they submit a NEW review: a re-request does not clear it. Without
# this check the outer loop (up to 5 iterations) re-adds the same reviewer and re-posts the same
# comment every time it pushes.
gh pr view PR_NUMBER --json reviewRequests --jq '[.reviewRequests[].login]'
# only if LOGIN is absent from that list:
gh pr edit PR_NUMBER --add-reviewer LOGIN
```

The re-request needs one PR-level comment carrying its answer, and that comment **must cite the
review** — the citation is what Step 5 reads to mark the review body resolved next pass; without it
the same review is re-fixed and re-replied every iteration:
```
[babysit-pr] Fixed in COMMIT_SHA — re: @LOGIN's review of SUBMITTED_AT.

- <point> → <what changed>
```
**If the "Post FIX replies" step already queued a REVIEW-SUMMARY reply for this same reviewer, reuse it** — add the
`- <point> → <what changed>` lines to that comment and post nothing new. The two formats are
deliberately identical, so posting both would notify the reviewer twice for one pass.

Do **not** re-request when you deferred everything they raised: a deferral is an argument for them to
read, not a fix to re-review. Post the citing reply anyway (so the thread resolves) and leave the
review standing.

This is a single pass. Requesting the re-review is where your job ends — do not wait for it.

### 4. Post DEFER replies

Real work scoped out of this PR:
```
[babysit-pr] Deferred — <category>. Tracked in #ISSUE_NUMBER.

<Explanation>
```
Declined (false positive / convention conflict / subjective) — no issue, and don't imply one:
```
[babysit-pr] Declined — <category>.

<Explanation>
```
Real work you could not file (no `gh`, issues disabled) — say so rather than implying it is tracked:
```
[babysit-pr] Deferred — <category>. NOT TRACKED (issues unavailable) — needs filing by hand.

<Explanation>
```
If a reply points at other work, add the field — a missing link is then visible, not implied:
```
Superseded by: <SHA | #PR | #ISSUE>
```
Never prose describing a fix that exists somewhere. **link it** — a commit SHA, PR, or issue. Prose describing a fix that exists somewhere is how a recurring
customer-facing 500 stayed live nine extra days after a PR was closed citing a script nobody had
committed.

### 5. Follow-up PR (follow-up mode only)

If fixes were made in follow-up mode, create the follow-up PR.

**Use the host's PR tool when one is available** (some agent hosts expose a create-pull-request tool
and disable `gh pr create`; theirs authors the PR as the prompting user and tracks it) with the title
and body below. Fall back to `gh pr create` where it works:
```bash
gh pr create --title "fix: address review comments from #NUMBER" \
  --body "$(cat <<'EOF'
## Summary
Follow-up to #NUMBER. Addresses review comments.

- <bullet for each fix>

Original PR: #NUMBER
EOF
)"
```

Post replies on the **original PR** with the follow-up PR number.

## Summary

```
### Comments
| # | File | Author | Comment | Action | Details |
|---|------|--------|---------|--------|---------|
| 1 | path/to/file.rb:42 | cursor[bot] | "Missing null check" | Fixed (abc1234) | Added nil guard |
| 2 | path/to/file.rb:78 | @reviewer | "Consider redesign" | Deferred | Architectural, out of scope |
| 3 | (review summary) | @reviewer | "Two concerns…" | Fixed (abc1234) | Re-requested review |
```

Also report the review state, which is what decides whether the PR can merge:
```
### Reviews
| Reviewer | Latest state | Action |
|---|---|---|
| @reviewer | CHANGES_REQUESTED → re-requested | addressed in abc1234 |
| @other | APPROVED | — |
reviewDecision: REVIEW_REQUIRED
```
If nobody has reviewed at all, say so explicitly — "no reviewer has looked at this PR" is a finding,
not an empty table.

If follow-up mode: include follow-up PR URL.
