# [MVP] Model Context Protocol (MCP) Integration Plugin

## 1. User-Centric Motivation & Goal
Users want to extend Synapto's capabilities by connecting it to third-party Model Context Protocol (MCP) servers (such as local filesystem servers, database connectors, developer tools, or web search APIs) using industry-standard configurations. By supporting Claude's `mcpServers` JSON configuration layout natively in a dedicated `synapto-plugin-mcp` integration plugin, users can reuse their existing MCP configuration files without modification to immediately expose external tools to Synapto AI.

## 2. Executive Summary
- **Context:** Synapto uses `synapto-interface` to manage tool definitions (`Tool`, `ErasedTool`, `ToolRegistryBuilder`) and plugin initialization (`Plugin`, `PluginInitContext`).
- **Motivation:** The Model Context Protocol (MCP) is an open standard for connecting AI models to external tools and resources. Claude's desktop configuration format (`mcpServers`) is the de facto standard configuration structure across the ecosystem.
- **Overarching Goals:** Provide a dynamic MCP client plugin (`synapto-plugin-mcp`) in `integrations/plugins/mcp` that reads a Claude-compatible `mcpServers` configuration block, manages standard JSON-RPC 2.0 communication over `stdio` and HTTP transports to MCP servers, discovers available tools dynamically, and registers them as `Arc<dyn ErasedTool>` with Synapto's tool registry.
- **Purpose:** Enable seamless integration of any MCP server into Synapto using standard Claude configuration syntax.
- **Problem:** `PluginRegistry` previously only supported compile-time static `Tool` registration, lacking dynamic `ErasedTool` registration and MCP JSON-RPC transport capabilities.
- **Solution:** Add `register_erased_tool` to `PluginRegistry` in `synapto-interface` and `synapto-core`. Create `synapto-plugin-mcp` implementing `synapto_interface::plugin::Plugin`. Upon plugin initialization, it parses the `mcpServers` configuration, connects to enabled MCP servers over stdio or HTTP, executes the MCP `initialize` and `tools/list` handshake, wraps discovered tools as `ErasedTool` implementations with `mcp__{server}__{tool}` namespacing, and delegates tool execution to the underlying MCP server via `tools/call`.

## 3. Current State Analysis
- Synapto's `integrations/plugins` directory contains individual feature plugins (`clock`, `mumble`, `linux-host-audio`).
- Tools were previously defined strictly by implementing `synapto_interface::tool::Tool` with compile-time generic types and registered via `PluginRegistry::register_tool`.
- There was no `register_erased_tool` method on `PluginRegistry` for dynamic tools discovered at runtime.
- **Flaws / Gaps:**
  - Standard MCP tool servers could not be registered without writing custom Rust wrappers for every tool at compile time.
  - No configuration parser existed for Claude-style `mcpServers` definitions.

## 4. Structural Hierarchy
- **Mission:** Dynamic MCP Tool Integration.
- **Goal:** Claude-compatible MCP server bridge for Synapto.
- **Atomic Responsibilities:**
  - **`PluginRegistry` Extension:** Adds `register_erased_tool(&mut self, tool: Arc<dyn ErasedTool>)` to `synapto-interface` and `synapto-core`.
  - **`McpPlugin` (`synapto-plugin-mcp`):** Implements `synapto_interface::plugin::Plugin`. Parses `mcpServers` config from `PluginInitContext` and manages child MCP client transport session actors.
  - **`McpTransportActor`:** Low-level JSON-RPC 2.0 transport actor running a dedicated Tokio background task for stdio child processes (`tokio::process::Child`) or HTTP endpoints (`reqwest`). Uses `mpsc` request channels and `oneshot` reply channels for race-free concurrent tool calls (**Principle 18, Principle 20** in `ARCHITECTURE.md`).
  - **`McpBridgedTool`:** Implements `synapto_interface::tool::ErasedTool`. Translates Synapto tool execution calls into MCP `tools/call` JSON-RPC requests via the transport actor and forwards structured JSON results back (**Principle 12** in `ARCHITECTURE.md`).

## 5. Data Models

### Configuration Schema (Claude `mcpServers` Alignment)

The configuration schema directly mirrors Claude Desktop's `mcpServers` configuration format.

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/username/Desktop"],
      "env": {
        "DEBUG": "mcp:*"
      },
      "disabled": false
    },
    "remote_service": {
      "url": "http://127.0.0.1:8000/sse",
      "headers": {
        "Authorization": "Bearer my-secret-token"
      }
    }
  }
}
```

### Rust Configuration Structs

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct McpPluginConfig {
    #[serde(default, rename = "mcpServers")]
    pub mcp_servers: HashMap<String, McpServerConfig>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct McpServerConfig {
    /// Toggle to disable this server
    #[serde(default)]
    pub disabled: bool,
    /// Target specification (stdio command or SSE/HTTP URL)
    #[serde(flatten)]
    pub target: McpServerTarget,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
pub enum McpServerTarget {
    Stdio {
        command: String,
        #[serde(default)]
        args: Vec<String>,
        #[serde(default)]
        env: HashMap<String, String>,
    },
    Sse {
        url: String,
        #[serde(default)]
        headers: HashMap<String, String>,
    },
}
```

### JSON-RPC 2.0 Protocol Contracts

#### Initialization Request (`initialize`)
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {},
    "clientInfo": {
      "name": "synapto-ai",
      "version": "0.1.0"
    }
  }
}
```

#### Tools Listing Request (`tools/list`)
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list",
  "params": {}
}
```

#### Tools Execution Request (`tools/call`)
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "read_file",
    "arguments": {
      "path": "test.txt"
    }
  }
}
```

## 6. Component Architecture

1. **Plugin Initialization (`McpPlugin::create`)**:
   - Reads `McpPluginConfig` via `context.config()?` or `context.optional_config()?`.
   - Iterates through `mcp_servers`. For each active server (where `disabled` is false):
     - For `McpServerTarget::Stdio`, spawns child process using `tokio::process::Command` with configured `args` and `env`. Pipes stdin and stdout.
     - For `McpServerTarget::Sse`, connects over HTTP using `reqwest::Client` with custom headers.
     - Spawns background worker task (`McpTransportActor`) managing line-delimited JSON-RPC framing or HTTP POST requests and request dispatching via `tokio::sync::mpsc`.
     - Sends `initialize` request and `notifications/initialized` notification.
     - Sends `tools/list` request to discover tools.
     - For each returned tool, creates an `McpBridgedTool` instance wrapping the server actor handle, namespaced tool name (`mcp__{server_name}__{tool_name}`), description, and JSON schema. Uses `Box::leak` for dynamic static string references required by `ErasedTool`.
   - Collects all bridged tools in `McpPlugin`.

2. **Tool Registration (`McpPlugin::register`)**:
   - Loops over all `McpBridgedTool` instances wrapped in `Arc`.
   - Registers each tool into the provided `PluginRegistry` via `registry.register_erased_tool(...)`.

3. **Execution Dispatch (`McpBridgedTool::erased_execute`)**:
   - Receives execution arguments as `serde_json::Value`.
   - Dispatches a `tools/call` command over the server actor's `mpsc` channel containing a `oneshot::Sender` response channel (**Principle 18**).
   - Awaits actor response and returns structured JSON result (**Principle 12**).

## 7. Architectural Diagrams

### Execution Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Core as Synapto Core
    participant Plugin as McpPlugin
    participant Tool as McpBridgedTool
    participant Actor as McpTransportActor
    participant Server as External MCP Process / Server

    Core->>Plugin: Plugin::create(init_context)
    Plugin->>Actor: Spawn Actor & Process / HTTP Client
    Actor->>Server: Send 'initialize'
    Server-->>Actor: Response { capabilities, protocolVersion }
    Actor->>Server: Send 'notifications/initialized'
    Actor->>Server: Send 'tools/list'
    Server-->>Actor: Response { tools: [...] }
    Actor-->>Plugin: Discovered Tools List
    Plugin->>Plugin: Wrap tools in McpBridgedTool instances
    Core->>Plugin: Plugin::register(registry)
    Plugin->>Core: registry.register_erased_tool(McpBridgedTool)

    Note over Core, Server: LLM Tool Execution Phase
    Core->>Tool: erased_execute(ctx_request, args)
    Tool->>Actor: Send McpRequest over mpsc
    Actor->>Server: Send 'tools/call' { name, arguments }
    Server-->>Actor: Response { content: [...] }
    Actor-->>Tool: Return via oneshot channel
    Tool-->>Core: Result<serde_json::Value, String>
```

### State Machine Diagram

```mermaid
stateDiagram-v2
    [*] --> Uninitialized
    Uninitialized --> Connecting: Plugin::create called
    Connecting --> Initializing: Child Process / HTTP Session Initialized
    Initializing --> DiscoveringTools: 'initialize' Handshake Complete
    DiscoveringTools --> Ready: 'tools/list' Fetched & McpBridgedTool Instances Created
    Connecting --> Failed: Spawn Error / Connection Error
    Initializing --> Failed: JSON-RPC Handshake Timeout
    Ready --> ExecutingTool: erased_execute called
    ExecutingTool --> Ready: Response Returned via Oneshot
    Ready --> Terminated: System Shutdown / Process Exit
```

## 8. Robustness & Pragmatic Security Trade-offs

### Risk & Hazard Analysis
1. **Unresponsive / Hanging Process:** An external MCP server process may block or hang when executing a tool or during initialization.
   - *Mitigation:* Timeout handling for stdio JSON-RPC initialization and tool execution calls.
2. **Malformed JSON-RPC Payload:** Server sends unexpected JSON structures.
   - *Mitigation:* Strict serde parsing error wrapping and friendly error message propagation.
3. **Process Crash / Unexpected Exit:** Process terminates unexpectedly while Synapto is running.
   - *Mitigation:* Catch pipe EOF / write errors gracefully during `tools/call` and return informative error to LLM.
4. **Environment / Process Isolation:** Third-party MCP servers execute commands on the host system.
   - *Mitigation:* Explicit user configuration required in `mcpServers`. Document security boundary that MCP servers inherit privileges of the executing user.

### Accepted Risks (MVP Scope)
- Automatic process restart / re-initialization on crash is out of scope for MVP.

## 9. Systemic Consequences

### Affected Workspace Modules
- `synapto/crates/interface`: Added `register_erased_tool` method to `PluginRegistry`. Enhanced rustdoc documentation.
- `synapto/crates/core`: Implemented `register_erased_tool` on `Synapto` (`PluginRegistry` impl).
- `integrations/Cargo.toml`: Added `plugins/mcp` to workspace members, enabled `process` and `io-util` features on `tokio`.
- `integrations/plugins/mcp`: New crate `synapto-plugin-mcp`.

### Documentation Requirements
- High-level architecture and interface documentation updated in `synapto/crates/interface/README.md` and module docs.
- Usage instructions added to `integrations/plugins/mcp/README.md`.

## 10. Definition of Done (DoD)
- [x] Add `register_erased_tool` to `PluginRegistry` in `synapto-interface` and implement in `synapto-core`.
- [x] Enable `process` and `io-util` features on `tokio` in `integrations/Cargo.toml`.
- [x] Create `integrations/plugins/mcp` crate with `Cargo.toml` and `src/lib.rs`.
- [x] Implement `McpPluginConfig` and `McpServerTarget` matching Claude `mcpServers` schema.
- [x] Implement JSON-RPC stdio client transport actor and MCP initialization protocol.
- [x] Implement `McpBridgedTool` implementing `synapto_interface::tool::ErasedTool`.
- [x] Implement `synapto_interface::plugin::Plugin` for `McpPlugin`.
- [x] Write unit and scenario integration tests for `synapto-plugin-mcp`.
- [x] Enhance rustdoc documentation in `synapto/crates/interface`.
- [x] Verify `cargo check` and `clippy` pass cleanly across workspace.
