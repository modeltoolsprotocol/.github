# Model Tools Protocol (MTP)

**A minimal standard for making CLI tools LLM-discoverable via `--describe`.**

LLM agents need to discover and use tools. Right now there are two worlds:

**CLI tools** are the backbone of software development. They're composable (`|`), scriptable, version-controlled, and work everywhere. But LLMs can't discover what a CLI does. They have to parse `--help` text, guess at arguments, and hope for the best.

**MCP (Model Context Protocol)** solves discovery beautifully. Tools declare typed schemas, and LLM hosts discover them via a structured handshake. But MCP requires running a server process, speaking JSON-RPC over stdio/SSE, and building within the MCP ecosystem. Your existing CLI tools don't get any of this for free.

| | CLI tools | MCP tools |
|---|---|---|
| **Composability** | First-class. Pipes, shell scripts, xargs. 50 years of Unix | Requires an orchestrator or agent framework |
| **Discovery** | Poor. Parse `--help` and hope | Excellent. Typed schemas via handshake |
| **Runtime** | Any shell, anywhere | Needs an MCP host (Claude Desktop, etc.) |
| **Deployment** | `brew install`, `cargo install`, a binary in PATH | Run a server process, configure the host |
| **Interop** | Pipes to anything | Talks to MCP clients only |
| **Scriptable without an LLM** | Yes, that's the whole point of CLIs | Not really, designed for LLM interaction |
| **Streaming / subscriptions** | Limited (stdout streaming) | Built-in (SSE, notifications) |
| **Stateful interaction** | Stateless by design (each invocation is fresh) | Stateful sessions with context |
| **Adoption cost** | One flag/decorator | New protocol, server scaffolding, SDK |

MCP is the right choice when you need stateful sessions, streaming, or deep integration with an LLM host. CLIs are the right choice when you want composability, scriptability, and zero infrastructure.

The gap is discovery. MTP fills it.

Any tool that responds to `--describe` with a JSON manifest becomes instantly discoverable and usable by any LLM agent. No sidecar processes, no daemon servers, no complex handshakes. Just a CLI flag that returns structured JSON.

```bash
$ my-tool --describe
```
```json
{
  "name": "my-tool",
  "version": "1.0.0",
  "description": "Does useful things",
  "commands": [
    {
      "name": "greet",
      "description": "Greet someone by name",
      "args": [
        { "name": "name", "type": "string", "required": true, "description": "Name to greet" }
      ]
    }
  ]
}
```

## Repositories

| Repository | Description |
|---|---|
| [modeltoolsprotocol](https://github.com/modeltoolsprotocol/modeltoolsprotocol) | Protocol spec, auth spec, and project README |
| [mtpcli](https://github.com/modeltoolsprotocol/mtpcli) | Reference CLI for discovering, authenticating, and bridging MTP tools |
| [typescript-sdk](https://github.com/modeltoolsprotocol/typescript-sdk) | TypeScript SDK (Commander.js) |
| [python-sdk](https://github.com/modeltoolsprotocol/python-sdk) | Python SDK (Click) |
| [go-sdk](https://github.com/modeltoolsprotocol/go-sdk) | Go SDK (Cobra) |
| [rust-sdk](https://github.com/modeltoolsprotocol/rust-sdk) | Rust SDK (Clap) |

## Quick Start

### Add `--describe` to an existing CLI

**TypeScript** (Commander.js)
```bash
npm install @modeltoolsprotocol/sdk
```
```typescript
import { withDescribe } from "@modeltoolsprotocol/sdk";
withDescribe(program).parse();
```

**Python** (Click)
```bash
pip install mtp-sdk
```
```python
from mtp_sdk import with_describe
cli = with_describe(cli)
```

**Go** (Cobra)
```bash
go get github.com/modeltoolsprotocol/go-sdk
```
```go
mtpsdk.WithDescribe(rootCmd)
```

**Rust** (Clap)
```toml
[dependencies]
mtp-sdk = "0.1"
```
```rust
use mtp_sdk::Describable;
Cli::parse_with_describe();
```

### Discover and use tools with mtpcli

```bash
npm install -g @modeltoolsprotocol/mtpcli

# Search for tools
mtpcli search "convert files" -- filetool mytool

# Authenticate
mtpcli auth login mytool

# Serve CLI tools over MCP
mtpcli serve --tool mytool

# Wrap an MCP server as a CLI
mtpcli wrap --server "npx @mcp/server-github" --describe
```

## License

Apache-2.0
