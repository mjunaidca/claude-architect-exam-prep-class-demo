# Context Window Management

How to keep an agent's working memory from filling up. `tool-use-and-agentic-loops.md` showed that every loop turn appends a tool call and result to the history, and `prompt-caching.md` showed how to make that history cheaper. This note covers the next problem: the history can still grow past what the model can hold.

## 1. The big idea

The **context window** is the maximum amount of text (measured in tokens) the model can consider in one request. Your system prompt, tool definitions, the whole conversation, tool results, and the model's reply all share it.

> Analogy: a desk. Everything Claude is working on must be laid out on the desk. The desk has a fixed size, so once it is covered you must put something away before adding anything new. The analogy stops at one point: a desk is cleared by you choosing what to remove, and Claude does not tidy it for you. If you send too much, the request fails or older material gets dropped by whatever policy your code or tooling applies.

Bigger context is not automatically better. Long contexts cost more, are slower, and relevant facts can get buried among irrelevant ones.

## 2. What fills the window

| Source | Typical growth |
| --- | --- |
| System prompt and tool definitions | Fixed, but large if you attach many tools or MCP servers |
| User and assistant messages | Slow, linear |
| Tool results (file contents, search output, logs) | Fast, often the biggest cause |
| Model output | Counts toward the limit and toward `max_tokens` |

In an agentic loop, a single `cat` of a big log file can use more space than the entire conversation so far.

## 3. Measuring before you send

Do not guess. The API offers a token-counting endpoint, and each response reports its own usage.

```python
count = client.messages.count_tokens(
    model="claude-sonnet-5-5",
    system=SYSTEM_PROMPT,
    tools=TOOLS,
    messages=messages,
)
print(count.input_tokens)  # compare against your budget
```

Keep a budget: for example, act when input reaches a chosen share of the limit, leaving room for the reply. The exact limits depend on the model, so check the current docs.

## 4. Strategies, from cheapest to heaviest

| Strategy | What you do | Trade-off |
| --- | --- | --- |
| Trim tool output | Return only needed fields, paginate, or cap the length | Claude may need to ask again for more |
| Retrieve on demand | Keep big documents outside the prompt and fetch only the relevant part with a tool | Needs a good search or lookup tool |
| Clear old tool results | Drop stale tool results from the history once they have served their purpose | Claude can no longer see them |
| Summarize | Replace older turns with a short summary and keep recent turns verbatim | Detail is lost, and the summary can be wrong |
| Use a subagent | Let a separate agent do a messy task in its own context and return only a short report | Extra call, and the report must carry what the parent needs |
| Write notes to a file | Save key decisions or progress outside the context, then reread them when needed | You must design what to save |

Start with the cheapest option that works. Trimming at the source is almost always better than summarizing later.

## 5. Summarizing safely

A summary replaces detail, so write it to keep what the next step needs:

- The goal and any constraints the user gave.
- Decisions made and the reason for them.
- What is finished, what is in progress, and what remains.
- Exact identifiers that must not be paraphrased (file paths, IDs, error messages).

```python
if count.input_tokens > BUDGET:
    summary = summarize(messages[:-6])          # a separate model call
    messages = [{"role": "user", "content": summary}] + messages[-6:]
```

Be careful where you cut. A `tool_use` block must stay paired with its `tool_result`, or the request is invalid.

## 6. Interaction with caching

Changing earlier parts of the history changes the prefix, which resets the cache (see `prompt-caching.md`). So:

- Compact in occasional bigger steps rather than editing the history a little every turn.
- Keep the system prompt and tools stable while you compact the messages.
- Expect one cache write right after a compaction, then reads again.

## 7. Where it fits in agents

- **Agentic loops**: set a budget check inside the loop, before each request.
- **MCP**: every connected server adds tool definitions to the window. Connect only what the task needs (see `model-context-protocol.md`).
- **CI workers**: a run starts with an empty context, so a long job should write progress to files or PR comments instead of relying on memory (see `ci-notes.md`).

## Check yourself

1. Name four things that share the context window.
2. Which usually fills the window fastest in an agent loop, and what is the cheapest fix?
3. Why is it risky to summarize without keeping exact file paths and IDs?
4. When trimming history, what pairing must you not break?
5. Why does compacting the history affect the prompt cache?
6. In the desk analogy, where does the analogy stop matching how the API behaves?

<details>
<summary>Answers</summary>

1. For example: the system prompt, tool definitions, messages, tool results, and the model's output.
2. Tool results, such as large files or logs. The cheapest fix is to trim or paginate the output at the source.
3. A summary can paraphrase or drop them, and the agent may then act on the wrong file or ID.
4. Each `tool_use` block must keep its matching `tool_result`.
5. It changes the prompt prefix, so cached content after the change must be written again.
6. Claude does not clear the desk for you. Your code or tooling decides what is removed, and sending too much can make the request fail.

</details>
