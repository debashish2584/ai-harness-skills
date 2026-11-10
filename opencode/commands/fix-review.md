---
description: Triage review comments, then fix only explicitly approved valid findings
agent: review-fixer
subagent: false
---

Load the `review-comment-triage` skill and follow it exactly.

Review comments, PR reference, or target:
$ARGUMENTS

Phase 1 is triage only: verify every comment, classify it, and produce a minimal fix plan. Do not edit anything yet. Stop and wait for me to approve specific comment IDs.

Phase 2 starts only after I say which IDs to apply. Apply only approved `VALID_*` findings, one at a time, and show the diff and test result after each.

Working tree:

!`git status --short && git diff --stat && git diff --name-only`

Staged:

!`git diff --cached --stat && git diff --cached --name-only`

Read the actual diff for the files relevant to each comment with a read-only command such as `git diff -U20 -- <path>`. If both working-tree and staged diffs are empty, ask which base branch or commit range to compare, or which files the review comments refer to.
