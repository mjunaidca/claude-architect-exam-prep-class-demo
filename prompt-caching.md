# Prompt Caching

How to stop paying (in time and money) to re-read the same long prompt on every call. `tool-use-and-agentic-loops.md` showed that an agent resends the whole history on every loop turn; this note explains how caching makes that cheap.

## 1. The big idea

Every API call is stateless: you resend the system prompt, tool definitions, and conversation each time. **Prompt caching** lets the API remember the processed *beginning* of a prompt, so the next call that starts with the same content reuses it instead of processing it again.

> Analogy: a bookmark in a long book. If you return to the same book, you resume from the bookmark instead of re-reading from page one. The analogy stops at one point: the bookmark only works for pages that are *identical* from the start. If you change one word early in the book, every bookmark after that word is useless.

Benefits: lower latency and lower cost on repeated prefixes. Caching does not change what Claude answers.

## 2. Prefixes and order

A cache hit needs an **exact match of the prompt prefix**. The prompt is assembled in this order:

```
tools  ->  system  ->  messages
```

So put stable content first and changing content last.

| Content | Changes often? | Where it belongs |
| --- | --- | --- |
| Tool definitions | Rarely | Start (cached first) |
| System instructions, reference docs | Rarely | Next |
| Conversation history | Grows each turn | After the stable parts |
| Current date, user name, the new question | Every call | At the very end |

A common mistake is putting a timestamp at the top of the system prompt. It changes every call, so nothing after it ever hits the cache.

## 3. Marking a cache breakpoint

You add `cache_control` to the last block you want included in the cached prefix.

```python
response = client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=1024,
    system=[
        {"type": "text", "text": "You are a support agent for Acme."},
        {
            "type": "text",
            "text": LONG_POLICY_DOCUMENT,
            "cache_control": {"type": "ephemeral"},  # cache everything up to here
        },
    ],
    messages=[{"role": "user", "content": "Can I return an opened item?"}],
)
```

Notes:

- A request can have several breakpoints (the documented limit is 4), for example one after the tools and one after the system prompt.
- The system checks backward from a breakpoint for earlier matching cache entries, so a growing conversation can keep reading the older part of its own history.
- Very short prefixes are not cached. The minimum length depends on the model, so check the current docs.

## 4. Lifetime and cost

| Item | Detail (check current docs for exact numbers) |
| --- | --- |
| Default lifetime | About 5 minutes, refreshed each time the entry is used |
| Longer option | 1 hour, at a higher write price |
| Cache write | Costs more than a normal input token (a premium over base price) |
| Cache read | Costs a small fraction of the base input price |

Caching pays off when the same prefix is reused several times within its lifetime. For a prompt used once, the write premium is a small net loss.

## 5. Checking that it works

The response `usage` tells you what happened:

| Field | Meaning |
| --- | --- |
| `cache_creation_input_tokens` | Tokens written to the cache on this call |
| `cache_read_input_tokens` | Tokens served from the cache |
| `input_tokens` | Tokens after the last breakpoint, processed normally |

If `cache_read_input_tokens` stays at 0 on repeated calls, something in the prefix is changing (or it is below the minimum length, or the entry expired).

## 6. What invalidates the cache

Any change at or before a breakpoint breaks the match from that point on.

- Editing the system prompt text.
- Adding, removing, or reordering tools (tools come first, so this invalidates everything after).
- Changing a setting that is part of the cached prefix, such as switching models.
- Waiting longer than the lifetime without reusing the entry.

Changing text *after* the last breakpoint is fine; only that tail is reprocessed.

## 7. Where it fits in agents

- **Agentic loops**: each turn appends a tool call and result to the history. Caching the earlier turns means only the newest messages are processed fresh.
- **MCP**: connecting a server adds its tools to the `tools` list. Changing the connected servers mid-session changes the prefix and resets the cache (see `model-context-protocol.md`).
- **CI workers**: a fresh runner keeps nothing locally, but the API-side cache still works if the prefix is identical and the calls happen within the lifetime.

## Check yourself

1. Why must stable content go before changing content in a prompt?
2. Your cache reads are always 0, and the system prompt begins with "Today is <current time>". What is wrong and how do you fix it?
3. Which fields in `usage` show a cache write versus a cache read?
4. Name two changes that invalidate a cached prefix.
5. A prompt is sent once per day. Is a 5-minute cache likely to help? Why or why not?
6. In the bookmark analogy, where does the analogy stop matching how caching behaves?

<details>
<summary>Answers</summary>

1. The cache matches an exact prefix, so any change early in the prompt invalidates everything after it.
2. The timestamp changes every call, so the prefix never matches. Move it to the end (after the last breakpoint).
3. `cache_creation_input_tokens` is a write; `cache_read_input_tokens` is a read.
4. For example: editing the system prompt, or adding/removing/reordering tools (also switching models).
5. No. The entry expires long before the next call, so you would pay the write premium with no reads.
6. The bookmark works only if everything before it is identical; a changed word early on invalidates all later content.

</details>
