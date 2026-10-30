# Structured Outputs and Validation

How to get an agent to return data your code can rely on. `tool-use-and-agentic-loops.md` showed that tool calls carry JSON arguments, and `subagents-and-orchestration.md` showed that agents pass results to each other. When a program, not a person, reads those results, free-form prose is fragile. This note covers asking for a fixed shape and checking it.

## 1. The big idea

A **structured output** is a model response that follows a format you define, usually JSON that matches a **schema** (a description of the fields, their types, and which are required). **Validation** is your code checking the response against that schema before using it.

> Analogy: a paper form with labeled boxes instead of a blank page. People fill in the "date" box with a date, so the clerk can process it quickly. The analogy stops at one point: a form box can stop someone writing letters in a number field, but a model's output is only guided, not guaranteed, unless your setup enforces the schema. Even a correctly shaped answer can contain wrong content, so you still check it.

## 2. Ways to get structure

| Approach | How it works | Strength |
| --- | --- | --- |
| Ask in the prompt | "Reply with JSON with keys `title` and `severity`" | Simple, but the model may add extra text or drift from the format |
| Tool with an input schema | Define a tool whose arguments are the fields you want; the model "calls" it | The schema is explicit and arguments arrive as JSON |
| Schema-enforced output | The API constrains the response to a supplied schema, if the feature is available for your model | Strongest format guarantee; check current docs for support and limits |
| Examples in the prompt | Show one or two sample outputs | Helps with style, guarantees nothing |

Using a tool purely to collect structured data is a common pattern: you never run a real function, you just read the arguments.

```python
record_issue = {
    "name": "record_issue",
    "description": "Record one issue found in the code review.",
    "input_schema": {
        "type": "object",
        "properties": {
            "file": {"type": "string"},
            "severity": {"type": "string", "enum": ["low", "medium", "high"]},
            "summary": {"type": "string"},
        },
        "required": ["file", "severity", "summary"],
    },
}
```

An `enum` limits a field to listed values, which removes many spelling and wording variations.

## 3. Syntax versus meaning

Two different things can go wrong:

| Problem | Example | Caught by |
| --- | --- | --- |
| Wrong shape (syntax) | Missing field, text instead of a number, invalid JSON | Schema validation |
| Wrong content (semantics) | `file` names a path that does not exist | Your own checks, tests, or a verifier agent |

A schema proves the form is filled in correctly. It does not prove the answers are true.

## 4. Validate, then retry

Treat model output as untrusted input, like data from a web form.

```python
import json
from jsonschema import validate, ValidationError

def get_issue(prompt, schema, max_tries=3):
    messages = [{"role": "user", "content": prompt}]
    for _ in range(max_tries):
        text = call_model(messages)
        try:
            data = json.loads(text)
            validate(data, schema)
            return data
        except (json.JSONDecodeError, ValidationError) as err:
            messages.append({"role": "assistant", "content": text})
            messages.append({"role": "user",
                             "content": f"Invalid: {err}. Return corrected JSON only."})
    raise RuntimeError("no valid output after retries")
```

Key points:

1. Parse and validate before any other code touches the data.
2. Feed the specific error back so the model can fix it.
3. Cap the retries, as with any loop, and decide what happens on failure (fail loudly, use a default, or ask a human).

## 5. Designing good schemas

1. **Keep it small.** Ask only for fields you will use.
2. **Use enums** for categories instead of free text.
3. **Allow "unknown".** Make a field optional or nullable so the model is not forced to invent a value.
4. **Describe each field.** Short descriptions in the schema act as instructions.
5. **Order for reasoning.** If you want an explanation, put a `reasoning` field before the final `verdict` field so the model writes its thinking first.
6. **Version it.** When agents pass data to each other, a changed schema can break the receiver silently.

## 6. Pitfalls

1. **Truncation.** If the response hits `max_tokens`, the JSON can be cut off and fail to parse. Check `stop_reason` and leave enough room.
2. **Extra text.** Prose or code fences around JSON break a naive `json.loads`. Enforce the schema or strip carefully.
3. **Trusting values.** Never pass a model-supplied field straight into a shell command, SQL query, or file path. Validate and sanitize it as you would user input.
4. **Injection via content.** Data extracted from untrusted pages may contain instructions or odd values; it is still data.
5. **Overly strict schemas.** Rejecting every small deviation can cause endless retries. Decide what is truly required.

## Check yourself

1. What is the difference between a schema and validation?
2. Why might you define a tool you never actually run?
3. A response parses as valid JSON and matches the schema, but the `file` it names does not exist. Which kind of error is this, and what catches it?
4. Why should a retry message include the validation error?
5. Why allow a field to be `null` instead of required?
6. A long JSON answer fails to parse every time, ending mid-object. What is a likely cause?

<details>
<summary>Answers</summary>

1. A schema describes the expected shape; validation is the code step that checks a response against it.
2. To get structured arguments: the tool's input schema defines the fields, and you just read what the model supplies.
3. A semantic (content) error. Schema validation cannot catch it; your own checks, tests, or a verifier can.
4. It tells the model exactly what to fix, so the next attempt is more likely to succeed.
5. So the model can say "unknown" instead of inventing a value to fill a required field.
6. Truncation from hitting `max_tokens`; check `stop_reason` and raise the limit or shorten the output.

</details>
