# Model Tools Protocol - Bash is all you need!

The core philosophy: MCP servers shouldn't need to exist. The world already has millions of CLI tools that are composable, scriptable, and work everywhere. The only thing they were missing was a way for LLMs to understand them. MTP fixes that with a single flag.

MTP is a spec that proposes every CLI tool should support a `--describe` flag: a machine-readable JSON schema that tells an LLM exactly what the tool does, what arguments it takes, and how to call it. Think of it as `--help` for machines. Same typed schemas that MCP provides, but without the server process, SDK, or JSON-RPC transport.

For the MCP servers that already exist, [mtpcli](https://github.com/modeltoolsprotocol/mtpcli) converts them into composable CLI tools, and converts any `--describe` CLI into an MCP server for backwards compatibility with MCP hosts.

## The problem

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

## Four things MTP does

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

If you're already using a CLI framework (Click, Cobra, Clap, Commander), just add the MTP SDK and you get `--describe` for free. The SDK reads the types, defaults, and help strings your framework already knows about. One function call, zero extra documentation.

MTP has two layers of type information, matching how CLIs actually work. Arg types are flat (`string`, `boolean`, `enum`, etc.) because CLI flags and positional arguments are always scalar. For structured data flowing through stdin/stdout, IO descriptors support full JSON Schema (draft 2020-12): nested objects, arrays, unions, pattern validation, conditional fields.

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

Atlassian ships an MCP server at `mcp.atlassian.com`. With `mtpcli wrap`, it's a CLI:

```bash
# Discover what tools the Atlassian MCP server offers
$ mtpcli wrap --url "https://mcp.atlassian.com/v1/mcp" --describe

# Fetch a Confluence page
$ mtpcli wrap --url "https://mcp.atlassian.com/v1/mcp" \
    getConfluencePage -- --cloudId "$CLOUD_ID" --pageId 12345 --contentFormat markdown

# Pipe it into jq, grep, or anything else
$ mtpcli wrap --url "https://mcp.atlassian.com/v1/mcp" \
    getConfluencePage -- --cloudId "$CLOUD_ID" --pageId 12345 --contentFormat markdown \
    | jq -r '.body'
```

The 2,500+ MCP servers people have built? They're all CLI tools now. Pipe their output, use them in scripts, compose them with other tools.

**You don't have to pick a side.** The same tool can be both a `--describe` CLI and an MCP server. The bridge is bidirectional. Build one, get both.

### 4. Search across tools without blowing out the context window

With MCP, every tool's schema has to be loaded into the LLM's context upfront. Five servers with 20 tools each means hundreds of tool definitions consuming tens of thousands of tokens before the conversation starts. `mtpcli search` takes a different approach: search across all your tools outside the model, and only surface the relevant ones.

```bash
# Search across specific tools
$ mtpcli search "get a confluence page" -- atlasctl jiractl mytool

# Search all cached tools
$ mtpcli search "deploy to production"

# Scan PATH for all describe-compatible tools and search
$ mtpcli search --scan-path "convert csv to json"
```

The LLM describes what it needs in natural language, `mtpcli search` finds the matching commands, and only those commands enter the context. Instead of 134K tokens of tool definitions, the model sees the 2-3 tools that are actually relevant.

## Why composability matters

When an LLM orchestrates tool calls via MCP, every intermediate result flows back through the model. Pull a Confluence page? The entire page body enters the context window. Search Jira for existing tickets? Every result goes into the context window. A five-step workflow means five inference round-trips, each one slow, expensive, and adding to an ever-growing context that the model has to re-read on every turn.

With composable CLIs, the same workflow is a bash script. This example mixes a native `--describe` CLI tool (`atlasctl`) with Atlassian's MCP server (via `mtpcli wrap`) in the same pipeline:

```bash
ATLASSIAN_MCP="mtpcli wrap --url https://mcp.atlassian.com/v1/mcp"

# Pull a Confluence page using a --describe CLI tool
atlasctl confluence page get --page-id "$PAGE_ID" --format markdown \
  | extract-actions \
  | while IFS=$'\t' read -r assignee summary; do
      # Search Jira via the Atlassian MCP server
      existing=$($ATLASSIAN_MCP searchJiraIssuesUsingJql \
        -- --cloudId "$CLOUD_ID" \
           --jql "assignee = '$assignee' AND summary ~ '$summary' AND status != Done" \
        | jq '.issues | length')
      if [ "$existing" -eq 0 ]; then
        # Create a ticket via the same MCP server
        $ATLASSIAN_MCP createJiraIssue \
          -- --cloudId "$CLOUD_ID" \
             --projectKey PROJ \
             --issueTypeName Task \
             --summary "$summary"
      fi
    done
```

A `--describe` CLI and an MCP server, composed in the same pipeline. No LLM in the loop. No tokens burned on intermediate data the model doesn't need to see. No context window bloat. Deterministic, fast, free. Runs in CI, runs in a cron job, runs on a Raspberry Pi.

And when you *do* want an LLM in the loop (to decide which action items matter, or to draft ticket descriptions), it can discover these tools via `--describe` and invoke them through shell. You get the best of both worlds: LLM intelligence where it adds value, bash orchestration where it doesn't.

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
