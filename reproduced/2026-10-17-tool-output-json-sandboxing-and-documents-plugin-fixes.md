# [MVP] [DRAFT] RFC: Tool Output JSON Sandboxing and Documents Plugin Remediation

> **Classification:** `Flaw` (Double-Serialized JSON, Broken Attached-Document Tool Availability, & Polling Anti-Pattern)
> **Draft Status:** Reproduced (`rfcs/reproduced/`) — Failing scenario test `rfc_2026_10_17_tool_output_json_sandboxing_and_documents_plugin_fixes` verified in `synapto/crates/test/tests/scenario_tests.rs`; awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-tool-output-json-sandboxing-and-documents-plugin-fixes.md`.

## 1. User-Centric Motivation & Goal

When a user attaches a document to a chat message or asks the assistant to read a document or URL, the `read_document` tool must be available, wait efficiently for parsing to finish, and return a clean JSON object (`{"content": "..."}`) to the LLM.

Currently, three runtime bugs break this flow in `synapto-plugins/documents` and `crates/core`:
1. **Double-Serialized JSON String:** `DocumentService::execute_read` and `execute_read_url` serialize `{"content": text}` to a `String`, and `ReadDocumentPluginTool::execute` / `ReadUrlPluginTool::execute` wrap that string in `serde_json::Value::String(result_text)`. When `ToolOutput::to_json_string()` serializes `Value::String`, the LLM receives a double-encoded string literal (`"{\"content\":\"...\"}"`) instead of a JSON object.
2. **Broken `read_document` Availability on Attached Documents:** `ReadDocumentPluginTool::is_available` (`synapto-plugins/documents/src/plugin.rs:95-99`) only checks `compiled_context.get("available_documents")` and ignores `attached_documents`. If a user attaches a file in chat when `available_documents` is empty, `read_document` is not offered to the LLM at all!
3. **Unsandboxed Tool Outputs/Errors & 500ms Database Polling:** `RegistryToolExecutor` (`crates/core/src/cognitive/types.rs:100-142, 162, 187`) does not enforce Rule 12 JSON object wrapping on non-object tool outputs (`mcp`, `test/plugins/documents`) or tool error strings (`"Error executing tool: ..."`), and `DocumentService::execute_read` (`service.rs:375-393`) polls the database every 500ms while ignoring `DocumentEvent::Parsed` / `DocumentEvent::Failed` notifications (`service.rs:689-694`).

This RFC fixes the double JSON serialization, enforces Rule 12 JSON object wrapping at the core `RegistryToolExecutor` boundary, fixes `ReadDocumentPluginTool::is_available` to check both `available_documents` and `attached_documents`, and replaces the 500ms DB polling loop with event notification.

---

## 2. Executive Summary

- **Context:** `synapto-plugins/documents/src/{plugin.rs, service.rs}`, `synapto/crates/core/src/cognitive/types.rs`, `synapto/integrations/plugins/mcp/src/lib.rs`, and `synapto/crates/test/src/plugins/documents.rs`.
- **Motivation:** Fix malformed double-encoded tool responses, restore `read_document` tool availability for chat attachments, enforce Rule 12 JSON object sandboxing at the core boundary, and eliminate database busy-polling.
- **Overarching Goals:** Guarantee that every `ToolResponse` sent to an LLM is a single-encoded JSON Object (`serde_json::Value::Object`) and that document reading is event-driven and responsive to attached files.
- **Purpose:** Return `serde_json::Value::Object` from `documents`, `mcp`, and `RegistryToolExecutor`, check `attached_documents` in `ReadDocumentPluginTool::is_available`, and await `DocumentEvent` notifications in `execute_read`.
- **Problem:**
  1. **Double JSON Serialization (`synapto-plugins/documents/src/service.rs:381-382, 442-446, 462-470, 508-512, 597-601` & `plugin.rs:122, 175`):** `execute_read` and `execute_read_url` return `Result<String, String>` containing serialized JSON, which `plugin.rs:122, 175` wraps in `Value::String`.
  2. **Core Tool Executor Non-Object & Error String Responses (`synapto/crates/core/src/cognitive/types.rs:100-142, 162-163, 187-190`, `mcp/src/lib.rs:68-90, 352, 584`, `crates/test/src/plugins/documents.rs:97`):** Non-object `serde_json::Value` tool returns and error strings (`ToolOutput::new(format!("Error executing tool: {}", e))`) serialize as raw JSON strings/primitives rather than `{"content": ...}` / `{"error": ...}` JSON objects (Rule 12).
  3. **`ReadDocumentPluginTool::is_available` Misses `attached_documents` (`synapto-plugins/documents/src/plugin.rs:95-99`):** Only inspects `compiled_context.get("available_documents")` and ignores `attached_documents` in `current_messages`.
  4. **Busy Database Polling & Dropped Reply on Save Error (`synapto-plugins/documents/src/service.rs:375-393, 689-694, 739, 772-785`):** `execute_read` sleeps 500ms in a loop polling `db.get_document_content` instead of subscribing to `document_event_tx`, and `handle_add_document` (`L739`) returns early on `save_file` error without sending an error or closing cleanly.
- **Solution:**
  1. Change `execute_read` and `execute_read_url` (or `plugin.rs:122, 175`) to return `serde_json::json!({ "content": text })` (`Value::Object`) directly.
  2. In `ToolOutput::new` (`crates/llm/src/lib.rs:24`) and `RegistryToolExecutor` (`crates/core/src/cognitive/types.rs`), normalize any non-object `serde_json::Value` into `serde_json::json!({ "content": val })` and wrap tool errors as `serde_json::json!({ "error": err_msg })`.
  3. Update `ReadDocumentPluginTool::is_available` to check `compiled_context` for non-empty `available_documents` OR non-empty `attached_documents` (via fulltext serialization scan per `ARCHITECTURE.md:160`).
  4. Convert `document_event_tx` in `synapto-plugins/documents/src/service.rs` to a `tokio::sync::broadcast::Sender<DocumentEvent>` so `execute_read` awaits `DocumentEvent::Parsed(id)` / `DocumentEvent::Failed(id, err)` directly instead of polling every 500ms.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto-plugins/documents/src/service.rs:381-382` & `plugin.rs:122, 175`**:
   ```rust
   // service.rs:381-382:
   let json_response = serde_json::json!({ "content": text });
   return serde_json::to_string(&json_response).map_err(|e| e.to_string());
   // plugin.rs:122:
   Ok(serde_json::Value::String(result_text))
   ```
2. **`synapto-plugins/documents/src/plugin.rs:95-99`**:
   ```rust
   async fn is_available(&self, _request: &ContextRequest, compiled_context: &serde_json::Value) -> Result<bool, String> {
       Ok(compiled_context
           .get("available_documents")
           .and_then(|docs| docs.as_array())
           .is_some_and(|arr| !arr.is_empty()))
   }
   ```
3. **`synapto-plugins/documents/src/service.rs:375-393, 689-694`**:
   ```rust
   loop {
       match db.get_document_content(&document_id).await {
           Ok((status, Some(text))) if status == DocumentStatus::Available => { ... }
           Ok((DocumentStatus::Discovered | DocumentStatus::Parsing, _)) => {
               tokio::time::sleep(std::time::Duration::from_millis(500)).await;
           }
           ...
       }
   }
   ```
4. **`synapto/crates/core/src/cognitive/types.rs:100-142, 162-163, 187-190`**:
   ```rust
   let output = synapto_llm::ToolOutput::new(format!("Error executing tool: {}", e));
   ```

---

## 4. Structural Hierarchy

- **Mission:** Single-Encoded JSON Object Tool Sandboxing and Event-Driven Document Retrieval.
- **Goal:** Guarantee Rule 12 compliance at `ToolOutput` / `RegistryToolExecutor` and eliminate bugs in `synapto-plugins/documents`.
- **Atomic Responsibilities:**
  - **`ToolOutput::from_value` / `ToolOutput::error` (`crates/llm/src/lib.rs`):** Guarantees that serialized tool output is always a JSON Object (`{"content": ...}` or `{"error": ...}`).
  - **`ReadDocumentPluginTool` (`synapto-plugins/documents/src/plugin.rs`):** Enables `read_document` whenever `available_documents` or `attached_documents` contains at least one document ID, and returns `serde_json::Value::Object`.
  - **`DocumentService::execute_read` (`synapto-plugins/documents/src/service.rs`):** Subscribes to `document_event_tx: broadcast::Sender<DocumentEvent>` before checking initial DB state so it wakes immediately when `DocumentEvent::Parsed` or `DocumentEvent::Failed` fires.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 12 (Prompt Injection Mitigation / JSON Sandboxing):** All untrusted external data and tool outputs returned to the LLM as a `ToolResponse` must be serialized as structured JSON objects (`{"content": "..."}`).
- **Section 6 Dynamic Tool Availability (`ARCHITECTURE.md:160`):** `is_available` must evaluate availability deterministically over `compiled_context` without cross-plugin schema coupling.

### 5.2 Enforced `ToolOutput` Constructor
```rust
// synapto/crates/llm/src/lib.rs
impl ToolOutput {
    pub fn from_value(val: serde_json::Value) -> Self {
        match val {
            serde_json::Value::Object(_) => Self(val),
            other => Self(serde_json::json!({ "content": other })),
        }
    }

    pub fn from_error(err: impl std::fmt::Display) -> Self {
        Self(serde_json::json!({ "error": err.to_string() }))
    }
}
```

### 5.3 JSON Contract Examples

#### Compiled Prompt Context Input with Attached Document
```json
{
  "current_messages": [
    {
      "user": { "Recognized": "Alice" },
      "content": "Please review this attached PDF.",
      "attached_documents": ["doc_9a4f2c10e8b311d4"]
    }
  ],
  "context_plugins": {
    "documents": {
      "available_documents": []
    }
  }
}
```

#### Single-Encoded Tool Output Payload (`ToolResponse`)
```json
{
  "content": "Full extracted text or synthesized answer from the document..."
}
```

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For enforcing JSON Object sandboxing on `Tool::execute` across `synapto-interface` and `synapto-llm`:

**[ Option A: Enforce at `ToolOutput::from_value` / `from_error` Boundary in `synapto-llm` + Fix `documents` Double Serialization ]**
- **Action:** Keep `Tool::execute` returning `Result<serde_json::Value, String>` in `synapto-interface`, fix `documents` to return `serde_json::json!({ "content": text })` instead of `Value::String(serde_json::to_string(...))`, and make `ToolOutput` automatically wrap any non-`Value::Object` in `{"content": val}` and any `Err(e)` in `{"error": e}`.
- **Pros:** Enforces Rule 12 Centrally in `ToolOutput` for every tool (including third-party MCP tools that return strings, arrays, or nulls) without breaking the `Tool::execute` signature.
- **Cons:** None.

**[ Option B: Change `Tool::execute` Return Type to `Result<serde_json::Map<String, serde_json::Value>, String>` ]**
- **Action:** Change `Tool::execute` in `synapto-interface/src/tool.rs:32` to require a JSON `Map` at the trait level.
- **Pros:** Compile-time enforcement that `Tool::execute` returns a JSON object.
- **Cons:** Still requires `ToolOutput::from_error` for error wrapping and requires converting `serde_json::json!({...})` via `.as_object()` in every tool implementation.

**My Recommendation:** I recommend **Option A** because it enforces Rule 12 centrally at the `ToolOutput` constructor for both normal outputs and error outputs without ergonomic friction in `serde_json::json!` macros.
Please select Option A or Option B for this RFC.
