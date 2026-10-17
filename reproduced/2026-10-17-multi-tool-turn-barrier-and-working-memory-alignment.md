# [MVP] [DRAFT] RFC: Multi-Tool Turn Barrier and Working Memory Alignment

> **Classification:** `Flaw` (Concurrency Race & Data Loss in Tool Execution)
> **Draft Status:** Reproduced (`rfcs/reproduced/`) — Failing scenario test `rfc_2026_10_17_multi_tool_turn_barrier_and_working_memory_alignment` implemented in `synapto/crates/test/tests/scenario_tests.rs`; awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-multi-tool-turn-barrier-and-working-memory-alignment.md`.

## 1. User-Centric Motivation & Goal

When a user asks a question that requires the AI to call multiple tools in the same turn (such as reading two attached documents or querying two MCP tools), the AI must wait for all requested tools in that turn to complete before generating its answer, and it must never lose any tool result or hang forever when it hits its tool-call limit.

Currently, `RegistryToolExecutor::execute` spawns a separate task for each tool call that sends its result individually to `tool_resolved_tx`. The cognitive loops (`cognitive_direct_task` and `cognitive_side_task`) wake up as soon as the **first** tool finishes and immediately invoke the LLM while the remaining tools from the same turn are still running. Worse, if two tools finish while `cognitive_direct_task` is waiting for speech playback to finish, the second tool overwrites `resolved_tools`, permanently dropping the first tool's response and corrupting `WorkingMemoryStore` indexing. Finally, if the LLM emits a tool call when `max_tool_turns` has been reached, `synapto-llm` returns `Ok(LLMResult::Interrupted(None, vec![]))` with zero tools, causing the cognitive loop to wait on `tool_resolved_rx` forever.

This RFC introduces an atomic turn-scoped barrier for multi-tool execution, aligns `WorkingMemoryStore` with batch resolution, and fixes the `max_tool_turns` hang and counter reset bugs.

---

## 2. Executive Summary

- **Context:** `synapto` (`crates/core/src/cognitive/{types.rs, direct.rs, side.rs}`, `working_memory/store.rs`) and `synapto-llm` (`crates/llm/src/lib.rs`) orchestrate native LLM tool calling across turns.
- **Motivation:** Concurrent tool calls within a single LLM turn currently race against each other, corrupt working memory slice offsets, and can deadlock the cognitive loop when `max_tool_turns` is reached.
- **Overarching Goals:** Guarantee that all tool calls emitted in a single LLM turn execute concurrently and resolve as a single atomic batch before waking the cognitive loop.
- **Purpose:** Fix multi-tool turn coordination, `WorkingMemoryStore` offset alignment, `consecutive_tool_turns` counter tracking, barge-in cancellation tool cleanup, and `max_tool_turns` error handling.
- **Problem:**
  1. **Premature Wakeup on Multi-Tool Turns (`crates/core/src/cognitive/types.rs:50-143`, `direct.rs:347`, `side.rs:202`):** `RegistryToolExecutor::execute` spawns an independent task per tool call sending `(ToolOutput, ToolCall)` over `tool_resolved_tx`. Both cognitive loops wake on the first single message and trigger an LLM inference before sibling tools finish.
  2. **Overwritten `resolved_tools` & Corrupted `WorkingMemoryStore` Slice (`crates/core/src/cognitive/direct.rs:364-367` & `working_memory/store.rs:137-141`):** While `cognitive_direct_task` waits for `cognitive_speaking_semaphore` (`run_after_unpaused = true`), each arriving tool appends to `working_memory`, but `resolved_tools = Some(vec![(tool_call.clone(), doc_text)])` (`direct.rs:367`) overwrites the vector instead of appending. `WorkingMemoryProvider::context` (`working_memory/store.rs:139`) then slices `mem.len().saturating_sub(request.resolved_tools_count)` with `resolved_tools_count = 1`, misclassifying current-turn tools as historical working memory and losing their `ToolResponse` messages.
  3. **Infinite Hang When LLM Emits Tool Calls at `max_tool_turns` (`crates/llm/src/lib.rs:263-267` & `direct.rs:641`, `side.rs:457`):** When `consecutive_tool_turns >= max_tool_turns`, `dynamic_tools` is empty (`has_tools = false`). If the LLM still outputs a tool call, `crates/llm/src/lib.rs:265` returns `Tools::interrupted(text_content, vec![])` (`Ok(LLMResult::Interrupted(None, vec![]))`) without executing any tool, hanging the cognitive loop forever on `tool_resolved_rx`.
  4. **`consecutive_tool_turns` Counter Reset Bug (`crates/core/src/cognitive/direct.rs:398-402`):** If a user speech message arrives while a tool resolves during speaking, `!new_speech_messages.is_empty()` resets `consecutive_tool_turns = 0` on a tool turn.
  5. **Barge-In Cancellation Orphan Tool Execution (`crates/core/src/cognitive/direct.rs:659-675` & `crates/llm/src/lib.rs:270-272`):** Because `self.tools_state.execute(tool_calls)` is called inside `LLMClient::call_inner` before returning `LLMResult::Interrupted`, if `interrupt.interrupted()` wins the `tokio_select!` at the exact boundary or before the interaction is recorded, background tool tasks are already spawned for a discarded turn.
- **Solution:**
  1. Spawn a single turn-coordinator task in `RegistryToolExecutor::execute` that runs all `tool_calls` concurrently via `futures::future::join_all` and sends the complete `Vec<(ToolOutput, ToolCall)>` batch over `tool_resolved_tx`.
  2. In `direct.rs` and `side.rs`, accumulate `resolved_tools` via `.extend(...)` rather than overwriting with `Some(vec![...])`, and only reset `consecutive_tool_turns = 0` when `resolved_tools.is_none()`.
  3. In `synapto-llm/src/lib.rs:265`, return `Err(LLMError::UnexpectedToolCalls)` when `!tool_calls.is_empty() && !has_tools`.
  4. Move `tools_state.execute(tool_calls)` out of `LLMClient::call_inner` or trigger it in the cognitive loop only after the turn's `Interaction` is committed (not discarded by barge-in).

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/crates/core/src/cognitive/types.rs:50-143`**:
   ```rust
   fn execute(&self, tool_calls: Vec<ToolCall>) {
       for call in tool_calls {
           ...
           tokio::spawn(async move {
               ...
               tx.send((output, call_clone)).await
           });
       }
   }
   ```
2. **`synapto/crates/core/src/cognitive/direct.rs:347-388, 398-402`**:
   ```rust
   Some((output, tool_call)) = tool_resolved_rx.recv() => {
       ...
       working_memory.append(tool_call.call_id.clone(), tool_call.fn_name.clone(), args_str, doc_text.clone()).await;
       resolved_tools = Some(vec![(tool_call.clone(), doc_text)]);
       ...
   }
   // L398:
   if !new_speech_messages.is_empty() {
       consecutive_tool_turns = 0;
   }
   ```
3. **`synapto/crates/core/src/working_memory/store.rs:137-141`**:
   ```rust
   let mem = self.store.inner.read().await;
   let end_idx = mem.len().saturating_sub(request.resolved_tools_count);
   let slice = &mem[..end_idx];
   ```
4. **`synapto/crates/llm/src/lib.rs:263-274`**:
   ```rust
   if !tool_calls.is_empty() {
       if !has_tools {
           tracing::error!("LLM returned tool calls unexpectedly");
           return Tools::interrupted(text_content, vec![]);
       }
       self.tools_state.execute(tool_calls.clone());
       return Tools::interrupted(text_content, tool_calls);
   }
   ```

---

## 4. Structural Hierarchy

- **Mission:** Atomic Multi-Tool Turn Execution and Deterministic Working Memory State.
- **Goal:** Ensure 1:1 batch alignment between LLM tool calls, `tool_resolved_tx` delivery, and `WorkingMemoryStore` entries.
- **Atomic Responsibilities:**
  - **`RegistryToolExecutor::execute` (`crates/core/src/cognitive/types.rs`):** Spawns one task per turn that awaits `futures::future::join_all` across all `tool_calls` in that turn and sends `Vec<(ToolOutput, ToolCall)>` as a single message over `tool_resolved_tx`.
  - **`LLMClient::call_inner` (`crates/llm/src/lib.rs`):** Returns `Err` if `!tool_calls.is_empty() && !has_tools`.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 4 & 5 (Native Tool Calling & Generic Tool Architecture):** Tool calls within a turn must resolve cleanly into `ToolResponse` messages without context corruption or infinite loops.

### 5.2 Batch Channel Type
```rust
// synapto/crates/core/src/cognitive/types.rs
pub type ResolvedToolBatch = Vec<(synapto_llm::ToolOutput, synapto_interface::llm::ToolCall)>;
```

### 5.3 JSON Contract Examples

#### Multi-Tool Resolution Turn Input
```json
{
  "resolved_tools": [
    {
      "call_id": "call_1",
      "fn_name": "read_document",
      "output": "{\"content\":\"Doc A contents\"}"
    },
    {
      "call_id": "call_2",
      "fn_name": "read_document",
      "output": "{\"content\":\"Doc B contents\"}"
    }
  ]
}
```

#### LLM Structured Output Command Payload (`CognitiveSideCommands`)
```json
{
  "reasoning": "Both Doc A and Doc B have been retrieved.",
  "commands": {
    "write": {
      "content": "Comparison of Doc A and Doc B...",
      "target": { "plugin": "google_chat", "context": "{}" }
    }
  }
}
```

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For handling where `ToolExecutor::execute(tool_calls)` is invoked (`synapto/crates/llm/src/lib.rs:271` vs. cognitive loops):

**[ Option A: Keep `ToolExecutor::execute` in `LLMClient` Using Turn-Scoped `join_all` Batching ]**
- **Action:** Keep `self.tools_state.execute(tool_calls)` in `LLMClient::call_inner`, change `RegistryToolExecutor::execute` to spawn a single `join_all` batch task sending `Vec<(ToolOutput, ToolCall)>`, and if `cognitive_direct_task` is interrupted by barge-in during `llm_client.call`, drain/discard any stale batch in `tool_resolved_rx` when starting the next turn.
- **Pros:** Preserves the `ToolExecutor` trait workflow in `synapto-llm` with minimal API surface changes.
- **Cons:** Requires draining `tool_resolved_rx` on barge-in cancellation.

**[ Option B: Trigger Tool Execution Explicitly in the Cognitive Loop After Barge-In Check ]**
- **Action:** Have `LLMClient::call` return `LLMResult::Interrupted(text_content, tool_calls)` *without* calling `self.tools_state.execute(...)` inside `synapto-llm`, and invoke `tool_executor.execute(tool_calls)` in `direct.rs` / `side.rs` only after the `tokio_select!` completes without barge-in interruption.
- **Pros:** Impossible for a barge-in cancelled LLM call to spawn orphan tool executions.
- **Cons:** Moves the `execute(tool_calls)` call site from `LLMClient` to `direct.rs` and `side.rs`.

**My Recommendation:** I recommend **Option B** because it completely eliminates the barge-in cancellation race by never spawning tool executions if the LLM call was interrupted.
Please select Option A or Option B for this RFC.
