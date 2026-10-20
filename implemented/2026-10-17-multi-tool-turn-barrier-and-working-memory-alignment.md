# [MVP] RFC: Multi-Tool Turn Barrier and Working Memory Alignment

> **Classification:** `Flaw` (Concurrency Race & Data Loss in Tool Execution)

## 1. User-Centric Motivation & Goal

When a user asks a question that requires the AI to call multiple tools in the same turn (such as reading two attached documents or querying multiple tools in parallel), the AI must wait for all requested tools in that turn to complete before generating its answer, and it must never lose any tool result or hang forever when it hits its tool-call limit.

Currently, `RegistryToolExecutor::execute` spawns a separate task for each tool call that sends its result individually to `tool_resolved_tx`. The cognitive loops (`cognitive_direct_task` and `cognitive_side_task`) wake up as soon as the **first** tool finishes and immediately invoke the LLM while the remaining tools from the same turn are still running. Worse, if two tools finish while `cognitive_direct_task` is waiting for speech playback to finish, the second tool overwrites `resolved_tools`, permanently dropping the first tool's response and corrupting `WorkingMemoryStore` indexing. Finally, if the LLM emits a tool call when `max_tool_turns` has been reached, `synapto-llm` returns `Ok(LLMResult::Interrupted(None, vec![]))` with zero tools, causing the cognitive loop to wait on `tool_resolved_rx` forever.

This RFC introduces an atomic turn-scoped barrier for multi-tool execution, triggers tool execution explicitly in the cognitive loops only after barge-in validation and interaction persistence (Option B), aligns `WorkingMemoryStore` with batch resolution, and fixes the `max_tool_turns` hang and counter reset bugs.

---

## 2. Executive Summary

- **Context:** `synapto` (`crates/core/src/cognitive/{types.rs, direct.rs, side.rs}`, `working_memory/store.rs`) and `synapto-llm` (`crates/llm/src/lib.rs`) orchestrate native LLM tool calling across turns.
- **Motivation:** Concurrent tool calls within a single LLM turn currently race against each other, corrupt working memory slice offsets, spawn orphan tasks on barge-in, and can deadlock the cognitive loop when `max_tool_turns` is reached.
- **Overarching Goals:** Guarantee that all tool calls emitted in a single LLM turn execute only after turn commit, run concurrently, and resolve as a single atomic batch before waking the cognitive loop.
- **Purpose:** Fix multi-tool turn coordination, `WorkingMemoryStore` offset alignment, `consecutive_tool_turns` counter tracking, barge-in cancellation tool cleanup, and `max_tool_turns` error handling.
- **Problem:**
  1. **Premature Wakeup on Multi-Tool Turns (`crates/core/src/cognitive/types.rs:50-198`, `direct.rs:347`, `side.rs:202`):** `RegistryToolExecutor::execute` spawns an independent task per tool call sending `(ToolOutput, ToolCall)` over `tool_resolved_tx`. Both cognitive loops wake on the first single message and trigger an LLM inference before sibling tools finish.
  2. **Overwritten `resolved_tools` & Corrupted `WorkingMemoryStore` Slice (`crates/core/src/cognitive/direct.rs:367` & `working_memory/store.rs:139`):** While `cognitive_direct_task` waits for `cognitive_speaking_semaphore` (`run_after_unpaused = true`), each arriving tool appends to `working_memory_store`, but `resolved_tools = Some(vec![(tool_call.clone(), doc_text)])` (`direct.rs:367`) overwrites the vector instead of appending. `WorkingMemoryProvider::context` (`working_memory/store.rs:139`) then slices `mem.len().saturating_sub(request.resolved_tools_count)` with `resolved_tools_count = 1`, misclassifying current-turn tools as historical working memory and losing their `ToolResponse` messages.
  3. **Infinite Hang When LLM Emits Tool Calls at `max_tool_turns` (`crates/llm/src/lib.rs:263-267` & `direct.rs:634-642`, `side.rs:450-458`):** When `consecutive_tool_turns >= max_tool_turns`, `dynamic_tools` is empty (`has_tools = false`). If the LLM still outputs a tool call, `crates/llm/src/lib.rs:266` returns `Tools::interrupted(None, vec![])` (`Ok(LLMResult::Interrupted(None, vec![]))`) without executing any tool, hanging the cognitive loop forever on `tool_resolved_rx`.
  4. **`consecutive_tool_turns` Counter Reset Bug (`crates/core/src/cognitive/direct.rs:398-402`):** If a user speech message arrives while a tool resolves during speaking, `!new_speech_messages.is_empty()` resets `consecutive_tool_turns = 0` on a tool turn.
  5. **Barge-In Cancellation Orphan Tool Execution & In-Flight Ordering Race (`crates/core/src/cognitive/direct.rs:659-675`, `side.rs:264-287`, & `crates/llm/src/lib.rs:270-272`):** Because `self.tools_state.execute(ctx, tool_calls)` is called inside `LLMClient::call_inner` before returning `LLMResult::Interrupted`, if `interrupt.interrupted()` wins the `tokio_select!` at the boundary or before the interaction is recorded, background tool tasks are already spawned for a discarded turn. Furthermore, fast or immediate error tool completions can emit `resolve_in_flight_tool_tx` before `process_llm_output` sends the `Interaction` containing `in_flight_tools` to `new_interaction_tx`.
- **Solution (Option B):**
  1. Spawn a single turn-coordinator task in `RegistryToolExecutor::execute` that runs all `tool_calls` concurrently via `futures::future::join_all` and sends the complete `ResolvedToolBatch` (`Vec<(ToolOutput, ToolCall)>`) as one atomic message over `tool_resolved_tx`.
  2. Remove automatic `self.tools_state.execute(...)` from `LLMClient::call_inner` (`synapto/crates/llm/src/lib.rs`), expose `LLMClient::execute_tools(&self, ctx, tool_calls)`, and invoke it in `direct.rs` and `side.rs` only after the LLM call completes without barge-in and `process_llm_output` commits the `Interaction` (with `in_flight_tools`) to `new_interaction_tx`.
  3. In `direct.rs` and `side.rs`, accumulate `resolved_tools` across the entire `ResolvedToolBatch` via `.extend(...)` rather than overwriting with `Some(vec![...])`, and increment `consecutive_tool_turns` whenever `resolved_tools.is_some()` prior to checking `!new_speech_messages.is_empty()`.
  4. In `synapto/crates/llm/src/lib.rs:266`, return `Err(anyhow::anyhow!("LLM returned tool calls unexpectedly"))` when `!tool_calls.is_empty() && !has_tools`.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/crates/core/src/cognitive/types.rs:33-199`**:
   `RegistryToolExecutor` holds `pub tool_resolved_tx: tokio::sync::mpsc::Sender<(ToolOutput, ToolCall)>`. Inside `ToolExecutor::execute`, it iterates `for call in tool_calls` and spawns an independent `tokio::spawn` per found tool (or sends immediately on the current task if the tool is not found in `tools`), calling `tool_resolved_tx.send((output, call)).await` individually for each tool call.
   ```rust
   pub(super) struct RegistryToolExecutor {
       pub tool_resolved_tx: tokio::sync::mpsc::Sender<(ToolOutput, ToolCall)>,
       pub tools: Arc<synapto_interface::tool::ToolRegistryBuilder>,
       pub cognitive_direct_state_tx: Option<broadcast::Sender<CognitiveDirectStateUpdate>>,
   }
   ```
2. **`synapto/crates/core/src/cognitive/direct.rs:347-402, 659-717`**:
   - At lines 347–386, `tool_resolved_rx.recv()` receives a single `(doc_text, tool_call)` tuple, appends one `WorkingMemoryEntry` to `working_memory_store`, and overwrites `resolved_tools = Some(vec![(tool_call.clone(), doc_text)])` at line 367.
   - At lines 398–402:
     ```rust
     if !new_speech_messages.is_empty() {
         consecutive_tool_turns = 0;
     } else if resolved_tools.is_some() {
         consecutive_tool_turns += 1;
     }
     ```
   - At lines 659–675, `llm_client.call(...)` races against `interrupt.interrupted()`. Because `llm_client.call` internally triggers `self.tools_state.execute(...)` before returning, a barge-in interrupt or an interaction creation after `llm_client.call` happens after tool tasks are already running.
3. **`synapto/crates/core/src/cognitive/side.rs:202-245, 469-513`**:
   - At lines 202–234, `tool_resolved_rx.recv()` receives a single `(doc_text, tool_call)` tuple and immediately wakes the side cognitive loop with `Some(vec![(tool_call, doc_text)])` while sibling tool calls from the same turn are still executing.
   - At lines 265–287, `side.rs` waits for `interaction_memory_rx` to reflect `resolved_tools` for each `call_id`. Because `tools_state.execute` was called inside `llm_client.call` before `process_llm_output` sent `new_interaction_tx`, fast tools (or unknown-tool errors that send on `tool_resolved_tx` before returning from `execute`) can send `resolve_in_flight_tool_tx` before the interaction exists in `interaction_memory_task`.
4. **`synapto/crates/core/src/working_memory/store.rs:137-141`**:
   ```rust
   async fn context(&self, request: &ContextRequest) -> Result<Self::Context, String> {
       let mem = self.store.memory.read().await;
       let visible_len = mem.len().saturating_sub(request.resolved_tools_count);
       Ok(ActiveWorkingMemory::new(mem[..visible_len].to_vec()))
   }
   ```
   `WorkingMemoryProvider::context` relies on `request.resolved_tools_count` matching the exact number of current-turn entries appended at the tail of `WorkingMemoryStore`.
5. **`synapto/crates/llm/src/lib.rs:263-292, 322-344`**:
   ```rust
   if !tool_calls.is_empty() {
       if !has_tools {
           tracing::error!("LLM returned tool calls unexpectedly");
           return Tools::interrupted(None, vec![]);
       }

       let tool_calls_cloned: Vec<_> = tool_calls.into_iter().cloned().collect();
       self.tools_state
           .execute(ctx.unwrap_or_default(), tool_calls_cloned.clone())
           .await;
       ...
       return Tools::interrupted(parsed_output, tool_calls_cloned);
   }
   ```
   When `Tools` is `WithTools<Executor>`, `Tools::interrupted(None, vec![])` returns `Ok(LLMResult::Interrupted(None, vec![]))`, which causes `direct.rs` and `side.rs` to finish the turn expecting a tool resolution that will never arrive.

---

## 4. Structural Hierarchy

- **Mission:** Atomic Multi-Tool Turn Execution and Deterministic Working Memory State.
- **Goal:** Ensure 1:1 batch alignment between LLM tool calls, post-commit execution dispatch, `tool_resolved_tx` delivery, and `WorkingMemoryStore` tail slicing.
- **Atomic Responsibilities:**
  - **`LLMClient` (`crates/llm/src/lib.rs`):** Executes the LLM request, validates that tool calls are only returned when `has_tools` is true (returning `Err` otherwise), returns `LLMResult::Interrupted(parsed_output, tool_calls)` without side-effecting background tasks, and provides `LLMClient::execute_tools(&self, ctx, tool_calls)` for explicit post-commit dispatch.
  - **`RegistryToolExecutor::execute` (`crates/core/src/cognitive/types.rs`):** Spawns one coordinator task per turn that executes all `tool_calls` concurrently via `futures::future::join_all` and sends `ResolvedToolBatch` (`Vec<(ToolOutput, ToolCall)>`) as a single atomic message over `tool_resolved_tx`.
  - **Cognitive Loops (`crates/core/src/cognitive/{direct.rs, side.rs}`):** Persist the interrupted `Interaction` (with `in_flight_tools`) via `process_llm_output`, then invoke `llm_client.execute_tools(request, tool_calls).await`, and upon receiving `ResolvedToolBatch` from `tool_resolved_rx`, append all entries to `WorkingMemoryStore`, resolve all in-flight tool markers, and accumulate `resolved_tools`.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 4 & Rule 5 (`synapto/docs/src/ARCHITECTURE.md` — Native Tool Calling & Generic Tool Architecture):** Tool calls within a turn must resolve cleanly into `ToolResponse` messages via the `ToolExecutor` abstraction without context corruption or infinite loops.
- **Rule 8 & Rule 23 (`synapto/docs/src/ARCHITECTURE.md` — Direct Channel Coupling & Direct Wiring Principle):** The `tool_resolved_tx` / `tool_resolved_rx` channel directly couples the loop-scoped `RegistryToolExecutor` to its owning cognitive task without intermediate brokers.
- **Rule 6 (`synapto/docs/src/ARCHITECTURE.md` — Strict LLM I/O Boundaries `!LLMSafe`):** `ResolvedToolBatch` and `ToolOutput` remain internal transport types outside the LLM schema boundary, converted into structured JSON `ToolResponse` payloads at the `LLMClient` boundary.
- **Rule 17 (`synapto/docs/src/ARCHITECTURE.md` — Opaque Resource Identifier Protocol):** Tool call correlation uses the strongly-typed `synapto_interface::tool::ToolCallId` newtype when resolving in-flight tool state.

### 5.2 Batch Channel Type & `LLMClient` Dispatch Signature

```rust
// synapto/crates/core/src/cognitive/types.rs
pub(super) type ResolvedToolBatch = Vec<(synapto_llm::ToolOutput, synapto_interface::llm::genai::chat::ToolCall)>;

pub(super) struct RegistryToolExecutor {
    pub tool_resolved_tx: tokio::sync::mpsc::Sender<ResolvedToolBatch>,
    pub tools: Arc<synapto_interface::tool::ToolRegistryBuilder>,
    pub cognitive_direct_state_tx: Option<broadcast::Sender<CognitiveDirectStateUpdate>>,
}
```

```rust
// synapto/crates/llm/src/lib.rs
impl<Content: Serialize + std::fmt::Debug, Output: DeserializeOwned, Executor: ToolExecutor>
    LLMClient<Content, Output, WithTools<Executor>>
{
    pub async fn call(
        &self,
        content: Content,
        instructions: Option<Vec<Instruction>>,
        override_reasoning_effort: Option<ReasoningEffort>,
        resolved_tools: Option<ResolvedTools>,
        override_tools: Option<Vec<Tool>>,
    ) -> Result<LLMResult<Output>, anyhow::Error>;

    pub async fn execute_tools(
        &self,
        ctx: synapto_interface::context::ContextRequest,
        tool_calls: Vec<ToolCall>,
    );
}
```

### 5.3 JSON Contract Examples

#### Compiled Prompt Context Input (`CognitiveLLMContent` on Turn Following Multi-Tool Resolution)
```json
{
  "historical_contexts": {},
  "current_contexts": {
    "working_memory": [
      {
        "tool_name": "mock_lookup_key",
        "arguments": {},
        "output": { "key": "SEC-9942-OMEGA" },
        "state": "Original"
      },
      {
        "tool_name": "mock_slow_lookup_key",
        "arguments": {},
        "output": { "key": "SEC-9942-OMEGA" },
        "state": "Original"
      },
      {
        "tool_name": "mock_slow_read",
        "arguments": {},
        "output": { "architect": "Alice" },
        "state": "Original"
      }
    ]
  },
  "prospective_contexts": {},
  "interaction_memory": [],
  "user_messages": []
}
```

#### Multi-Tool Resolution Wire Input (`RawLlmOptions::resolved_tools`)
```json
[
  [
    {
      "call_id": "call_1",
      "fn_name": "mock_lookup_key",
      "fn_arguments": {}
    },
    "{\"key\":\"SEC-9942-OMEGA\"}"
  ],
  [
    {
      "call_id": "call_2",
      "fn_name": "mock_slow_lookup_key",
      "fn_arguments": {}
    },
    "{\"key\":\"SEC-9942-OMEGA\"}"
  ],
  [
    {
      "call_id": "call_3",
      "fn_name": "mock_slow_read",
      "fn_arguments": {}
    },
    "{\"architect\":\"Alice\"}"
  ]
]
```

#### LLM Structured Output Command Payload (`CognitiveLLMOutput<CognitiveSideCommands>`)
```json
{
  "reasoning": "All three tools resolved in a single batch; security key stored and architect Alice identified.",
  "users_messages_evaluation": "Actionable",
  "commands": {
    "write": {
      "content": "Confirmation code: KEY-STORED-SUCCESSFULLY. Architect: Alice.",
      "target": {
        "plugin": "test_chat",
        "context": "{}"
      }
    }
  }
}
```

### 5.4 Architectural Fork Checkpoint (Phase 2 — Resolved: Option B Selected)

For handling where `ToolExecutor::execute(ctx, tool_calls)` is invoked (`synapto/crates/llm/src/lib.rs:271` vs. cognitive loops):

**[ Option A: Keep `ToolExecutor::execute` in `LLMClient` Using Turn-Scoped `join_all` Batching ]** *(Rejected)*
- **Action:** Keep `self.tools_state.execute(tool_calls)` in `LLMClient::call_inner`, change `RegistryToolExecutor::execute` to spawn a single `join_all` batch task sending `Vec<(ToolOutput, ToolCall)>`, and if `cognitive_direct_task` is interrupted by barge-in during `llm_client.call`, drain/discard any stale batch in `tool_resolved_rx` when starting the next turn.
- **Pros:** Preserves the `ToolExecutor` invocation inside `synapto-llm`.
- **Cons:** Still spawns background tool executions on a barge-in interrupted turn and can send `resolve_in_flight_tool_tx` before `new_interaction_tx` is recorded.

**[ Option B: Trigger Tool Execution Explicitly in the Cognitive Loop After Barge-In Check ]** *(Selected by Lead Architect)*
- **Action:** Have `LLMClient::call` return `LLMResult::Interrupted(parsed_output, tool_calls)` *without* calling `self.tools_state.execute(...)` inside `synapto-llm`, and invoke `llm_client.execute_tools(request, tool_calls).await` in `direct.rs` and `side.rs` only after `tokio_select!` completes without barge-in interruption and `process_llm_output` commits the `Interaction` (with `in_flight_tools`) to `new_interaction_tx`.
- **Pros:** Impossible for a barge-in cancelled LLM call to spawn orphan tool executions; guarantees `Interaction` with `in_flight_tools` is queued on `new_interaction_tx` before any tool task can emit `resolve_in_flight_tool_tx`.
- **Cons:** Moves the `execute_tools` call site to `direct.rs` and `side.rs`.

---

## 6. Component Architecture

### 6.1 `synapto-llm` (`synapto/crates/llm/src/lib.rs`)
1. **Unexpected Tool Call Error:** In `LLMClient::call_inner`, when `!tool_calls.is_empty()` and `!has_tools`, log `tracing::error!("LLM returned tool calls unexpectedly")` and return `Err(anyhow::anyhow!("LLM returned tool calls unexpectedly"))` instead of `Tools::interrupted(None, vec![])`.
2. **Decouple Tool Execution from `call_inner`:** Remove `ctx: Option<synapto_interface::context::ContextRequest>` and `self.tools_state.execute(...)` from `LLMClient::call_inner`, and remove `fn execute` from the internal `ToolMode` trait.
3. **Explicit `execute_tools` Method:** On `LLMClient<Content, Output, WithTools<Executor>>`, remove the `ctx` parameter from `pub async fn call(...)` and add:
   ```rust
   pub async fn execute_tools(
       &self,
       ctx: synapto_interface::context::ContextRequest,
       tool_calls: Vec<ToolCall>,
   ) {
       self.tools_state.0.execute(ctx, tool_calls).await;
   }
   ```

### 6.2 Turn-Scoped Batch Execution (`synapto/crates/core/src/cognitive/types.rs`)
1. Define `pub(super) type ResolvedToolBatch = Vec<(ToolOutput, ToolCall)>;` and update `RegistryToolExecutor::tool_resolved_tx` to `tokio::sync::mpsc::Sender<ResolvedToolBatch>`.
2. In `RegistryToolExecutor::execute(&self, ctx_request: ContextRequest, tool_calls: Vec<ToolCall>)`:
   - If `tool_calls.is_empty()`, return immediately.
   - Spawn a single coordinator task via `tokio::spawn` that maps each `call` in `tool_calls` to an async future executing that tool call concurrently via `futures::future::join_all`:
     - Emit `CognitiveDirectState::ToolExecuting` (if `cognitive_direct_state_tx` is `Some`).
     - If `tools.get(&call.fn_name)` is `Some(tool)`:
       - Parse `call.fn_arguments` into `serde_json::Value`.
       - On `Ok(args)`, run `tokio::time::timeout(std::time::Duration::from_secs(60), tool.erased_execute(&ctx_request, args)).await`, emit `CognitiveDirectState::ToolResolved`, and return `(output, call)`.
       - On `Err(e)`, emit `CognitiveDirectState::ToolResolved { is_error: true }` and return `(ToolOutput::new(format!("Error parsing arguments: {}", e)), call)`.
     - If `tools.get(&call.fn_name)` is `None`:
       - Emit `CognitiveDirectState::ToolResolved { is_error: true }` and return `(ToolOutput::new(format!("Error: Tool '{}' not found in registry.", call.fn_name)), call)`.
   - Once `join_all` completes and returns `batch: ResolvedToolBatch` in deterministic call order, send `batch` in a single `tool_resolved_tx.send(batch).await` operation.

### 6.3 Cognitive Direct Loop (`synapto/crates/core/src/cognitive/direct.rs`)
1. **Batch Reception & Accumulation:** When `tool_resolved_rx.recv()` yields `Some(batch)`:
   - Iterate `for (doc_text, tool_call) in batch`:
     - Parse `doc_text.to_json_string()` into `serde_json::Value` and append a `WorkingMemoryEntry` (`WorkingMemoryState::Original`) to `working_memory_store`.
     - Send `synapto_interface::tool::ToolCallId(tool_call.call_id.clone())` over `resolve_in_flight_tool_tx`.
     - Append `(tool_call, doc_text)` to `resolved_tools.get_or_insert_with(Vec::new)`.
   - Set `run_after_unpaused = true`.
2. **`consecutive_tool_turns` Priority:** Check `if resolved_tools.is_some()` first so a tool resolution turn always increments `consecutive_tool_turns += 1` even if `new_speech_messages` arrived while waiting:
   ```rust
   if resolved_tools.is_some() {
       consecutive_tool_turns += 1;
   } else if !new_speech_messages.is_empty() {
       consecutive_tool_turns = 0;
   }
   ```
3. **Post-Commit Tool Dispatch:** Extract `let interrupted_tool_calls = match &generated_text_result { Ok(synapto_llm::LLMResult::Interrupted(_, calls)) => Some(calls.clone()), _ => None };` before `process_llm_output(...)`. After `process_llm_output(...).await` finishes, if `let Some(tool_calls) = interrupted_tool_calls`, call `llm_client.execute_tools(request, tool_calls).await`.

### 6.4 Cognitive Side Loop (`synapto/crates/core/src/cognitive/side.rs`)
1. **Batch Reception:** When `tool_resolved_rx.recv()` yields `Some(batch)`:
   - Iterate `for (doc_text, tool_call) in batch`:
     - Append `WorkingMemoryEntry` to `working_memory_store`.
     - Send `synapto_interface::tool::ToolCallId(tool_call.call_id.clone())` over `resolve_in_flight_tool_tx`.
     - Push `(tool_call, doc_text)` into `batch_resolved`.
   - Return `(true, None, Some(batch_resolved))`.
2. **Post-Commit Tool Dispatch:** Extract `interrupted_tool_calls` before `process_llm_output(...)`. After `process_llm_output(...).await` records the `Interaction` on `new_interaction_tx`, if `let Some(tool_calls) = interrupted_tool_calls`, call `llm_client.execute_tools(request, tool_calls).await`.

---

## 7. Architectural Diagrams

### 7.1 Multi-Tool Turn Barrier & Post-Commit Execution Flow

```mermaid
sequenceDiagram
    participant Loop as Cognitive Loop (Direct / Side)
    participant LLM as LLMClient
    participant Mem as InteractionMemoryTask
    participant Exec as RegistryToolExecutor
    participant Tools as Tool Plugins (1..N)
    participant WM as WorkingMemoryStore

    Loop->>LLM: call(content, dynamic_tools)
    LLM-->>Loop: Ok(LLMResult::Interrupted(output, [call_1, call_2, call_3]))
    Note over Loop: Barge-in check passed
    Loop->>Mem: new_interaction_tx.send(Interaction { in_flight_tools: [1, 2, 3] })
    Loop->>Exec: llm_client.execute_tools(request, [call_1, call_2, call_3])
    Exec->>Tools: join_all([exec(call_1), exec(call_2), exec(call_3)])
    Tools-->>Exec: [(out_1, call_1), (out_2, call_2), (out_3, call_3)]
    Exec->>Loop: tool_resolved_tx.send(ResolvedToolBatch)
    Loop->>WM: append(entry_1), append(entry_2), append(entry_3)
    Loop->>Mem: resolve_in_flight_tool_tx.send(id_1, id_2, id_3)
    Loop->>WM: gather_contexts(resolved_tools_count = 3)
    WM-->>Loop: mem[..mem.len() - 3] (excludes current turn's 3 tools)
    Loop->>LLM: call(content, resolved_tools = [1, 2, 3])
    LLM-->>Loop: Ok(LLMResult::Output(final_response))
```

### 7.2 Turn & Tool Resolution State Machine

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Thinking: User Input Trigger
    Thinking --> Idle: Barge-In Interrupted (Zero Tools Spawned)
    Thinking --> DispatchingTools: LLMResult::Interrupted(tool_calls)
    Thinking --> Idle: LLMResult::Output (consecutive_tool_turns = 0)
    Thinking --> Idle: Err(UnexpectedToolCalls) (consecutive_tool_turns = 0)
    DispatchingTools --> WaitingForBatch: Interaction Committed & join_all Spawned
    WaitingForBatch --> BatchResolved: All N Tools Complete (Single Batch Message)
    BatchResolved --> Thinking: Append N to WorkingMemory & Increment consecutive_tool_turns
```

---

## 8. Robustness & Pragmatic Security Trade-offs

### 8.1 Hazard & Failure Mode Analysis
- **Slow or Hung Individual Tool in a Batch:** Because `join_all` waits for all tools in the batch, one hung tool could block the batch indefinitely if unbounded.
  - **Mitigation:** Every individual tool execution inside the `join_all` batch remains wrapped in `tokio::time::timeout(std::time::Duration::from_secs(60), ...)` (`crates/core/src/cognitive/types.rs:79-83`). Thus, the entire batch is strictly bounded to at most 60 seconds regardless of batch size.
- **LLM Ignores Tool Suppression at `max_tool_turns`:** Some LLM providers may still emit a tool call payload even when `tools: None` (`has_tools = false`) is sent on the request.
  - **Mitigation:** `LLMClient::call_inner` now returns `Err(anyhow::anyhow!("LLM returned tool calls unexpectedly"))` when `!tool_calls.is_empty() && !has_tools`. Both cognitive loops handle `Err` by resetting `consecutive_tool_turns = 0` and logging the error without waiting on `tool_resolved_rx`.
- **Barge-In During LLM Inference (`direct.rs`):** If a user speaks while `llm_client.call` is awaiting the network response, `interrupt.interrupted()` aborts the `llm_client.call` branch.
  - **Mitigation:** Under Option B, `llm_client.execute_tools` is only invoked after `tokio_select!` completes on the `llm_client.call` branch, preventing orphan tool executions.

### 8.2 Accepted Risks (MVP Scope)
- **Mid-Execution Barge-In After Tools Spawned:** If a user barges in *after* an interrupted LLM turn has already completed and spawned background tools, the background `join_all` task continues to completion and populates `WorkingMemoryStore`. Aborting in-flight external tool side effects mid-execution (or adding drop guards) is out of scope for this `[MVP]` RFC and treated as an Accepted Risk.

---

## 9. Systemic Consequences

### 9.1 Mandatory Codebase Audit
A mechanical search (`rg "ToolExecutor|WithTools|RegistryToolExecutor|tool_resolved_|resolved_tools|create_client_with_tools|ToolMode"`) across the workspace confirmed that only the following files reference or depend on the modified interfaces:

1. `synapto/crates/llm/src/lib.rs` (defines `ToolExecutor`, `ToolMode`, `WithoutTools`, `WithTools`, `LLMClient::call_inner`, `LLMClient::call`, `LLM::create_client_with_tools`)
2. `synapto/crates/core/src/cognitive/types.rs` (defines `RegistryToolExecutor` and implements `ToolExecutor`)
3. `synapto/crates/core/src/cognitive/direct.rs` (instantiates `RegistryToolExecutor`, calls `create_client_with_tools`, `llm_client.call`, and processes `tool_resolved_rx`)
4. `synapto/crates/core/src/cognitive/side.rs` (instantiates `RegistryToolExecutor`, calls `create_client_with_tools`, `llm_client.call`, and processes `tool_resolved_rx`)
5. `synapto/crates/core/src/working_memory/store.rs` (consumes `request.resolved_tools_count` in `WorkingMemoryProvider::context`)

No external plugins (`synapto/integrations/plugins/*` or `synapto-plugins/*`) call `create_client_with_tools` or implement `ToolExecutor`; all plugins use `create_client` (`WithoutTools`), whose signature is unchanged.

### 9.2 Performance & State Consistency Impact
- **Latency:** Multi-tool turns execute all tool calls concurrently inside `futures::future::join_all`, so wall-clock latency equals `max(tool_1, ..., tool_N)` rather than `sum(tool_1, ..., tool_N)`, while eliminating redundant intermediate LLM calls that previously fired on every partial tool completion.
- **State Consistency:** `WorkingMemoryStore` receives all `N` entries before the cognitive loop wakes, and `request.resolved_tools_count` equals `N`, guaranteeing exact tail slicing in `WorkingMemoryProvider::context`.

### 9.3 Documentation Impact
- `synapto/docs/src/ARCHITECTURE.md` (Rule 4 & Rule 5) already describes the high-level `LLMClient` and `ToolExecutor` architecture without coupling to internal method signatures. No high-level documentation changes are required, as this RFC fixes runtime concurrency defects within existing architectural boundaries.

---

## 10. Definition of Done (DoD)

- [x] **`synapto/crates/llm/src/lib.rs`:**
  - [x] In `LLMClient::call_inner`, return `Err(anyhow::anyhow!("LLM returned tool calls unexpectedly"))` when `!tool_calls.is_empty() && !has_tools`.
  - [x] Remove `self.tools_state.execute(...)` and the `ctx` parameter from `LLMClient::call_inner` and `ToolMode`.
  - [x] Remove the `ctx` parameter from `LLMClient<Content, Output, WithTools<Executor>>::call` and add `pub async fn execute_tools(&self, ctx: synapto_interface::context::ContextRequest, tool_calls: Vec<ToolCall>)`.
- [x] **`synapto/crates/core/src/cognitive/types.rs`:**
  - [x] Define `pub(super) type ResolvedToolBatch = Vec<(ToolOutput, ToolCall)>;` and update `RegistryToolExecutor::tool_resolved_tx` to `tokio::sync::mpsc::Sender<ResolvedToolBatch>`.
  - [x] Refactor `RegistryToolExecutor::execute` to spawn a single turn-scoped task that runs all `tool_calls` concurrently via `futures::future::join_all` and sends the resulting `ResolvedToolBatch` in one `tool_resolved_tx.send(batch).await` call.
- [x] **`synapto/crates/core/src/cognitive/direct.rs`:**
  - [x] Update `tool_resolved_rx.recv()` handling to iterate over the `ResolvedToolBatch`, appending each entry to `working_memory_store`, sending each `ToolCallId` to `resolve_in_flight_tool_tx`, and extending `resolved_tools` via `resolved_tools.get_or_insert_with(Vec::new).push(...)`.
  - [x] Update `consecutive_tool_turns` logic so `if resolved_tools.is_some()` increments `consecutive_tool_turns += 1` before checking `else if !new_speech_messages.is_empty()`.
  - [x] Update `llm_client.call(...)` invocation (removing `request` argument) and call `llm_client.execute_tools(request, tool_calls).await` after `process_llm_output(...).await` when the turn produced `Ok(LLMResult::Interrupted(_, tool_calls))`.
- [x] **`synapto/crates/core/src/cognitive/side.rs`:**
  - [x] Update `tool_resolved_rx.recv()` handling to iterate over the `ResolvedToolBatch`, appending each entry to `working_memory_store`, sending each `ToolCallId` to `resolve_in_flight_tool_tx`, and collecting the full batch into `resolved_tools`.
  - [x] Update `llm_client.call(...)` invocation (removing `request` argument) and call `llm_client.execute_tools(request, tool_calls).await` after `process_llm_output(...).await` when the turn produced `Ok(LLMResult::Interrupted(_, tool_calls))`.
- [x] **Unit & Scenario Verification:**
  - [x] Add a unit test in `synapto/crates/core/src/working_memory/store.rs` verifying multi-tool batch tail slicing (`resolved_tools_count = 3`).
  - [x] Remove `#[should_panic(expected = "Scenario Failed")]` from `rfc_2026_10_17_multi_tool_turn_barrier_and_working_memory_alignment` and `rfc_2026_10_17_multi_tool_barge_in_race` in `synapto/crates/test/tests/scenario_tests.rs`.
  - [x] Run `just test-scenarios -p synapto-test rfc_2026_10_17_multi_tool_turn_barrier_and_working_memory_alignment` and `just test-scenarios -p synapto-test rfc_2026_10_17_multi_tool_barge_in_race` in `synapto` and verify that both scenario tests pass cleanly.
  - [x] Rename `synapto/crates/test/tests/scenarios/2026-10-17-multi-tool-turn-barrier-and-working-memory-alignment` to `synapto/crates/test/tests/scenarios/multi-tool-turn-barrier-and-working-memory-alignment` and `synapto/crates/test/tests/scenarios/2026-10-17-multi-tool-barge-in-race` to `synapto/crates/test/tests/scenarios/multi-tool-barge-in-race`, update `name` in both `scenario.yaml` files, rename the test functions in `synapto/crates/test/tests/scenario_tests.rs` to `multi_tool_turn_barrier_and_working_memory_alignment` and `multi_tool_barge_in_race`, and re-run both scenario tests.
- [x] **Documentation Check:**
  - [x] Verify `synapto/docs/src/ARCHITECTURE.md` remains accurate and update `rfcs/README.md` lifecycle state.
