# Evaluating Prompts and Agents

How to find out whether your prompt, tool setup, or agent actually works. Earlier notes covered building blocks: tool loops, subagents, caching, context management, and structured outputs. Each one is a change you can make, and each change can quietly make results better or worse. This note covers measuring that with an **evaluation** (an "eval").

## 1. The big idea

An **eval** is a fixed set of test cases plus a way of scoring the output for each one. You run the same set before and after every change and compare scores.

> Analogy: a driving test with the same route and the same checklist for every candidate. Because the route stays fixed, a different result means something changed in the driver, not in the test. The analogy stops at one point: a driving test is pass/fail on one day, while an eval is something you re-run after every change, and model output can vary between runs of the same input.

Trying a prompt on two or three examples and saying "looks good" is not an eval. It misses the cases you did not think to try.

## 2. Parts of an eval

| Part | What it is | Example |
| --- | --- | --- |
| Test case | One input, plus what a good result looks like | A support email and the correct category `billing` |
| Dataset | The collection of test cases | 50 emails, including odd and tricky ones |
| Grader | The rule that scores one output | Exact match on the category field |
| Metric | The summary number across all cases | 46 of 50 correct = 92% |
| Baseline | The score of the current version, to beat | 88% with the old prompt |

Good datasets include normal cases, edge cases (empty input, very long input, ambiguous wording), and cases that failed in real use. Add every real failure you find, so it cannot return unnoticed.

## 3. Three kinds of graders

| Grader | How it works | Best for | Watch out for |
| --- | --- | --- | --- |
| Code-based | A program compares output to the expected answer, or checks a schema or test suite | Fixed answers, structured output, code that can be run | Cannot judge open-ended quality |
| Model-based | Another model call scores the output against a rubric | Tone, helpfulness, summaries | The grader can be wrong or biased, so spot-check it against human judgments |
| Human | A person reads and scores | Final checks, building the rubric | Slow and costly, so use it on a sample |

Prefer the cheapest grader that is reliable for the question. If a program can check it, use a program.

A model-based grader needs a clear rubric:

```text
Score the summary from 1 to 5.
5 = covers every key point in the source, adds nothing not in the source.
3 = misses one key point or adds a small unsupported detail.
1 = misses most key points or invents facts.
Reply with JSON: {"score": <1-5>, "reason": "<one sentence>"}
```

Asking for a short reason and a structured score (see `structured-outputs-and-validation.md`) makes the grader easier to check and parse.

## 4. Evaluating agents, not just single answers

An agent takes many steps (see `tool-use-and-agentic-loops.md`), so a correct final answer can hide a bad path. Look at two levels:

| Level | Question | Example check |
| --- | --- | --- |
| Outcome | Did it reach the right end state? | The file was fixed and the tests pass |
| Trajectory | Did it get there sensibly? | Right tools called, no needless repeats, stayed within the step limit |

Also record cost and time per case (tokens, number of tool calls). A change that raises accuracy by one point but triples cost may not be an improvement. Run each case in a clean, isolated environment so one run cannot leave files or state that help or hurt the next.

## 5. Running evals well

1. Change **one thing at a time**, so you know what caused a score change.
2. Keep the dataset and grader fixed while you compare versions.
3. Because outputs vary, run important cases more than once, or treat small differences (a point or two on a small dataset) as noise.
4. Keep some cases you never tune against. If you keep adjusting the prompt until every case passes, the prompt may only fit those cases and fail on new ones. A held-out set shows whether it generalizes.
5. Re-run the eval when the model version changes, not only when your prompt does.
6. Read the failures, not only the score. The pattern in the failures tells you what to fix.

```python
def run_eval(cases, run_system, grade):
    results = []
    for case in cases:
        output = run_system(case["input"])
        results.append(grade(output, case["expected"]))
    return sum(results) / len(results)   # e.g. 0.92

baseline = run_eval(cases, old_prompt_system, grade)
candidate = run_eval(cases, new_prompt_system, grade)
print(baseline, candidate)
```

## 6. Common mistakes

| Mistake | Why it hurts |
| --- | --- |
| Only testing easy examples | The score looks high but real traffic fails |
| Changing prompt and grader together | You cannot tell which change moved the score |
| Trusting a model grader unchecked | It may reward confident-sounding wrong answers |
| Tuning on every case | The prompt overfits and fails on new inputs |
| Looking only at the average | A good average can hide one category that always fails |

## Check yourself

1. In your own words, what are the test case, grader, and metric in an eval?
2. Your classifier must output one of five fixed labels. Which grader type fits, and why?
3. An agent's final answer is correct, but it called the same search tool 30 times. Which level of evaluation reveals that problem?
4. You rewrite a prompt and also swap the grader rubric, and the score rises. Why is that result hard to trust?
5. Why keep a held-out set of cases you never tune against?
6. A model-based grader gives high scores to long answers regardless of correctness. What can you do to check and fix this?

<details>
<summary>Answers</summary>

1. Test case: an input with the expected good result. Grader: the rule that scores one output. Metric: the summary number over all cases.
2. A code-based grader (exact match), because the answer is fixed and a program can check it cheaply and reliably.
3. Trajectory evaluation. The outcome was right, but the path was wasteful.
4. Two things changed at once, so you cannot tell whether the prompt or the easier grading caused the gain.
5. To test whether the prompt generalizes to new inputs instead of only fitting the cases you tuned on.
6. Compare its scores to human scores on a sample, then tighten the rubric (for example, penalize padding and require facts to match the source) and re-check.

</details>
