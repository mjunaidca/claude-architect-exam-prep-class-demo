# Retrieval-Augmented Generation (RAG) and Citations

How to let Claude answer questions from documents it was never trained on. `agent-memory-and-state-persistence.md` and `model-context-protocol.md` both pointed at "retrieval" for large document sets. This note explains what that means, where it breaks, and how to make answers checkable.

## 1. The big idea

Claude only knows its training data plus what is in the prompt. Your company handbook, last week's tickets, and a new API version are in neither. **Retrieval-augmented generation (RAG)** fixes this in three steps: **find** the relevant passages, **place** them in the prompt, and **ask** Claude to answer from them.

> Analogy: an open-book exam. The student does not memorize the textbook; they look up the right pages, then write the answer. The analogy stops at one point: a student knows when they could not find the page, while Claude may still write a confident answer unless you tell it what to do when the pages are missing.

## 2. The pipeline

| Stage | What happens | Typical choice |
| --- | --- | --- |
| Prepare (offline) | Split documents into chunks, index them | Chunks of a few hundred tokens, with titles kept |
| Retrieve | Turn the question into a search, fetch top matches | Keyword search, embedding (meaning-based) search, or both |
| Rerank (optional) | Reorder matches so the best come first | A cheaper model or scoring function |
| Generate | Put the chunks in the prompt and ask the question | Claude, with grounding instructions |

Claude does not provide an embeddings model of its own. You choose a retrieval system, and Claude does the reading and writing.

## 3. Two ways to retrieve

| Style | How it works | Good at | Weak at |
| --- | --- | --- | --- |
| Keyword (e.g. BM25) | Matches exact words | Names, error codes, IDs | Different wording of the same idea |
| Embedding | Matches meaning via number vectors | Paraphrases, vague questions | Exact strings, rare terms |

Combining both ("hybrid" search) is a common, sturdy default.

## 4. RAG as a tool

Instead of retrieving once up front, expose search as a tool and let the agent loop decide (`tool-use-and-agentic-loops.md`, `designing-good-tools.md`).

```json
{
  "name": "search_docs",
  "description": "Search the internal handbook. Returns up to 5 passages with doc IDs. Call again with a different query if results look unrelated.",
  "input_schema": {
    "type": "object",
    "properties": { "query": { "type": "string" } },
    "required": ["query"]
  }
}
```

| Approach | Use when |
| --- | --- |
| Retrieve once, then answer | Simple single-question lookups; lower cost and latency |
| Search as a tool | Questions need several lookups, or a first search may miss |

## 5. Building the prompt

Wrap each passage with its source, and put the question after the documents. Fence the documents clearly as data.

```text
<documents>
  <document id="hr-12"><source>Leave policy v3</source>
    <content>Employees accrue 1.5 days per month...</content>
  </document>
</documents>

Answer using only the documents above. Quote the supporting text and give the
document id. If the documents do not contain the answer, say you cannot find it.
```

Long reference material is stable, so it pairs well with `prompt-caching.md`. Retrieved text is untrusted input, so treat instructions inside it as data (`prompt-injection-and-agent-safety.md`).

## 6. Citations

A citation ties each claim to the passage it came from, so a reader can verify it. You can do this by prompting for quotes and IDs, or by using the API's citations feature, which returns structured pointers to the source text instead of relying on the model to format them. Check the current API documentation for exact parameters.

Whichever way, **verify in code** that a cited ID exists and the quote really appears in the source. A citation that was never checked is decoration (`structured-outputs-and-validation.md`).

## 7. Common failures

| Failure | Cause | Fix |
| --- | --- | --- |
| Right document never found | Bad chunking or query wording | Overlap chunks, rewrite the query, use hybrid search |
| Answer ignores the passages | Too many or buried passages | Fewer, better-ranked chunks; question placed last |
| Confident answer with no evidence | No "I don't know" instruction | Require a quote, allow a refusal |
| Stale answer | Index not refreshed | Re-index on change; show document dates |
| Leaked private document | Search ignores who is asking | Filter by the user's permissions *before* retrieval |
| Injected instructions in a page | Retrieved text trusted | Fence as data; limit what tools the agent has |

## 8. Is RAG the right tool?

| Situation | Better fit |
| --- | --- |
| Small corpus that fits in the window | Put it all in the prompt, cached |
| Need a consistent output style | Prompting or examples, not retrieval |
| Facts about this user across sessions | Memory (`agent-memory-and-state-persistence.md`) |
| Live systems and APIs | Tools or MCP servers |
| Large, changing corpus | RAG |

Measure retrieval and answers separately: did the right passage appear in the top results, and was the answer faithful to it? See `evaluating-prompts-and-agents.md`.

## Check yourself

1. What are the three steps of RAG, and what does Claude do in each?
2. Where does the open-book analogy mislead?
3. A user searches for error code `E-4471`. Keyword or embedding search, and why?
4. When would you make search a tool instead of retrieving once?
5. Why must permission filtering happen before retrieval, not after Claude answers?
6. A response cites document `hr-99`, which was not in the prompt. What should your code do?
7. Your answers are wrong but the right passage was retrieved. Which stage do you investigate?
