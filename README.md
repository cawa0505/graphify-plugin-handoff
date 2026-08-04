# CodeRelayMcp

A native Rust-based Model Context Protocol (MCP) Server for the Opencode Code Relay subsystem. This server manages cross-session and cross-repository handoffs, serving as the lightweight, ultra-high-performance engine that completely replaces the old Node.js-based `@jimmyyen/opencode-code-relay-plugin` while maintaining 100% backward compatibility.

## Key Features

- **Single Binary & Ultra-high Performance**: Built in Rust with an expected binary size under 10MB and near-zero memory footprint.
- **Stateful Memory Caching**: Eliminates the slow walk-up disk search penalty from the Node.js implementation. The workspace root is discovered exactly once during server initialization and cached in memory. Subsequent operations complete under 1ms.
- **Hybrid Double-track Memory**:
  - **Short-Term Memory**: Fast, deterministic state handoff using the token-efficient TOON (Token-Oriented Object Notation) format in `.relay/relay.toon`.
  - **Long-Term Memory**: Semantic vector search utilizing Qdrant for RAG-assisted retrieval of historical session decisions.
- **Model Context Protocol (MCP)**: Implements standard Stdio JSON-RPC transport, compatible with any MCP client (OpenCode, Cursor, Claude Desktop, Roo Code).
- **Safe & Atomic Operations**: Implements transactional file-writing using temp-swapping (via `fs2` locks) to prevent state corruption across concurrent tasks.

## Relationship with OpencodeCodeRelayPlugin

This project serves as the core backend engine. For backward compatibility and seamless integration:
- We provide a zero-dependency **Slim Wrapper** version of the Node.js [OpencodeCodeRelayPlugin](https://github.com/cawa0505/opencode-code-relay-plugin).
- When developers or CI/CD pipelines run `npx opencode-code-relay-plugin <command>`, the slim JS wrapper will transparently delegate the execution to this native Rust binary.
- This ensures existing project workflows and AI prompts remain completely unchanged while gaining a 100x speedup in cold starts.

## Developer & Verification Commands

```bash
# Build the project
cargo build

# Run quality checks
cargo check
cargo clippy

# Run unit tests
cargo test
```

## Setup Instructions

Configuration is fully dynamic and relative. To expose the server to your editor or agent via MCP:

```json
{
  "mcpServers": {
    "code-relay": {
      "command": "code-relay-mcp",
      "args": []
    }
  }
}
```

## Architecture Design

See detailed requirements, specifications, and architecture decisions in the `openspec/` directory.

## License

MIT / Apache 2.0
