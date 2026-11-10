# Claude Architect Exam Prep

Study notes and preparation material for the Claude Architect exam.

## Status

Early stage. The repo contains Markdown files only.

## Structure

```
.
├── README.md                        # this file
├── agent-memory-and-state-persistence.md # saving facts and task state so agents remember across runs
├── api-errors-rate-limits-and-retries.md # handling API errors, rate limits, and safe retries
├── ci-notes.md                      # running Claude Code as a CI worker
├── context-window-management.md     # keeping an agent's context from filling up
├── designing-good-tools.md          # writing tool names, descriptions, schemas, and errors Claude can use
├── evaluating-prompts-and-agents.md # measuring whether prompts and agents work with evals
├── extended-thinking.md             # letting Claude reason first, and the loop rules it adds
├── github-actions-and-triggers.md   # GitHub Actions triggers for agents
├── human-in-the-loop-and-approval-gates.md # deciding which agent actions need human approval
├── message-batches-and-async-processing.md # bulk, discounted async requests and partial failure
├── model-context-protocol.md        # MCP: connecting agents to shared tool servers
├── model-selection-and-routing.md   # choosing and routing between models by quality, cost, and speed
├── observability-and-tracing-for-agents.md # logging traces and metrics to explain agent runs
├── prompt-caching.md                # reusing a stable prompt prefix to cut cost and latency
├── prompt-injection-and-agent-safety.md  # defending agents against injected instructions
├── streaming-responses.md           # receiving replies as events, including streamed tool calls and mid-stream failures
├── structured-outputs-and-validation.md  # getting machine-readable output and checking it
├── subagents-and-orchestration.md   # splitting work across agents with separate contexts
├── tool-use-and-agentic-loops.md    # how tool calls and the agent loop work
└── ...                              # topic notes added as Markdown files
```

## How to use

1. Work through the topic notes one at a time.
2. Add your own notes, practice questions, and summaries as `.md` files.
3. Keep one topic per file and use clear, descriptive filenames.

## Contributing

Notes are personal study material. Keep files in Markdown, one topic per file.
