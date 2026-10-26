# Model Context Protocol (MCP)

How an agent gets tools without you hand-writing each one. `tool-use-and-agentic-loops.md` showed one tool defined inline; `ci-notes.md` mentioned `--mcp-config` in passing. This note explains what sits behind that flag.

## 1. The big idea

**MCP** is an open standard for connecting an AI application to outside tools and data. Instead of every app writing its own GitHub integration, a database integration, and so on, someone writes an **MCP server** once, and any MCP-aware app can use it.

> Analogy: MCP is like a USB port. Any device that follows the standard plugs into any computer that has the port, with no custom wiring. The analogy stops at safety: plugging in a USB device does not decide what the device is allowed to do, and in the same way connecting an MCP server does not mean you should trust everything it offers. You still choose which of its tools to allow.

## 2. The three roles

| Role | What it is | Example |
| --- | --- | --- |
| Host | The AI application the user runs | Claude Code, a custom agent |
| Client | The connector inside the host, one per server | Created by the host automatically |
| Server | A program that exposes capabilities | A GitHub server, a database server |

The host talks to each server through its own client connection.

## 3. What a server can offer

| Primitive | Who decides to use it | Meaning |
| --- | --- | --- |
| Tools | The model | Actions Claude may call (`create_issue`, `run_query`) |
| Resources | The app or user | Read-only data to load into context (a file, a schema) |
| Prompts | The user | Reusable prompt templates, often shown as slash commands |

Tools are the primitive most relevant to the agent loop. When a server's tools are connected, they are added to the `tools` list you already know. Claude requests them with `tool_use`, and the host routes the call to the server and returns a `tool_result`. The loop itself does not change.

## 4. Transports: how host and server talk

| Transport | Server runs | Typical use |
| --- | --- | --- |
| `stdio` | As a local child process on your machine | Local tools, file access |
| HTTP (streamable) | Remote, reached by URL | Hosted services, shared team servers |

Messages are JSON-RPC either way. Remote servers usually need authentication (for example OAuth or a token header), so keep those credentials in secrets, not in config files you commit.

## 5. Configuring servers in Claude Code

Project-level config lives in a `.mcp.json` file at the repo root, which can be committed so the whole team shares it:

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://example.com/mcp",
      "headers": { "Authorization": "Bearer ${GITHUB_TOKEN}" }
    },
    "local-db": {
      "command": "npx",
      "args": ["-y", "some-db-mcp-server"],
      "env": { "DB_URL": "${DB_URL}" }
    }
  }
}
```

Note the `${VAR}` placeholders: the secret values come from the environment, not from the file.

In CI, pass the file with `claude_args: "--mcp-config .mcp.json"` (see `ci-notes.md`). Claude Code also has `claude mcp add` and `claude mcp list` commands for managing servers locally.

## 6. Tool names and allowlists

MCP tools are named `mcp__<server>__<tool>`, for example `mcp__github__create_issue`. Use these names in `--allowedTools` and `--disallowedTools`:

```
--allowedTools "mcp__github__get_issue,mcp__github__list_issues"
```

This is least privilege (from `tool-use-and-agentic-loops.md`) applied to MCP: a review agent can be allowed to read issues but not to create or delete anything, even if the server offers those tools.

## 7. Design and safety rules

1. **Trust the server like code you run.** A `stdio` server executes on your machine or runner. Use servers from sources you trust.
2. **Allowlist specific tools.** Do not enable every tool a server exposes just because it is connected.
3. **Treat tool results as untrusted.** A server returning issue text or web content can carry prompt injection, exactly as in the earlier lessons.
4. **Keep credentials out of git.** Use environment variables and repo Secrets.
5. **Mind the context cost.** Each connected tool adds its description to every request. Connect only the servers a job needs.
6. **Prefer MCP when the integration is reusable.** For a one-off function in your own app, a plain inline tool is simpler.

## 8. MCP versus a plain tool definition

| | Inline tool | MCP server |
| --- | --- | --- |
| Who writes it | You, inside your app | Anyone, packaged separately |
| Reuse across apps | Copy and paste | Connect the same server |
| Execution | Your app's code | The server process |
| Best for | Small, app-specific actions | Shared or third-party integrations |

## Check yourself

1. What problem does MCP solve compared with every app writing its own integrations?
2. Name the three roles (host, client, server) and give one example of a host.
3. Which MCP primitive lets Claude take an action, and which one loads read-only data?
4. What is the difference between the `stdio` and HTTP transports?
5. What is the name format of an MCP tool, and where would you use it to limit an agent?
6. Why should an API token not be written directly in `.mcp.json`?

<details>
<summary>Answers</summary>

1. A server is written once and works with any MCP-aware app, so integrations are not rebuilt for each one.
2. The host is the AI app (e.g. Claude Code), the client is its per-server connector, and the server exposes the capabilities.
3. Tools perform actions; resources provide read-only data.
4. `stdio` runs the server as a local process; HTTP reaches a remote server by URL.
5. `mcp__<server>__<tool>`; use it in `--allowedTools` / `--disallowedTools` for an allowlist.
6. The file is committed to git, which would leak the secret. Use an environment variable placeholder and a repo Secret.

</details>
