# Model Tools Protocol (MTP)

**A minimal standard for making CLI tools LLM-discoverable via `--describe`.**

LLM agents need to discover and use tools. Right now there are two worlds:

**CLI tools** are the backbone of software development. They're composable (`|`), scriptable, version-controlled, and work everywhere. But LLMs can't discover what a CLI does. They have to parse `--help` text, guess at arguments, and hope for the best.

**MCP (Model Context Protocol)** solves discovery beautifully. Tools declare typed schemas, and LLM hosts discover them via a structured handshake. But MCP tools aren't composable. Each invocation goes through the model. You can't pipe one MCP tool's output into another. You can't script them without an LLM in the loop.

| | CLI tools | MCP tools |
|---|---|---|
| **Composability** | First-class. Pipes, shell scripts, xargs. 50 years of Unix | Requires an orchestrator or agent framework |
| **Discovery** | Poor. Parse `--help` and hope | Excellent. Typed schemas via handshake |
| **Runtime** | Any shell, anywhere | Needs an MCP host (Claude Desktop, etc.) |
| **Deployment** | `brew install`, `cargo install`, a binary in PATH | Run a server process, configure the host |
| **Interop** | Pipes to anything | Talks to MCP clients only |
| **Scriptable without an LLM** | Yes, that's the whole point of CLIs | Not really, designed for LLM interaction |
| **Adoption cost** | One flag/decorator | New protocol, server scaffolding, SDK |

The gap is discovery. MTP fills it.

## Three things MTP does

### 1. Makes any CLI tool understandable to an LLM

Any tool that responds to `--describe` emits a JSON schema describing its commands, arguments, types, and usage examples. An LLM reads the schema, knows exactly what to call, and invokes it through shell. No protocol server, no SDK, no infrastructure.

```bash
$ atlasctl --describe
```
```json
{
  "name": "atlasctl",
  "version": "0.2.1",
  "description": "Atlassian CLI for Confluence and Jira",
  "commands": [
    {
      "name": "confluence page get",
      "description": "Get a Confluence page and all comments",
      "args": [
        { "name": "--page-id", "type": "string", "required": true },
        { "name": "--format", "type": "enum", "values": ["markdown", "html", "raw"], "default": "markdown" }
      ],
      "examples": [
        { "command": "atlasctl confluence page get --page-id 12345" }
      ]
    }
  ]
}
```

If you're already using a CLI framework (Click, Cobra, Clap, Commander), you get `--describe` for free. The SDK reads the types, defaults, and help strings your framework already knows about. One function call, zero extra documentation.

### 2. Turns any CLI into an MCP server

```bash
$ mtpcli serve --tool atlasctl

mtpcli serve: serving 4 tool(s) from 1 CLI tool(s)
  - atlasctl__config set
  - atlasctl__config get
  - atlasctl__confluence page get
  ...
```

Drop it into your Claude Desktop config and it works like any other MCP server. The bridge reads `--describe`, translates commands to MCP tools, and shells out to the real CLI when the host calls a tool.

### 3. Turns any MCP server into a composable CLI

```bash
$ mtpcli wrap --server "uvx snowflake-mcp --account myaccount" \
    list_objects -- --object_type database
```

The 2,500+ MCP servers people have built? They're all CLI tools now. Pipe their output, use them in scripts, compose them with other tools.

**You don't have to pick a side.** The same tool can be both a `--describe` CLI and an MCP server. The bridge is bidirectional. Build one, get both.

## Why composability matters

```bash
# Pull a Confluence page, find action items, create Jira tickets for untracked ones
atlasctl confluence page get --page-id "$PAGE_ID" --format markdown \
  | extract-actions \
  | while IFS=$'\t' read -r assignee summary; do
      existing=$(jiractl search --assignee "$assignee" --summary "$summary" --status open)
      if [ -z "$existing" ]; then
        jiractl create --assignee "$assignee" --summary "$summary" --type task
      fi
    done
```

No LLM needed. No tokens burned. Runs in CI, runs in a cron job, runs on a Raspberry Pi. And when you *do* want an LLM in the loop, it can discover these tools via `--describe` and invoke them through shell.

## Repositories

| Repository | Install | Description |
|---|---|---|
| [modeltoolsprotocol](https://github.com/modeltoolsprotocol/modeltoolsprotocol) | | Protocol spec, auth spec |
| [mtpcli](https://github.com/modeltoolsprotocol/mtpcli) | `npm i -g @modeltoolsprotocol/mtpcli` | Discover, authenticate, bridge, validate |
| [typescript-sdk](https://github.com/modeltoolsprotocol/typescript-sdk) | `npm i @modeltoolsprotocol/sdk` | Commander.js |
| [python-sdk](https://github.com/modeltoolsprotocol/python-sdk) | `pip install mtp-sdk` | Click |
| [go-sdk](https://github.com/modeltoolsprotocol/go-sdk) | `go get github.com/modeltoolsprotocol/go-sdk` | Cobra |
| [rust-sdk](https://github.com/modeltoolsprotocol/rust-sdk) | `cargo add mtp-sdk` | Clap |

## License

Apache-2.0
