# Streaming Responses

How to receive Claude's answer piece by piece instead of waiting for the whole thing. `api-errors-rate-limits-and-retries.md` covered what to do when a request fails, and `tool-use-and-agentic-loops.md` covered the loop that calls tools. This note covers how both change when the reply arrives as a stream.

## 1. The big idea

Without streaming, you send a request and receive nothing until the full reply is finished. With **streaming**, the API sends the reply in small events as it is generated, using server-sent events (SSE) over one open HTTP connection.

> Analogy: a live broadcast versus a recorded video. With a recording you wait until the whole file is ready. With a broadcast you see each moment as it happens. The analogy stops at one point: a broadcast cannot be rewound, and neither can a stream. If the connection drops midway, you hold only a partial reply, and you must decide what to do with it.

Streaming does not make the model faster or cheaper. The total work is the same. It improves **perceived** speed because the user sees the first words quickly, and it avoids long idle connections for big outputs.

## 2. When to stream

| Situation | Stream? | Why |
| --- | --- | --- |
| Chat UI where a person reads the answer | Yes | Text appears right away |
| Long outputs (large `max_tokens`) | Yes | Long non-streamed requests risk network timeouts, and SDKs may require streaming for very large limits |
| Background job that only needs the final result | Often no | A plain request is simpler |
| Bulk offline work | No, use batches | See `message-batches-and-async-processing.md` |

## 3. The shape of a stream

A stream is an ordered list of events. The reply is built from **content blocks** (text, tool calls, thinking), and each block has a start, some deltas, and a stop.

| Event | Meaning |
| --- | --- |
| `message_start` | The reply began; carries the message shell and input usage |
| `content_block_start` | A new block begins (for example text or `tool_use`) |
| `content_block_delta` | A small piece of that block |
| `content_block_stop` | That block is complete |
| `message_delta` | Message-level changes, such as `stop_reason` and final output usage |
| `message_stop` | The reply is complete |

The API may also send `ping` events to keep the connection alive, and can send an `error` event mid-stream. Code should ignore event types it does not recognize, because new ones can be added.

## 4. Using the SDK

The SDKs hide the raw events and give you helpers.

```python
with client.messages.stream(
    model="claude-sonnet-5-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Explain SSE briefly."}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)   # show text as it arrives

    final = stream.get_final_message()    # the complete, assembled message
```

Use the final assembled message for logging, saving, and checking `stop_reason`. Do not rebuild it yourself from printed text.

## 5. Streaming and tool calls

Tool arguments arrive as text fragments of JSON in `input_json_delta` events. A fragment such as `{"city": "Pa` is **not valid JSON** on its own.

```text
content_block_start  -> tool_use, name="get_weather"
content_block_delta  -> partial_json: '{"city": '
content_block_delta  -> partial_json: '"Paris"}'
content_block_stop   -> now the input is complete and parseable
```

Rules to follow:

- Join the fragments and parse them only after `content_block_stop`.
- Do not run a tool until its block is complete. Running on a half-built argument is a bug and can be a safety problem (see `human-in-the-loop-and-approval-gates.md`).
- After the stream ends, check `stop_reason`. If it is `tool_use`, run the tools and send back `tool_result` blocks exactly as in a non-streamed loop.

## 6. Streaming and thinking

With extended thinking on (see `extended-thinking.md`), thinking blocks stream before the answer, as their own deltas. You can show them or hide them in your UI, but when you continue a tool loop you must return the thinking blocks unchanged, same as without streaming.

## 7. Failures mid-stream

Errors can happen before the first event (a normal HTTP error) or after output has started (an `error` event or a dropped connection).

| Failure | Typical handling |
| --- | --- |
| Error before streaming starts | Normal retry with backoff, as in `api-errors-rate-limits-and-retries.md` |
| Connection drops midway | The partial text is incomplete. Retry the whole request, or continue from the partial text only if you design for it |
| Overloaded `error` event mid-stream | Treat as retryable, but discard or clearly mark the partial output |
| User cancels | Close the stream; you pay for tokens generated so far |

Never show a partial tool call or partial structured output as if it were final. For machine-readable output, validate only the completed result (see `structured-outputs-and-validation.md`).

## 8. Where it fits in agents

- **Chat agents**: stream text to the user, but run the loop on the final message.
- **Observability**: record time to first token and total time as separate metrics (see `observability-and-tracing-for-agents.md`).
- **Caching**: streaming works with prompt caching, and usage numbers arrive in `message_start` and `message_delta` (see `prompt-caching.md`).
- **Subagents**: a parent usually needs only the child's final report, so streaming the child is rarely useful (see `subagents-and-orchestration.md`).

## Check yourself

1. Does streaming reduce the total time or cost of a request? What does it improve?
2. Name the order of events for one content block.
3. Why can't you parse a tool call's JSON as soon as the first `input_json_delta` arrives?
4. Where do you find the `stop_reason` and complete message when streaming?
5. A stream drops after half the text. Why is it unsafe to treat that text as the final answer?
6. In the broadcast analogy, where does the analogy stop matching reality?

<details>
<summary>Answers</summary>

1. No. Total work is the same. It improves perceived speed (first words appear quickly) and avoids long idle connections.
2. `content_block_start`, one or more `content_block_delta`, then `content_block_stop`.
3. The fragments are pieces of one JSON string, so the early ones are incomplete and invalid. Parse after `content_block_stop`.
4. In the final assembled message (for example `get_final_message()`), with `stop_reason` arriving in `message_delta`.
5. It is incomplete and may cut off mid-sentence or mid-instruction. Retry or clearly mark it as partial.
6. A broadcast cannot be rewound, and a stream cannot either, but unlike a broadcast a dropped stream leaves you holding a partial reply that you must handle in code.

</details>
