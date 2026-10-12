# epiphanyapps/.github

Shared GitHub configuration for the `epiphanyapps` org.

## Shared AI PR reviewer

[`.github/workflows/pr-agent-review.yml`](.github/workflows/pr-agent-review.yml)
is a reusable workflow that runs [Qodo PR-Agent](https://github.com/qodo-ai/pr-agent)
on Claude (our own Anthropic key, no Qodo subscription). On every PR that opens
for review it:

- posts an **adversarial review** ("PR Reviewer Guide" comment): acceptance
  criteria of the linked issue, edge cases, security and data scoping,
  unexpected production writes, missing tests, each finding graded
  blocker / should-fix / nit with a `file:line`. It follows the repo's
  `CLAUDE.md` / `AGENTS.md` when present;
- rewrites the PR description once (optional, `auto_describe`);
- **fails the job if no review was posted.** pr-agent swallows model and auth
  errors and exits 0; when the key expired on 2026-10-02 every reviewer stayed
  green for a week while reviewing nothing. A red "Verify a review was posted"
  step now means: check the key.

It never approves or merges. Comment commands work on any PR:
`/review`, `/describe`, `/improve`, `/ask "..."` (from owners, members and
collaborators only; bots, drafts and fork PRs are skipped automatically).

This repo is **public** on purpose: public repos can only call reusable
workflows from public repos. Nothing secret lives here.

### Add it to a repo

1. Set the key on that repo (see below).
2. Add `.github/workflows/pr_agent.yml`:

```yaml
name: AI PR Reviewer

on:
  pull_request:
    types: [opened, reopened, ready_for_review]
  issue_comment:
    types: [created]

permissions:
  contents: read
  issues: write
  pull-requests: write

jobs:
  review:
    uses: epiphanyapps/.github/.github/workflows/pr-agent-review.yml@v1
    with:
      extra_instructions: >-
        Optional repo-specific rules, appended to the shared adversarial base.
    secrets:
      ANTHROPIC_KEY: ${{ secrets.ANTHROPIC_KEY }}
```

On the PR that adds it, the reviewer runs from the PR's own copy. Comment
commands (`/review`) use the default branch's copy, so they work once merged.

### Inputs

| Input | Default | Notes |
|---|---|---|
| `extra_instructions` | `""` | Appended to the shared adversarial instructions. |
| `model` | `anthropic/claude-opus-5` | litellm model name. |
| `fallback_models` | `["anthropic/claude-sonnet-5"]` | Plain TOML array. **Not** `@json [...]`, which pr-agent receives as a literal string. |
| `context_files` | `["CLAUDE.md", "AGENTS.md"]` | Plain TOML array; missing files are skipped. |
| `auto_describe` | `true` | Describe once on open/reopen/ready; never on a push. |
| `review_on_push` | `false` | Incremental `/review -i` on every push. The caller must also add `synchronize` to its `pull_request` types. |
| `max_model_tokens` | `64000` | pr-agent's default (32000) silently truncates big diffs. |

Secret: `ANTHROPIC_KEY` (required).

Settings that only a repo-level `.pr_agent.toml` can express (for example
`[pr_reviewer] require_ticket_analysis_review = false`) still work: pr-agent
reads that file from the repo's default branch. Keys set by this workflow
(model, fallbacks, context files, instructions) win over the toml.

### Rotate the key

GitHub **Free** does not share org secrets with private repos, so every repo
carries its own `ANTHROPIC_KEY`. Rotate it in each one:

```bash
for r in GlanceMenu bingo trading AlfajoresNY platform VerdeDulce ductless-crew-site; do
  gh secret set ANTHROPIC_KEY --repo "epiphanyapps/$r"   # paste the key when prompted
done
```

Then comment `/review` on any open PR to confirm, or re-run the last reviewer
run with `gh run rerun <id> --repo epiphanyapps/<repo>`.

### Changing the shared workflow

Callers pin `@v1`. A backwards-compatible change: commit to `main`, then move
the tag (`git tag -f v1 && git push -f origin v1`). A breaking change (renamed
or removed input, new required input): tag `v2` and move callers over.

The pr-agent image is pinned by digest
(`pragent/pr-agent:0.44.0-github_action`); bump it deliberately.
