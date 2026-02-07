# Model Tools Protocol (MTP)

**A minimal standard for making CLI tools LLM-discoverable via `--describe`.**

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
