# CI Notes: Claude Code as a CI Worker

Exam-prep notes on running Claude Code inside CI, using this repo as the worked example.

## 1. Get the GitHub CLI (`gh`)

`gh` is the GitHub command line tool. Claude Code uses it (and the GitHub API) to read issues and PRs and to open PRs.

```bash
# macOS
brew install gh

# Debian/Ubuntu
sudo apt install gh

# Windows
winget install --id GitHub.cli
```

Then authenticate and check:

```bash
gh auth login      # choose GitHub.com, HTTPS, login with browser
gh auth status
```

Commands worth knowing:

| Task | Command |
| --- | --- |
| List issues | `gh issue list` |
| View an issue | `gh issue view 4` |
| Create a PR | `gh pr create --base main --fill` |
| Check CI runs | `gh run list` / `gh run view <id>` |
| Set a secret | `gh secret set CLAUDE_CODE_OAUTH_TOKEN` |

## 2. CI concept, using a GitHub project

CI (continuous integration) means every change is automatically checked when it is pushed or proposed. In GitHub this is done with **GitHub Actions**: YAML files in `.github/workflows/`.

A workflow has:

- **Trigger (`on:`)**: the event that starts it (`pull_request`, `push`, `issue_comment`, ...).
- **Jobs**: units of work, each on a fresh runner (`runs-on: ubuntu-latest`).
- **Steps**: commands or reusable actions (`uses: actions/checkout@v4`).
- **Permissions**: what the job's token may do.

This repo has three workflows:

| File | Trigger | Purpose |
| --- | --- | --- |
| `markdown-only.yml` | every PR, push to `main` | Fails if any tracked file is not `.md` (except `.gitignore` and `.github/`) |
| `claude.yml` | `@claude` mentioned in issue, comment or review | Claude does the requested work |
| `claude-code-review.yml` | PR opened/updated | Claude reviews the PR automatically |

Because of `markdown-only`, notes in this repo must be Markdown. That is a simple example of a CI gate.

## 3. Live demo: Claude as a CI worker

Flow demonstrated in class (this very issue):

1. Install the Claude GitHub app / workflows (`/install-github-app` in Claude Code).
2. Add the `CLAUDE_CODE_OAUTH_TOKEN` repo secret.
3. Open an issue describing the task.
4. Comment `@claude Work on this issue and create a pr to main`.
5. `claude.yml` fires. Claude checks out the repo, creates a branch (`claude/issue-<n>-...`), edits files, commits, pushes, and reports progress by updating a single comment.
6. A PR is opened; `markdown-only` and `claude-code-review` run on it.
7. A human reviews and merges.

Key idea: Claude is treated like any other contributor. It works through branches and PRs, and the normal CI checks still gate its output.

## 4. In-depth configuration of Claude Code as a CI worker

The action is `anthropics/claude-code-action@v1`.

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 1
- uses: anthropics/claude-code-action@v1
  with:
    claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    prompt: "..."            # optional; omit to follow the @claude comment
    claude_args: "--max-turns 10 --allowedTools Bash(gh pr *)"
```

### Authentication
- `claude_code_oauth_token` (Claude subscription) or `anthropic_api_key`. Always store it as a repo **secret**, never in the YAML.
- Bedrock and Vertex are supported for enterprise setups.

### Triggers and modes
- **Tag mode**: responds to `@claude` (issues, PR comments, reviews). Set up in `claude.yml`.
- **Automation mode**: runs when a `prompt` is given, with no mention needed (e.g. review every PR, nightly jobs via `schedule`).
- `trigger_phrase` changes the default `@claude`.

### Permissions
- Give the job the least privilege: `contents`, `pull-requests`, `issues`, and `id-token: write` (for OIDC auth).
- `actions: read` (with `additional_permissions`) lets Claude read CI results on PRs, so it can fix failing checks.
- GitHub Apps cannot modify `.github/workflows/`, so Claude cannot edit workflow files.

### Controlling what Claude can do (`claude_args`)
Passed straight to the Claude Code CLI:

- `--allowedTools` / `--disallowedTools`: restrict tools, e.g. `Bash(gh pr *)`, `Bash(npm test)`.
- `--max-turns`: cap the agent loop to bound cost and runaway behavior.
- `--model`: pick the model.
- `--append-system-prompt`: add standing instructions.
- `--mcp-config`: attach MCP servers.

### Repo guidance
- A `CLAUDE.md` at the repo root is read automatically: put build/test commands, style rules, and conventions there.
- Plugins and marketplaces (`plugins`, `plugin_marketplaces`) can load things like the `code-review` plugin, as in `claude-code-review.yml`.

### Safety checklist
- Secrets only in GitHub Secrets.
- Minimal permissions and an explicit tool allowlist.
- Only trusted users should be able to trigger it (the action checks write access by default).
- Branch protection plus required reviews on `main`, so Claude's PRs are never auto-merged unseen.
- Treat issue and PR text as untrusted input (prompt injection risk).
- Set `--max-turns` and workflow `timeout-minutes` to limit cost.

## Exam-style review questions

1. Which workflow event lets a human trigger Claude from an issue comment?
2. Why does Claude need `id-token: write`?
3. How do you stop Claude from running arbitrary shell commands in CI?
4. Why can Claude not edit files under `.github/workflows/`?
5. What is the difference between tag mode and automation mode?
