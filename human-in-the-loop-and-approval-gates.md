# Human-in-the-Loop and Approval Gates

How to decide which agent actions a person must approve before they happen. `prompt-injection-and-agent-safety.md` said to limit what an agent can do; this note covers the next step: for the actions that remain, where do you put a human checkpoint?

## 1. The big idea

An agent loop (see `tool-use-and-agentic-loops.md`) can call tools on its own, many times in a row. **Human-in-the-loop (HITL)** means the loop pauses at chosen points and waits for a person to approve, edit, or reject the next action. The point is not to slow everything down; it is to spend human attention only where a mistake would be costly.

> Analogy: a bank teller can hand out small amounts of cash alone, but a large withdrawal needs a manager's signature. The analogy has a limit: the manager only helps if they actually read what they sign. A gate that people approve without looking at is decoration, not safety.

## 2. What to gate: reversibility and blast radius

Ask two questions about each action:

1. **Can it be undone?** (reversible vs. irreversible)
2. **How much can it affect?** (one file vs. a whole production system, only you vs. other people)

| Action | Reversible? | Reach | Typical policy |
| --- | --- | --- | --- |
| Read a file, search code | Yes | Local | Allow automatically |
| Edit a file in a git branch | Yes (git) | Local | Allow, review the diff later |
| Run tests | Yes | Local | Allow automatically |
| Push a branch, open a PR | Mostly | Shared | Allow, but merge needs a human |
| Send an email or Slack message | No | Other people | Require approval |
| Delete data, drop a table | No | Possibly large | Require approval, or deny outright |
| Deploy to production, spend money | No | Large | Require approval |

Rule of thumb: automate what is reversible and local, gate what is irreversible or shared, and deny what the job never needs.

## 3. Three levels of control

| Level | Meaning | Use for |
| --- | --- | --- |
| Allow | The agent may do it without asking | Safe, frequent, reversible actions |
| Ask | The loop pauses for a person's decision | Risky or irreversible actions |
| Deny | The tool is never available | Anything the job should not do at all |

Deny is the strongest and cheapest control, because an action that is not offered cannot be approved by mistake. This is the allowlist and least-privilege idea from earlier notes (`model-context-protocol.md`, `ci-notes.md`).

## 4. How a gate works in a loop

The gate sits between "the model asked for a tool call" and "your code runs it."

```python
RISKY = {"send_email", "delete_record", "deploy"}

def run_tool_call(call):
    if call.name in RISKY:
        decision = ask_human(
            f"Agent wants to run {call.name} with {call.input}. Approve?"
        )
        if decision != "approve":
            # Tell the model what happened so it can adapt, not just crash.
            return {"is_error": True, "content": "A human declined this action."}
    return execute(call)
```

Points to notice:

- The check lives in **your code**, not in the prompt. A sentence like "always ask before deleting" can be ignored or talked around (for example by injected text); code cannot.
- A rejection goes back as a tool result, so the model can choose another path or stop and explain.
- Show the person the **exact** arguments, not a vague summary. They are approving the real action.

## 5. Where gates can go

| Gate type | When it happens | Example |
| --- | --- | --- |
| Per-action approval | Right before a risky tool call | "Approve sending this email?" |
| Plan approval | Before any work starts | Agent proposes steps; person approves the plan |
| Result review | After the work, before it takes effect | A PR that a human reviews and merges |
| Escalation | When the agent is unsure or stuck | Agent stops and asks a question instead of guessing |

Result review is the usual choice in CI: the agent works unattended on a branch, and the PR is the gate (see `github-actions-and-triggers.md`). Nothing reaches `main` until a person merges.

## 6. Designing gates people will actually use

- **Ask rarely.** Constant prompts cause approval fatigue, and people start clicking "yes" without reading. Fewer, better-placed gates are safer than many.
- **Make approval easy to judge.** Show a diff, the exact command, or the recipient and body, not a paragraph of reasoning.
- **Batch low-risk steps.** Approve a plan once instead of every small step.
- **Plan for no human.** Unattended jobs (scheduled runs, CI) cannot pause for a reply. For them, deny risky tools up front and rely on result review rather than waiting forever.
- **Log decisions.** Record what was proposed, who approved, and when, so you can audit later.

## 7. Common mistakes

- Putting the rule only in the system prompt instead of enforcing it in code or permissions.
- Gating everything, so people stop reading.
- Gating nothing because "the model is usually right." Usually is not always, and rare failures are the expensive ones.
- Approving a summary instead of the real action.
- Letting an unattended job wait on an approval nobody will see.

## Check yourself

1. What two questions help you decide whether an action needs approval?
2. Why is "deny" stronger than "ask"?
3. Why should an approval check live in code instead of only in the prompt?
4. An agent runs nightly with no one watching. How do you keep it safe?
5. What is approval fatigue, and how do you reduce it?
6. In the bank manager analogy, where does the analogy stop matching reality?

<details>
<summary>Answers</summary>

1. Can it be undone, and how far does it reach (who or what is affected)?
2. A denied tool is never offered, so it cannot be approved by accident or by someone rushing; "ask" still depends on a person judging correctly.
3. Prompts are instructions the model may ignore or be tricked out of; code runs every time regardless of what the model says.
4. Deny risky tools, give minimal permissions, and use result review (for example a PR a human merges later) instead of live approvals.
5. People who see too many prompts approve without reading. Ask only for risky actions, batch low-risk steps, and show clear, exact details.
6. A manager's signature only helps if they actually read what they sign; a gate people approve blindly gives no safety.

</details>
