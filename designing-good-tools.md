# Designing Good Tools for Agents

`tool-use-and-agentic-loops.md` shows *how* a tool is wired up. This note covers *how to design* the tools so Claude picks the right one, fills it in correctly, and recovers when it fails. Most "the agent is dumb" problems are really tool-design problems.

## 1. The big idea

Claude only knows a tool by its **name, description, and input schema**. It cannot see your code. If those three things are unclear, Claude guesses.

> Analogy: a tool is a button on a control panel, and the label is all the operator can read. A label saying "Do thing" gets pressed at the wrong time. The analogy has a limit: unlike a human operator, Claude receives the labels of *every* button on every request, so a bigger panel costs tokens and makes choosing harder.

## 2. What makes a good description

| Weak | Better |
|------|--------|
| `"Searches stuff"` | `"Search customer orders by order ID or email. Returns up to 10 matches. Use before issuing a refund. Does not search products."` |
| `"Run query"` | `"Run a read-only SQL SELECT on the analytics DB. Fails on INSERT/UPDATE."` |

A strong description says:

1. **What** it does and what it returns.
2. **When** to use it (and when *not* to, especially versus similar tools).
3. **Limits**: read-only, size caps, side effects, required formats.

## 3. Naming and boundaries

- Use clear verb-noun names: `search_orders`, `create_ticket`.
- Keep tools **distinct**. If two tools overlap (`find_user` and `lookup_customer`), Claude will confuse them. Merge them or state the difference.
- Prefer a few well-scoped tools over dozens of near-duplicates. Accuracy of tool choice tends to drop as the list grows.
- Don't make one giant tool with a `mode` flag that does ten unrelated things; the schema becomes hard to follow.

## 4. Designing the input schema

```json
{
  "name": "create_ticket",
  "description": "Create a support ticket. Returns the new ticket ID.",
  "input_schema": {
    "type": "object",
    "properties": {
      "title": { "type": "string", "description": "Short summary, under 80 chars" },
      "priority": { "type": "string", "enum": ["low", "normal", "high"] },
      "customer_email": { "type": "string", "description": "Customer's email address" }
    },
    "required": ["title", "customer_email"]
  }
}
```

- Use `enum` for fixed choices so Claude cannot invent values.
- Describe each parameter, including format and units.
- Mark only truly needed fields `required`.
- Ask for what Claude naturally knows (an email, a name), not internal IDs it would have to make up. If an ID is needed, provide a search tool that returns it.
- Schemas guide Claude but are not a security boundary: **always validate inputs in your own code** (see `structured-outputs-and-validation.md`).

## 5. Designing the output

The tool result is what Claude reads next, so shape it for Claude, not for a database.

- Return the **relevant fields only**; drop noisy internal columns.
- Prefer readable values (`"status": "shipped"`) over opaque codes (`"status": 3`).
- **Cap size** and paginate. A 50,000-token result fills the context (see `context-window-management.md`). Say so when truncated: `"showing 10 of 240, pass page=2 for more"`.

## 6. Errors Claude can act on

When a tool fails, return the error as the tool result with `"is_error": true` rather than crashing the loop. Make the message tell Claude what to do next.

```json
{"type": "tool_result", "tool_use_id": "toolu_01", "is_error": true,
 "content": "No order with ID 'A12'. Order IDs look like 'ORD-12345'. Try search_orders with the customer's email."}
```

A bare `"Error 500"` leaves Claude to retry blindly. This is different from API-level errors, which are covered in `api-errors-rate-limits-and-retries.md`.

## 7. Safety in the design

- Split **read** tools from **write** tools so you can gate the risky ones (`human-in-the-loop-and-approval-gates.md`).
- Make write tools **idempotent** where possible, or accept an idempotency key, since the loop may repeat a call.
- Give each tool least privilege: a tool that can only read one table is safer than raw database access.
- Treat tool output as untrusted data (`prompt-injection-and-agent-safety.md`).

## Cheat sheet

| Part | Rule of thumb |
|------|---------------|
| Name | Verb-noun, distinct from siblings |
| Description | What, when / when not, limits |
| Input schema | Enums, described fields, minimal required |
| Output | Small, readable, paginated |
| Errors | `is_error` plus a hint for the next step |
| Risk | Separate read/write, least privilege, validate |

## Check yourself

1. Claude keeps calling `find_user` when it should call `lookup_customer`. What are two fixes?
2. Why is an `enum` better than a free-text string for a `priority` field?
3. A tool returns an entire 2,000-row table. What problems does that cause, and what would you change?
4. Why return a tool failure with `is_error: true` and a helpful message instead of raising an exception that ends the loop?
5. True or false: if the schema marks a field as an integer, you no longer need to validate it in your code. Explain.

<details>
<summary>Answers</summary>

1. Merge the tools, or rewrite both descriptions to state exactly when to use each (and when not to). Also rename them to be clearly distinct.
2. It restricts Claude to valid values, so it cannot invent ones your code does not handle.
3. It wastes context and cost and can crowd out earlier information. Return only needed fields, cap the rows, and paginate with a note about what was omitted.
4. Claude can read the message and correct course (for example, fix the ID format). An exception stops the run and gives Claude no chance to recover.
5. False. Schemas guide Claude but do not guarantee safe values; validate in your own code.

</details>
