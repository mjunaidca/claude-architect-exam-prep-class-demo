# Model Selection and Routing

Picking the right model for each job. `evaluating-prompts-and-agents.md` showed how to measure quality, and `prompt-caching.md` and `extended-thinking.md` showed levers that change cost and latency. This note covers the biggest lever of all: which model answers each request.

> Model names, prices, and limits change often. The examples use model IDs from this repo's other notes; check the current docs for the up-to-date list and pricing.

## 1. The big idea

Claude comes in a family of models. Larger models are usually more capable but cost more and respond more slowly. Smaller models are cheaper and faster but may struggle on hard tasks. There is no single best model. There is a best model **for each task**, and you find it by testing.

> Analogy: a hospital does not send every patient to the top surgeon. A nurse handles routine checks, a general doctor handles common illnesses, and a specialist handles hard cases. The analogy stops in one place: a hospital triages by looking at the patient, while an app must often guess difficulty from the request alone, so you need a plan for wrong guesses (section 4).

## 2. The three trade-offs

| Factor | Larger model | Smaller model |
| --- | --- | --- |
| Quality on hard tasks | Usually higher | Usually lower |
| Cost per token | Higher | Lower |
| Latency | Slower | Faster |

Cost adds up through volume. A step that runs 100,000 times a day matters far more than one that runs ten times.

## 3. How to choose: start from evals

1. Write an eval set for the task (see `evaluating-prompts-and-agents.md`).
2. Run it on the most capable model. This shows what is possible.
3. Run the same set on smaller models.
4. Pick the **cheapest model that meets your quality bar** on your own data.
5. Re-run the evals when a new model ships or your prompt changes.

Do not choose by reputation or by one demo. A small model that passes your evals is the right choice even if a bigger one is "better" in general.

| Task | Usual starting point |
| --- | --- |
| Classifying, extracting, routing, formatting | Smaller, fast model |
| Everyday chat and summarizing | Mid-size model |
| Multi-step planning, hard code, ambiguous requests | Most capable model, maybe with thinking |
| Large offline jobs | Cheapest model that passes, via the Batch API |

## 4. Routing patterns

**Routing** means sending each request to a different model depending on the task.

```python
def pick_model(task_type: str) -> str:
    if task_type in {"classify", "extract"}:
        return "claude-haiku-5-5"
    if task_type in {"summarize", "chat"}:
        return "claude-sonnet-5-5"
    return "claude-opus-5-5"          # hard or unknown work

response = client.messages.create(
    model=pick_model(task.type),
    max_tokens=1024,
    messages=task.messages,
)
```

Common patterns:

- **Rule-based routing.** You decide by task type, as above. Simple, predictable, easy to test.
- **Classifier routing.** A small model first labels how hard the request is, then you pick the model. Adds one cheap call.
- **Escalation (fallback up).** Try the cheap model first. If a validator fails or confidence is low (see `structured-outputs-and-validation.md`), retry on a stronger model.
- **Orchestrator and workers.** A strong model plans and a smaller model does the many simple subtasks (see `subagents-and-orchestration.md`).

Default unknown cases to the stronger model. A wasted few cents is better than a wrong answer on a hard request.

## 5. Things that change when you switch models

- **Prompts are not fully portable.** A prompt tuned for one model can behave differently on another. Re-run evals after every switch.
- **Caches are per model.** A cached prefix on one model does not help another (see `prompt-caching.md`), so spreading similar traffic over many models can lower cache hits.
- **Features differ.** Context size, thinking support, and tool behavior can vary by model.
- **Limits differ.** Rate limits are often set per model (see `api-errors-rate-limits-and-retries.md`). A fallback model can sometimes absorb overflow, but test that its output is acceptable.
- **Pin versions.** Use a specific model ID in production, not an alias that may move, so behavior does not change under you.

## 6. Common mistakes

- Using the biggest model everywhere "to be safe" and overpaying.
- Using the smallest model everywhere without evals and shipping silent quality drops.
- Routing by guesswork with no way to measure whether routing helps.
- Never revisiting the choice when newer models are released.
- Escalating without a validator, so nothing decides when the cheap answer is bad.

## Check yourself

1. Why is there no single best model for every task?
2. What is the process for choosing a model for a new task?
3. Describe the escalation pattern. What must exist for it to work?
4. Why should you re-run evals after changing models, even with the same prompt?
5. A request's difficulty is unclear. Which model should it go to by default, and why?

### Answers

1. Models trade quality against cost and speed, and tasks differ in how much quality they need. The best choice is the cheapest model that meets the bar for that task.
2. Build evals, run them on the most capable model, then on smaller ones, and pick the cheapest that meets your quality bar. Repeat when models or prompts change.
3. Try a cheap model first, and retry on a stronger one when a check fails. You need a validator or confidence signal to decide when the first answer is not good enough.
4. Prompts behave differently across models, so quality can rise or fall. Only evals show which.
5. The stronger model. A small extra cost is better than a wrong answer on a hard request.
