# API Errors, Rate Limits, and Retries

Every agent that calls the Claude API will eventually get an error: a busy server, a dropped connection, a rate limit. This note builds on `tool-use-and-agentic-loops.md` (an agent makes many API calls per task), `ci-notes.md` (unattended runs cannot be babysat), and `prompt-caching.md` (cost and latency). It covers which errors to retry, how to wait, and how to keep a retry from doing damage.

## 1. The big idea

Errors fall into two groups: **temporary** (try again later and it may work) and **permanent** (the request itself is wrong, so trying again just fails again). A reliable agent retries the first group carefully and fixes or reports the second.

> Analogy: you call a restaurant and hear a busy signal. Redialing a few minutes later makes sense. But if you dialed the wrong number, redialing a hundred times never helps. The analogy has a limit: a busy signal costs nothing, while a repeated API request can cost money or repeat a side effect, so retries need rules.

## 2. Common HTTP errors

| Status | Error type | Meaning | Retry? |
|--------|-----------|---------|--------|
| 400 | `invalid_request_error` | Malformed request or bad parameters | No, fix the request |
| 401 | `authentication_error` | Missing or invalid API key | No, fix the key |
| 403 | `permission_error` | Key is not allowed to do this | No |
| 404 | `not_found_error` | Resource (such as a model name) not found | No |
| 413 | `request_too_large` | Request body over the size limit | No, shrink it (see `context-window-management.md`) |
| 429 | `rate_limit_error` | You are sending too much too fast | Yes, after waiting |
| 500 | `api_error` | Unexpected problem on the server side | Yes |
| 529 | `overloaded_error` | The service is temporarily overloaded | Yes |

Rule of thumb: 4xx errors (except 429) are your problem; 429 and 5xx are usually temporary. Check the current API docs for the full list.

## 3. Rate limits

Limits are applied per organization and model tier, typically on requests per minute and tokens per minute (input and output counted separately). When you exceed one you get a 429. The response often includes a `retry-after` header saying how many seconds to wait; honor it when present.

Ways to stay under limits:

- Cache stable prefixes (`prompt-caching.md`); cached reads usually count less against input limits than fresh input, so check the docs for your model.
- Cap how many agents or CI jobs run in parallel (`subagents-and-orchestration.md`).
- Trim what you send; smaller contexts use fewer tokens per minute.
- Use the Batch API for work nobody is waiting on (section 6).

## 4. Retrying with exponential backoff and jitter

**Exponential backoff** means waiting longer after each failure: 1s, 2s, 4s, 8s. **Jitter** adds a random bit to each wait so many clients do not all retry at the same instant.

```python
import random, time

def call_with_retry(send, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            return send()
        except TemporaryError as err:          # 429, 500, 529, network timeout
            if attempt == max_attempts - 1:
                raise                          # give up, surface the error
            wait = err.retry_after or min(2 ** attempt, 30)
            time.sleep(wait + random.uniform(0, 1))
```

> Analogy: a crowd leaving a stadium. If everyone leaves through one gate at the final whistle, it jams; staggering departures spreads the load. The analogy has a limit: stadium-goers can see each other, but your clients cannot, which is why randomness is used instead of coordination.

Official Anthropic SDKs already retry some errors automatically with backoff (a small default number of attempts that you can configure). Before writing your own loop, check whether the SDK does this, so you do not stack two retry layers and multiply the waits.

## 5. Making retries safe

- **Always set a maximum** on attempts and total time. An unbounded retry loop in CI can hang a job for hours.
- **Set timeouts.** A request that never returns is worse than one that fails. For long outputs, use streaming so the connection stays alive.
- **Watch side effects.** If a call failed after the agent already ran a tool (created a ticket, pushed a commit), replaying the whole step may repeat the action. Make tools **idempotent** (running twice has the same effect as once), for example by checking "does this branch already exist?" first.
- **Do not retry blindly on a reply that is wrong.** A `max_tokens` cutoff or schema failure is not a transient error; see `structured-outputs-and-validation.md` for validate-then-retry.
- **Log every retry** with status, attempt number, and wait time, so you can see when a service is struggling.
- **Fail clearly.** After the final attempt, return a clear error or leave the job marked failed. Do not let the agent continue as if the call had succeeded.

## 6. Work that can wait: the Message Batches API

For jobs such as scoring 10,000 documents overnight (see `evaluating-prompts-and-agents.md`), the **Message Batches API** lets you submit many requests at once and collect results later. Batches are typically discounted (around half price at the time of writing) and finish within a window of up to 24 hours, but they are not instant. Each request in a batch succeeds or fails on its own, so after the batch ends, collect the failed ones and resubmit just those.

| Need | Use |
|------|-----|
| User waiting for an answer | Normal (ideally streaming) calls |
| Large offline job, cost matters | Batch API |
| Agent loop with many dependent steps | Normal calls; later steps need earlier results |

## 7. Common mistakes

- Retrying 400 or 401 errors forever.
- Retrying instantly with no backoff, which makes an overload worse.
- No cap on attempts or total time.
- Stacking SDK retries on top of your own loop without realizing it.
- Re-running a whole agent step that already caused a side effect.
- Using real-time calls for large jobs that could be batched.

## 8. Check yourself

1. Which of these should be retried: 400, 401, 429, 529? Why?
2. What is the purpose of jitter in a backoff schedule?
3. A response includes `retry-after: 20`. What should your code do?
4. An agent created a pull request, then its next API call hit a 500. Why can naive replay be harmful, and how do you guard against it?
5. When is the Batch API a poor choice?

<details>
<summary>Answers</summary>

1. Retry 429 and 529: they are temporary conditions. Do not retry 400 or 401: the request or key is wrong and will fail the same way.
2. It randomizes waits so many clients do not retry at the same moment and overload the service again.
3. Wait at least 20 seconds (plus a little jitter) before retrying.
4. Replaying the step might open a second pull request. Make the action idempotent: check whether the PR or branch already exists before creating it.
5. When a user is waiting, or when each step depends on the previous result, as in an agent loop. Batches are for offline, independent requests.

</details>
