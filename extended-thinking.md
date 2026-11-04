# Extended Thinking

Letting Claude reason before it answers. `tool-use-and-agentic-loops.md` showed the loop that runs tools, and `context-window-management.md` showed that everything shares a fixed window. This note covers a setting that trades extra tokens and time for better answers on hard problems, and the rules it adds to your loop.

> API details for thinking (parameter names, which models support it, how budgets work) change between model versions. Treat the code below as the shape of the idea and check the current docs for exact fields.

## 1. The big idea

With **extended thinking** turned on, Claude first writes out step-by-step reasoning in separate `thinking` blocks, then writes its final answer in normal `text` blocks. The reasoning is produced as output tokens, so you pay for it and it takes time.

> Analogy: scratch paper in an exam. Working a hard problem out on scratch paper makes mistakes less likely than blurting an answer. The analogy stops in two places: scratch paper costs the student nothing but thinking tokens are billed like other output, and more scratch paper does not guarantee a right answer. It helps most on multi-step problems and does little for simple lookups.

## 2. When it helps

| Task | Thinking worth it? |
| --- | --- |
| Multi-step math, logic, or planning | Yes |
| Debugging or reviewing tricky code | Yes |
| Choosing between several tools in a complex agent task | Often |
| Simple classification, extraction, or formatting | Usually no |
| Latency-sensitive chat replies | Usually no |

Start with thinking off. Turn it on when evals show the model fails on reasoning-heavy cases (see `evaluating-prompts-and-agents.md`), and measure whether it actually fixed them.

## 3. Turning it on and setting a budget

```python
response = client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=16000,
    thinking={"type": "enabled", "budget_tokens": 8000},
    messages=[{"role": "user", "content": "Plan the migration steps."}],
)

for block in response.content:
    if block.type == "thinking":
        pass                      # reasoning; log it, do not show it as the answer
    elif block.type == "text":
        print(block.text)         # the final answer
```

- The **thinking budget** caps how many tokens Claude may spend reasoning. A larger budget allows deeper reasoning but costs more and is slower.
- Thinking tokens count toward `max_tokens`, so keep `max_tokens` larger than the budget or the answer can be cut off (`stop_reason` of `max_tokens`).
- Some newer models can decide how much to think on their own. Check which style your model uses.

## 4. Thinking inside the agentic loop

When Claude thinks and then calls a tool, the thinking blocks are part of that assistant turn. When you send back the `tool_result`, you must return the **whole assistant turn unchanged**, thinking blocks included.

```python
messages.append({"role": "assistant", "content": response.content})  # keep thinking blocks
messages.append({"role": "user", "content": tool_results})
```

If you rebuild the assistant message from only the `text` and `tool_use` blocks, the request can fail or the model loses its line of reasoning mid-task. Thinking blocks may carry a signature that the API checks, so never edit them.

## 5. Costs and interactions

| Area | What to know |
| --- | --- |
| Cost | Thinking tokens are billed as output tokens |
| Latency | More thinking means a slower first useful token. Use streaming for long requests |
| Context window | Thinking from earlier turns may be dropped by the API to save space, but the current turn's thinking still counts |
| Prompt caching | Changing thinking settings between requests can invalidate cached message content (see `prompt-caching.md`) |
| Tool choice | Forcing a specific tool with `tool_choice` generally does not work with thinking. Use automatic choice (see `tool-use-and-agentic-loops.md`) |
| Retries | A long thinking request is more likely to hit timeouts, so use streaming and sensible retry rules (see `api-errors-rate-limits-and-retries.md`) |

## 6. Is the thinking trustworthy?

The reasoning text is useful for debugging and for seeing why an agent chose a step, but it is not a guaranteed faithful record of every factor behind the answer. Do not use it as the only safety check. Gate risky actions by what the agent is about to *do* (see `human-in-the-loop-and-approval-gates.md`) and validate outputs separately (see `structured-outputs-and-validation.md`).

## 7. Common mistakes

- Turning thinking on everywhere "just in case", then paying more for simple tasks.
- Setting `max_tokens` below the thinking budget.
- Dropping or editing thinking blocks when returning tool results.
- Showing raw thinking to end users as if it were the final answer.
- Treating reasoning text as proof that an action is safe.
- Raising the budget without an eval to show it helps.

## Check yourself

1. What are the two kinds of output blocks you see when thinking is on?
2. Why do thinking tokens affect your bill and your `max_tokens` setting?
3. In a tool loop, what must you do with thinking blocks when you send back a tool result?
4. Name two kinds of task where thinking usually is not worth it.
5. Why should you not rely on the thinking text as your only safety check?
6. In the scratch paper analogy, where does the analogy stop matching?

<details>
<summary>Answers</summary>

1. `thinking` blocks (reasoning) and `text` blocks (the final answer).
2. They are billed as output tokens and count toward `max_tokens`, so too small a limit can cut off the answer.
3. Send back the full assistant turn unchanged, thinking blocks included.
4. For example: simple classification or extraction, and latency-sensitive chat replies.
5. It may not be a complete or faithful account of why the model acted. Gate and validate the actual actions and outputs instead.
6. Thinking costs tokens and time, and more of it does not guarantee a correct answer.

</details>
