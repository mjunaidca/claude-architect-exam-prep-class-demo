# System Prompts and Prompt Design

How to write the instructions that steer Claude. Earlier notes covered what surrounds the prompt: tools, caching, evals, and safety. This note covers the prompt text itself. `evaluating-prompts-and-agents.md` shows how to measure a prompt; this note shows how to write one worth measuring.

## 1. The big idea

A prompt is a set of instructions plus context for a reader who knows nothing about your situation. Claude is capable, but it cannot see your company, your users, or your intent unless you write them down. Most weak outputs come from missing information, not from a weak model.

> Analogy: a written brief for a skilled contractor who has just arrived and cannot phone you. The better the brief, the better the work. The analogy stops at one point: a contractor would ask questions when something is unclear, but Claude usually guesses, so ambiguity must be removed in advance.

## 2. System prompt vs user message

| Part | Purpose | Changes how often |
| --- | --- | --- |
| System prompt | Role, rules, tone, output format, tool guidance | Rarely (same for every request) |
| User message | The specific task and its input data | Every request |

Put standing rules in the system prompt and per-request data in the user message. This also helps caching, since the stable part forms a reusable prefix (see `prompt-caching.md`).

## 3. What a good system prompt contains

1. **Role and goal**: who Claude is acting as and what success looks like.
2. **Context**: the audience, the product, and why the task exists.
3. **Rules and constraints**: what to do, and what to avoid, with the reason for each.
4. **Output format**: exact shape of the answer (see `structured-outputs-and-validation.md`).
5. **Tool guidance**: when to use each tool and when not to (see `designing-good-tools.md`).

```text
You are a support assistant for Acme Billing, answering customers by chat.
Goal: resolve the question in as few turns as possible.

Rules:
- Only discuss Acme Billing. Politely decline other topics.
- Never state a refund is approved; only a human agent can approve refunds.
  (Approval promises have caused legal disputes.)
- If the account ID is missing, ask for it before calling lookup_account.

Answer in plain text, under 120 words, with no headings.
```

## 4. Techniques that reliably help

| Technique | What to do | Why it works |
| --- | --- | --- |
| Be specific | Say "three bullets, under 15 words each" instead of "short" | Removes guessing |
| Explain the why | Add the reason behind a rule | Lets Claude handle cases the rule did not list |
| Give examples | Include 2-5 varied input/output pairs | Shows the format and tone directly |
| Separate sections | Wrap parts in tags like `<document>` and `<instructions>` | Keeps data apart from instructions |
| Say what to do | "Reply in plain prose" beats "Don't use markdown" | Positive instructions are easier to follow |
| Allow "I don't know" | Say what to do when information is missing | Reduces invented answers |

Example with tags:

```text
<instructions>Summarize the ticket in one sentence.</instructions>
<ticket>
{{ticket_text}}
</ticket>
```

Tags also help with safety: text inside `<ticket>` is data, not commands (see `prompt-injection-and-agent-safety.md`). Tags help but do not make injected text harmless on their own.

## 5. Examples need care

Examples are powerful, and Claude tends to copy them closely.

- Make them varied. If every example is positive, outputs skew positive.
- Make them match the real format exactly, including edge cases.
- Keep them separate from real input, for instance inside `<example>` tags.
- Do not include an example you would not accept as an answer.

## 6. Common mistakes

| Mistake | Result | Fix |
| --- | --- | --- |
| Vague goal ("be helpful") | Inconsistent behavior | State the concrete task and success criteria |
| Long list of "never" rules | Rules conflict or are ignored | Prioritize, give reasons, trim |
| Contradicting instructions | Unpredictable choice between them | Re-read the whole prompt for conflicts |
| Secrets in the prompt | Can leak to users | Keep secrets in code, not text |
| Changing the prompt without testing | Silent regressions | Re-run your eval set after every edit |

## 7. Iterating on a prompt

1. Write a first version with the sections above.
2. Run it on a set of real, varied inputs, including hard ones.
3. Read the failures and find the missing information or unclear sentence.
4. Change one thing, then re-run the same inputs.
5. Keep the version history so you can roll back.

## 8. Links to other topics

- **Evals**: a prompt change is only an improvement if scores rise on a fixed test set (`evaluating-prompts-and-agents.md`).
- **Model choice**: smaller models need clearer, more explicit prompts (`model-selection-and-routing.md`).
- **Subagents**: each subagent needs its own full brief, because it does not see the parent's context (`subagents-and-orchestration.md`).
- **Extended thinking**: give the goal and constraints, and let Claude plan, rather than scripting every step (`extended-thinking.md`).

## Check yourself

1. What belongs in the system prompt and what belongs in the user message?
2. Why is a rule with a reason better than a rule alone?
3. Your examples are all short, polite refusals, and now every answer is a refusal. What went wrong?
4. Rewrite "Don't be wordy" as a specific, positive instruction.
5. Why should you re-run your eval set after editing a prompt?
6. In the contractor-brief analogy, where does the analogy stop matching how Claude behaves?

<details>
<summary>Answers</summary>

1. Standing role, rules, format, and tool guidance go in the system prompt; the specific task and its data go in the user message.
2. The reason lets Claude apply the intent to cases the rule did not list.
3. The examples were not varied, and Claude copied their pattern. Add diverse examples, including helpful answers.
4. For example: "Answer in at most three sentences."
5. A change can fix one case and break others; only a fixed test set shows the net effect.
6. A contractor would ask clarifying questions, while Claude usually guesses, so ambiguity must be removed up front.

</details>
