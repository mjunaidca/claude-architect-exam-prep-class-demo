# GitHub Actions and CI Triggers for Claude Code (or Any Agent)

How GitHub Actions decides *when* to run an agent, *what* it can touch, and *how* to wire that up safely. See `ci-notes.md` for the Claude-specific action configuration; this note focuses on triggers.

## 1. The mental model

```
event happens  ->  trigger (on:) matches  ->  job runs on a fresh runner  ->  steps run the agent
```

- **Workflow**: a YAML file in `.github/workflows/`.
- **Trigger**: the `on:` block. It lists the events that start the workflow.
- **Job**: a set of steps on one runner. Jobs run in parallel unless linked with `needs:`.
- **Step**: a shell command (`run:`) or a reusable action (`uses:`).
- **Runner**: a throwaway VM. Nothing persists between runs except what you push, upload or comment.

An agent is just another step. The trigger decides who or what can start it, which is the main security and cost boundary.

## 2. Common triggers and agent use cases

| Trigger | Fires when | Typical agent use |
| --- | --- | --- |
| `pull_request` | PR opened, updated, reopened | Automatic code review |
| `push` | Commits pushed to a branch | Post-merge checks, changelog or docs updates |
| `issues` | Issue opened, assigned, labeled | Triage, or start work when assigned |
| `issue_comment` | Comment on an issue or PR | `@agent` commands in chat |
| `pull_request_review_comment` | Inline review comment | "Fix this line" requests |
| `pull_request_review` | Review submitted | Act on a review's feedback |
| `schedule` | Cron time | Nightly dependency check, stale issue sweep |
| `workflow_dispatch` | Manual "Run workflow" button or API | On-demand tasks with inputs |
| `workflow_run` | Another workflow finishes | React to a failed CI run |
| `repository_dispatch` | External HTTP call to the API | Trigger from another system |

Note: `issue_comment` fires for comments on both issues and PRs. Check `github.event.issue.pull_request` if you need to tell them apart.

## 3. Filtering a trigger

Narrow `on:` so the agent only runs when it should.

```yaml
on:
  pull_request:
    types: [opened, synchronize, ready_for_review]   # which activity types
    branches: [main]                                  # target branch
    paths: ["src/**", "!**/*.md"]                     # only when these files change
  schedule:
    - cron: "0 3 * * 1-5"                             # 03:00 UTC, weekdays
  workflow_dispatch:
    inputs:
      task:
        description: "What should the agent do?"
        required: true
```

Then add job-level `if:` conditions for what `on:` cannot express:

```yaml
jobs:
  agent:
    if: |
      github.event_name == 'issue_comment' &&
      contains(github.event.comment.body, '@claude')
```

This is exactly how `claude.yml` in this repo works: it listens to several events, then the `if:` keeps only those that mention `@claude`.

## 4. Two ways to start an agent

1. **Interactive (tag mode)**: a human mentions the agent in an issue or PR comment. The instructions come from the comment. Good for ad hoc work.
2. **Automation**: a trigger such as `pull_request` or `schedule` plus a fixed `prompt:`. No human message is needed. Good for repeatable jobs like review.

Examples in this repo:

| File | Trigger | Style |
| --- | --- | --- |
| `claude.yml` | `issue_comment`, `issues`, review events with `@claude` | Interactive |
| `claude-code-review.yml` | `pull_request` | Automation |
| `markdown-only.yml` | `pull_request`, `push` to `main` | Plain CI gate (no agent) |

## 5. What the agent receives

The trigger's event payload is available as the `github.event` context, for example the issue body, comment text, PR number and author. Action wrappers such as `claude-code-action` read it and build the agent's context for you. If you build your own agent step, pass what it needs explicitly:

```yaml
- run: ./my-agent --issue "${{ github.event.issue.number }}"
```

Warning: never interpolate untrusted text such as an issue title or comment body directly into a `run:` script. Pass it through `env:` instead, to avoid shell injection:

```yaml
- run: ./my-agent --text "$BODY"
  env:
    BODY: ${{ github.event.comment.body }}
```

## 6. Permissions and secrets

- `permissions:` sets what the job's `GITHUB_TOKEN` may do. Start from the minimum (`contents: read`) and add only what is needed (`pull-requests: write` to comment, `contents: write` to push).
- Secrets (API keys, OAuth tokens) live in repo or org **Secrets** and are read as `${{ secrets.NAME }}`.
- Fork PRs: `pull_request` from a fork runs with a read-only token and no secrets. `pull_request_target` runs with secrets against the base branch, so it is dangerous to combine with an agent that checks out or runs fork code. Avoid that combination.
- Tokens from the GitHub App cannot change files under `.github/workflows/`.

## 7. Safety checklist for agent triggers

- Restrict who can trigger it (write access only, or an explicit `if:` on the author).
- Treat issue, PR and comment text as untrusted input (prompt injection).
- Give an explicit tool allowlist and minimal permissions.
- Bound cost: `timeout-minutes`, a turn limit, and `concurrency:` to avoid duplicate runs.
- Prevent loops: an agent that comments or pushes can retrigger itself. Filter out bot actors (`github.actor != 'claude[bot]'`) or use triggers that bot activity does not fire.
- Keep branch protection and required reviews so agent PRs are never merged unseen.

```yaml
concurrency:
  group: agent-${{ github.event.issue.number || github.event.pull_request.number }}
  cancel-in-progress: true
```

## 8. Debugging triggers

- Workflow not running? The workflow file must exist on the branch GitHub evaluates. For `issue_comment`, `issues` and `schedule`, that is the **default branch**, so a new workflow only fires after it is merged.
- Check the **Actions** tab, or `gh run list` and `gh run view <id> --log`.
- `if:` evaluated false shows as a skipped job. Print `github.event_name` and `github.event.action` to see what arrived.
- `schedule` runs can be delayed and are disabled after 60 days of repo inactivity.

## 9. Works for any agent, not just Claude

Nothing above is Claude-specific. Any agent CLI or SDK can run as a step: install it, give it credentials via a secret, pass it a prompt built from the event, and let it use git and `gh` to push a branch and open a PR. The trigger, permissions, and review gates are what make it safe.

## Review questions

1. Which trigger fires for a comment on a PR, and how do you tell it from an issue comment?
2. Why does a new `@mention` workflow not work until it is merged to the default branch?
3. What is the risk of `pull_request_target` with an agent?
4. How do you stop an agent's own comment from retriggering it?
5. When would you pick `schedule` over `issue_comment`?
