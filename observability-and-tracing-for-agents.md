# Observability and Tracing for Agents

How to see what an agent actually did after it has run. `evaluating-prompts-and-agents.md` measured quality on a fixed test set before release; `api-errors-rate-limits-and-retries.md` handled failures as they happen. This note covers the third piece: recording enough detail in production that you can explain any run, good or bad, after the fact.

## 1. The big idea

An agent makes many decisions you never see: which tool to call, what the tool returned, when it stopped. If you only keep the final answer, you cannot tell *why* a run went wrong. **Observability** means logging each step so you can reconstruct the run. A **trace** is the full record of one run, made of **spans** (one per step, such as a model call or a tool call).

> Analogy: a flight data recorder. After an incident you do not guess what happened; you replay the recorded readings. The analogy stops at one point: a flight recorder is only read after a crash, while agent traces are also used daily to spot slow, costly, or drifting behavior before anything breaks.

## 2. What to record per step

| Step type | Record | Why |
| --- | --- | --- |
| Model call | Model name, prompt version, `stop_reason`, token counts, latency | Cost, speed, and truncation (`max_tokens`) debugging |
| Tool call | Tool name, input, output (or error), latency | Finds bad arguments and slow tools |
| Retry | Attempt number, error type, wait time | Shows rate-limit pressure |
| Run as a whole | Run ID, user or task ID, total turns, total cost, final outcome | Lets you find and compare runs |

Give every run one **run ID** and attach it to every span. Without a shared ID, you cannot stitch the steps back together.

## 3. A minimal trace

```python
import time, uuid, json

def traced_call(run_id, step, fn, **meta):
    start = time.time()
    status = "ok"
    try:
        return fn()
    except Exception as e:
        status = f"error:{type(e).__name__}"
        raise
    finally:
        print(json.dumps({          # one structured line per span
            "run_id": run_id, "step": step, "status": status,
            "ms": round((time.time() - start) * 1000), **meta,
        }))

run_id = str(uuid.uuid4())
resp = traced_call(run_id, "model_call", lambda: client.messages.create(...),
                   model="claude-sonnet-5-5", prompt_version="v12")
```

Log **structured** lines (JSON), not free text, so you can filter and total them later.

## 4. Metrics worth watching

| Metric | A rising value suggests |
| --- | --- |
| Turns per run | The agent is looping or confused |
| Tokens / cost per run | Context bloat (see `context-window-management.md`) |
| Tool error rate | A broken tool or poor tool descriptions |
| `stop_reason: max_tokens` rate | Output limit set too low |
| 429 / overloaded rate | You need backoff or more capacity |
| Cache hit rate (falling) | The prompt prefix changed (see `prompt-caching.md`) |
| Human-rejection rate | Agent proposals are poor (see `human-in-the-loop-and-approval-gates.md`) |

Watch **percentiles** (p50, p95), not just averages. An average hides the few runs that took 40 turns.

## 5. Protect what you log

Traces contain prompts and tool outputs, which may hold personal data, secrets, or injected text.

- **Redact** secrets and personal data before writing logs.
- Restrict who can read traces; they are as sensitive as the data they contain.
- Set a **retention** limit and delete old traces.
- Treat logged content as untrusted data when you view or replay it (see `prompt-injection-and-agent-safety.md`).
- Do not log credentials "just in case".

## 6. Using traces to improve

1. A metric alarms (for example, turns per run doubled).
2. Filter traces to the bad runs and read a few end to end.
3. Name the cause: bad tool output, vague prompt, missing context.
4. Turn the failing case into a new eval test case.
5. Fix, rerun the eval, and compare to the baseline.

Observability finds the problems; evals prove the fix works and keep it fixed.

## 7. Check yourself

1. Why does every span need the same run ID?
2. A run cost ten times the average. Which two fields would you look at first?
3. Why is p95 turns-per-run more useful than the average?
4. Name two things to do before storing traces that contain user prompts.
5. How do traces and evals work together?

<details>
<summary>Answers</summary>

1. It lets you stitch separate model and tool steps back into one run.
2. Total turns and token counts per model call; they show a loop or context bloat.
3. Averages hide rare runaway runs; p95 exposes them.
4. Redact secrets and personal data, and restrict access and set a retention limit.
5. Traces reveal real failures; each one becomes an eval case, and the eval confirms the fix and prevents regressions.

</details>
