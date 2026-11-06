# Message Batches and Async Processing

How to run thousands of requests cheaply when nobody is waiting for the answer. `model-selection-and-routing.md` mentioned the Batch API for large offline jobs; `api-errors-rate-limits-and-retries.md` covered handling failures one call at a time. This note shows how to submit work in bulk and handle partial failure.

## 1. The big idea

A normal call is **synchronous**: you send a request and wait for the response. The **Message Batches API** is **asynchronous**: you submit many requests at once, get a batch ID back immediately, and collect the results later.

> Analogy: a dry cleaner. Dropping off one shirt and waiting at the counter is a normal call. Dropping off a bag of 500 shirts and picking it up tomorrow at a lower price is a batch. The analogy stops at one point: a dry cleaner hands shirts back in the order you gave them, but batch results can come back in any order, so you must match them by ID.

Trade-off: a discount (documented at roughly half the normal price; check current docs) in exchange for no latency promise. Most batches finish within hours, but there is a maximum processing window (documented as 24 hours), after which unfinished requests expire.

## 2. When to use it

| Good fit | Poor fit |
| --- | --- |
| Nightly classification of support tickets | A chatbot reply a user is waiting for |
| Running an eval suite over 5,000 test cases | A step inside an agent loop that needs the result to decide the next step |
| Summarizing an archive of documents | Anything with a strict response-time target |
| Bulk data extraction into a schema | One-off requests (no benefit from batching) |

Rule of thumb: if a human or another step is blocked waiting, use a normal call. If the work can wait, batch it.

Agent loops are a poor fit because each turn depends on the previous tool result. You can still batch *many independent single-turn tasks*.

## 3. Submitting a batch

Each item has a `custom_id` you choose and the same `params` you would send to a normal Messages call.

```python
batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": f"ticket-{t.id}",
            "params": {
                "model": "claude-haiku-5-5",
                "max_tokens": 200,
                "messages": [{"role": "user", "content": f"Classify: {t.text}"}],
            },
        }
        for t in tickets
    ]
)
print(batch.id, batch.processing_status)  # "in_progress"
```

Notes:

- There are limits on requests and bytes per batch (check current docs); split bigger jobs into several batches.
- Each request is validated and processed independently. One bad request does not sink the others.
- Features like prompt caching generally work inside batches, but caching is best-effort there because requests run in no fixed order.

## 4. Collecting results

Poll the batch until `processing_status` is `ended`, then stream the results.

```python
while client.messages.batches.retrieve(batch.id).processing_status != "ended":
    time.sleep(60)          # batches take a while; don't poll every second

for entry in client.messages.batches.results(batch.id):
    if entry.result.type == "succeeded":
        save(entry.custom_id, entry.result.message)
    else:
        failed.append(entry.custom_id)
```

Each entry has one of these result types:

| `result.type` | Meaning | What to do |
| --- | --- | --- |
| `succeeded` | A normal message came back | Use it (still check `stop_reason`) |
| `errored` | Invalid request or server error | Fix the request if invalid; retry if transient |
| `canceled` | You canceled before it ran | Resubmit if still needed |
| `expired` | Not finished within the window | Resubmit in a new batch |

## 5. Handling partial failure

A batch ending does not mean every request worked. The safe pattern:

1. Key every request with a stable, meaningful `custom_id` (a ticket ID, not `1, 2, 3` that shifts between runs).
2. After the batch ends, split results into succeeded and not succeeded.
3. Validate succeeded outputs as you would any model output (see `structured-outputs-and-validation.md`).
4. Collect the failures and `expired` items into a **new batch** and resubmit, rather than looping on single calls.
5. Make saving results **idempotent** (keyed on `custom_id`), so rerunning never double-counts.

Results are kept for a limited time (documented as 29 days), so download and store them rather than treating the API as your database.

## 6. Batch versus other cost tools

| Tool | Saves | Cost to you |
| --- | --- | --- |
| Batch API | Money (discount) | Latency (up to hours) |
| Prompt caching | Money and latency on repeated prefixes | Careful prompt ordering |
| Smaller model | Money and latency | Possible quality drop; verify with evals |

They can stack: a cheaper model, a cached shared prefix, and a batch for a nightly job.

## 7. Check yourself

1. Why must you use `custom_id` to match results to requests?
2. Your agent needs the result of one call to choose its next tool. Is batching appropriate? Why or why not?
3. A batch ends with 4,900 `succeeded` and 100 `expired`. What should you do with the 100?
4. A teammate polls batch status once per second. What is wrong, and what would you do instead?
5. Name one situation where a batch is cheaper but the normal API is still the right choice.

<details>
<summary>Answers</summary>

1. Results may arrive in any order, not the order submitted.
2. No. The next step depends on the result, and batches have no latency guarantee. Batching suits independent tasks.
3. Collect them and submit them in a new batch; they were not processed, so nothing is lost.
4. Batches take minutes to hours, so polling that fast wastes calls. Poll about once a minute (or back off) and stop at `ended`.
5. Any user-facing request: a user cannot wait hours, so the discount is not worth the delay.

</details>
