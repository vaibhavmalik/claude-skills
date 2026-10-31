---
name: pr-description
description: Use when writing or updating a pull request description.
---

# PR Description

The reader is a capable engineer who knows nothing about this code or this task. They should understand what changed and why in under a minute, without opening the code.

**Visible part** (under 200 words, plain words, no code, file names or internal ids), under short bold headings that fit this PR:
1. The situation today and what is wrong or missing in it, concretely enough to picture.
2. What this changes, as its effect on the product or system.
3. What was checked, and what wasn't.
4. Risks or anything left out, if there is any.

Use real names and explain only terms specific to this codebase. Don't invent labels.

**Then one collapsed section**, `<details><summary><b>Technical details</b></summary>`, containing the files, the key design choice, the commands run and any ticket references.

**Write from the code: the final diff for what the PR does, the base branch for how things worked before.** The issue and commit messages describe intent and earlier versions, so check every claim against the code before you use it.

**Before posting**, give the draft and the branch to a fresh subagent. Ask it to list every claim the code doesn't support, with the line that contradicts it, and every visible sentence a reader new to this code doesn't need or can't follow. Fix or cut each one.

After any push that changes behaviour, rewrite whatever parts of the description are no longer true.
