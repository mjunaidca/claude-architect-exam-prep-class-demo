# Subagents and Orchestration

How one agent can hand pieces of a job to other agents. `tool-use-and-agentic-loops.md` showed a single agent looping on tools, and `context-window-management.md` showed that its history eventually fills up. This note covers a common fix and its trade-offs: split the work across several agents, each with its own fresh context.

## 1. The big idea

A **subagent** is a separate agent loop that a main agent (the **orchestrator**) starts for one focused task. The subagent does its work in its own context window and returns a short result. The orchestrator sees only that result, not the subagent's whole working history.

> Analogy: a project lead who hands a research question to a colleague. The colleague reads many documents at their own desk and comes back with a one-page summary. The lead's desk stays clear. The analogy stops at one point: the colleague only knows what the lead tells them in the request. Unlike a real colleague, a subagent does not share the lead's memory of the project, so anything it needs must be written into its task.

## 2. Why use subagents

| Benefit | What it means |
| --- | --- |
| Context isolation | Noisy work (reading 40 files, scanning logs) stays out of the main history |
| Parallelism | Independent tasks can run at the same time |
| Specialization | Each subagent can have its own system prompt, tools, and model |
| Least privilege | A reviewer subagent can be given read-only tools |

## 3. The basic pattern

```
orchestrator
   |-- (task: "find where auth tokens are validated") --> subagent A --> summary
   |-- (task: "list all callers of verify_token")     --> subagent B --> summary
   v
orchestrator combines the summaries and continues
```

In many setups a subagent is exposed to the orchestrator as just another tool. The orchestrator calls it with a task description, and the tool result is the subagent's final answer.

```python
def run_subagent(task: str) -> str:
    messages = [{"role": "user", "content": task}]   # fresh history
    while True:
        resp = client.messages.create(model=MODEL, max_tokens=1024,
                                      system=SUB_PROMPT,
                                      tools=READ_ONLY_TOOLS,
                                      messages=messages)
        messages.append({"role": "assistant", "content": resp.content})
        if resp.stop_reason != "tool_use":
            return "".join(b.text for b in resp.content if b.type == "text")
        messages.append({"role": "user", "content": run_tools(resp.content)})
```

The loop is the same one from the tool-use note; only the starting history and tool list differ.

## 4. Writing a good task for a subagent

The task text is the subagent's entire briefing. Include:

1. **The goal**, and why it matters.
2. **What is already known or ruled out**, so work is not repeated.
3. **Where to look** (file paths, names).
4. **The scope**: what not to touch.
5. **The output format** you want back, and a length limit.

A vague task such as "look into the auth bug" gets a vague answer.

## 5. Common orchestration shapes

| Shape | Use when | Example |
| --- | --- | --- |
| Single subagent | One noisy side task | Search the codebase, return a summary |
| Parallel fan-out | Independent pieces | Review five files at once |
| Pipeline | Each step needs the last | Plan, then implement, then review |
| Verifier | You want a second opinion | A separate agent checks the first one's findings |

## 6. Trade-offs and pitfalls

1. **Cost.** Each subagent has its own loop and token usage, so total cost can exceed one agent's. Several agents are not automatically cheaper or better.
2. **Information loss.** A summary drops detail. If the orchestrator needs the exact line of code, ask for it in the output format.
3. **Wrong or confident summaries.** The orchestrator cannot see how the subagent reached its answer. Verify important claims before acting on them.
4. **Conflicting edits.** Parallel subagents that write to the same files can collide. Give writers separate files, or let only one agent write.
5. **Runaway spawning.** Cap depth and count so agents cannot start agents forever. Apply the iteration and budget limits from the tool-use note to every agent.
6. **Untrusted content.** A subagent that reads web pages or files can be hit by prompt injection. Its output is still data to the orchestrator, not an instruction. Keep risky tools away from agents that read untrusted input.
7. **Not always needed.** For a small task, one agent is simpler. Use subagents when context size, parallelism, or privilege separation is a real problem.

## Check yourself

1. What does the orchestrator receive back from a subagent: its full history or a result?
2. Name two reasons to use a subagent instead of one big agent.
3. A subagent keeps failing because it does not know which repo to search. What went wrong in the task?
4. Two parallel subagents both edit `config.yaml`. What can go wrong, and how do you avoid it?
5. A subagent's summary says "no bugs found." Why should you not treat that as proof?
6. When is a single agent the better choice?

<details>
<summary>Answers</summary>

1. A result (usually a short summary). The history stays in the subagent.
2. Any two of: context isolation, parallelism, specialization, least privilege.
3. The task lacked needed context; the subagent only knows what the task text says.
4. One can overwrite the other's changes. Give writers separate files or use a single writer.
5. The orchestrator cannot see the subagent's process, and summaries can be wrong or incomplete; verify important claims.
6. When the task is small and context, speed, and privilege are not issues, since extra agents add cost and complexity.

</details>
