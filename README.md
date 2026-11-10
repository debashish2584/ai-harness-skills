# AI Harness Skills

Portable, model-agnostic skills and harness templates for AI coding agents. The core skill is plain Markdown, so it can be adapted to OpenCode, Claude Code, Codex, or any agent that supports instruction files.

The first skill is **Review Comment Triage**: an evidence-first workflow for validating review comments before changing code.

## Why this exists

AI code reviewers are often given less context than they need. They may review only a diff, miss cross-feature dependencies, or confidently reference files, symbols, line numbers, and behavior that do not exist in the repository.

Fixing every review comment blindly makes this worse. This skill treats each comment as a hypothesis:

- Verify it against the actual repository.
- Check callers, tests, config, contracts, migrations, and feature flags.
- Classify it as valid, a false positive, a hallucination, out of scope, or needing context.
- Plan the smallest safe fix.
- Apply only the findings a human explicitly approves.

## Repository layout

```text
skills/
  review-comment-triage/
    SKILL.md
opencode/
  agents/
    review-triage.md
    review-fixer.md
  commands/
    triage-review.md
    fix-review.md
```

The `skills/` directory contains the reusable procedure. The `opencode/` directory contains optional OpenCode agents and slash commands that wire the skill into a read-only triage phase and an approval-gated fix phase.

## Install in OpenCode

### Option A: point OpenCode at this repo's skills directory

Clone the repo somewhere permanent:

```bash
git clone https://github.com/debashish2584/ai-harness-skills.git ~/ai-harness-skills
```

Add the skills directory to your OpenCode config:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "skills": ["~/ai-harness-skills/skills"]
}
```

The `skills` value is resolved relative to the active OpenCode working directory for relative paths, or from your home directory for `~/` paths.

### Option B: copy or symlink the skill into a project

```bash
mkdir -p .opencode/skills
cp -R ~/ai-harness-skills/skills/review-comment-triage .opencode/skills/
```

Or symlink it:

```bash
ln -s ~/ai-harness-skills/skills/review-comment-triage .opencode/skills/review-comment-triage
```

### Optional: install the OpenCode agents and commands

Copy the templates into your project:

```bash
mkdir -p .opencode/agents .opencode/commands
cp ~/ai-harness-skills/opencode/agents/*.md .opencode/agents/
cp ~/ai-harness-skills/opencode/commands/*.md .opencode/commands/
```

OpenCode discovers `.opencode` from the current directory up to the project root. Commit `.opencode/` if you want the workflow shared with the team.

Verify the agents are visible:

```bash
opencode debug agents
```

You should see `review-triage` and `review-fixer`.

## Usage

### Triage only

```text
/triage-review
```

Then paste the review comments. The command runs the read-only `review-triage` agent, which verifies every claim and returns a report. It makes no edits.

You can also pass a PR reference:

```text
/triage-review #123
```

### Triage, then fix approved findings

```text
/fix-review
```

Paste the review comments. The `review-fixer` agent triages first, stops, and waits for approval.

Then reply with the comment IDs you want applied:

```text
Apply C1 and C3 only. Skip C2.
```

It applies one approved `VALID_*` finding at a time, shows the diff, runs the relevant test, and re-verifies the change.

### Direct agent use

If you prefer not to use the slash commands, select the `review-fixer` agent in OpenCode and ask it to triage and fix the comments. For an independent, fresh-context review, ask the primary agent to launch the `review-triage` subagent.

## How the workflow works

| Phase | What happens | Edits allowed? |
| --- | --- | --- |
| Scope map | Identify the feature, entry points, changed files, dependencies, tests, and docs | No |
| Parse | Split the review into `C1..Cn` and restate each as a falsifiable claim | No |
| Verify | Read the actual code, follow the call chain, check tests/config/history, answer 3 verification questions | No |
| Classify | Assign one verdict and a confidence level | No |
| Fix plan | Propose the smallest safe change, test, and risk for each valid finding | No |
| Approval | Wait for the human to choose comment IDs | No |
| Fix | Apply approved findings one at a time, show the diff, run tests, re-verify | Yes, approved only |

## Verdicts

| Verdict | Meaning |
| --- | --- |
| `VALID_BUG` | True, reachable, and introduced or exposed by this change |
| `VALID_DESIGN_RISK` | True and worth discussing, but not necessarily a defect |
| `VALID_TEST_GAP` | Behavior is untested and the gap matters |
| `ALREADY_ADDRESSED` | The diff or another layer already fixes it |
| `INVALID_FALSE_POSITIVE` | The code is correct; the reviewer missed existing handling |
| `INVALID_HALLUCINATION` | The referenced file, symbol, line, API, or behavior does not exist |
| `OUT_OF_SCOPE` | Pre-existing or unrelated, with no regression risk from this change |
| `PREFERENCE` | Style/naming/formatting with no project rule or correctness impact |
| `NEEDS_CONTEXT` | The claim cannot be verified because a file, ticket, or runtime fact is missing |
| `DUPLICATE` | Same root cause as another comment |

## Design principles

- **Evidence before verdict.** Every verdict cites an exact `path:line` and the tool result that proves it.
- **Read the whole path, not just the diff.** Check callers, callees, tests, config, migrations, events, feature flags, and public contracts.
- **Prefer no action over a weak finding.** False-positive fixes are worse than missed nits.
- **Read-only triage, approval-gated fixes.** The triage agent has `edit: deny`; the fixer uses `edit: ask` and `shell: ask`.
- **Model-agnostic.** No model is hardcoded. Agents inherit the session model, and the skill works with local or hosted models.
- **Batching for smaller context windows.** Process at most 5 comments per batch, or 3 when context is tight.
- **Uncertainty is allowed.** `NEEDS_CONTEXT` and `UNDETERMINED` are valid outcomes; a forced verdict is worse than an honest gap.

## Customizing

- Add project-specific checks to the skill's Phase 0 scope map.
- Add your build, test, and lint commands to `AGENTS.md` so every agent can find them.
- If the permission prompts in `review-fixer` are too noisy, change `edit: ask` to `edit: allow` and `shell: ask` to `shell: allow`, while keeping `review-triage` read-only.
- If you want a specific model for the commands, add `model: provider/model` to the command frontmatter. Omit it to stay model-agnostic.

## Compatibility

- The `SKILL.md` format is Markdown with YAML frontmatter and works with OpenCode skills.
- The agent and command files use OpenCode V2 frontmatter. Other harnesses can use the skill body as a prompt template.
- The workflow assumes a git repository and, for PR review, the GitHub CLI (`gh`). If `gh` is unavailable, paste the review comments directly.
