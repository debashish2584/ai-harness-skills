---
description: Triage review comments against the repo before changing code
agent: review-triage
subagent: true
---

Load the `review-comment-triage` skill and follow it exactly.

Review comments, PR reference, or target:
$ARGUMENTS

Use the change summary below as the starting point. Then inspect the relevant files, callers, tests, config, and cross-feature dependencies yourself. Do not assume the change is limited to the listed files.

Working tree:

!`git status --short && git diff --stat && git diff --name-only`

Staged:

!`git diff --cached --stat && git diff --cached --name-only`

Read the actual diff for the files relevant to each comment with a read-only command such as `git diff -U20 -- <path>`. If both working-tree and staged diffs are empty, ask which base branch or commit range to compare, or which files the review comments refer to.
