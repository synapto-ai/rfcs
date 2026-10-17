# [MVP] [DRAFT] RFC: Deterministic In-Flight Tool Resolution Synchronization and Non-Blocking Observer Fan-Out

> **Classification:** `Flaw` (Deadlock, Race Condition & Head-of-Line Blocking in Interaction Memory)
> **Draft Status:** Reproduced (`rfcs/reproduced/`) — Failing scenario test `rfc_2026_10_17_deterministic_in_flight_tool_resolution_sync` verified in `synapto/crates/test/tests/scenario_tests.rs`; awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-deterministic-in-flight-tool-resolution-sync.md`.

## 1. User-Centric Motivation & Goal

When a tool finishes executing, the cognitive loop updates `interaction_memory` to mark that tool as resolved before calling the LLM with the tool's output.

Currently, `cognitive_side_task` sends the `ToolCallId` to `interaction_memory_task` and then loops forever on `interaction_memory_rx.changed().await` waiting for the ID to appear in `resolved_tools`. If the interaction was rolled out or missed, `interaction_memory_task` logs a warning and never sends on `interaction_memory_tx`, causing `cognitive_side_task` to **deadlock forever**. Conversely, `cognitive_direct_task` sends `ToolCallId` and does not wait at all, racing against `interaction_memory_task` and feeding stale `in_flight_tools` to the LLM. Additionally, `interaction_memory_task` dispatches new interactions to all `InteractionObserver` plugins sequentially via `.send().await`, meaning a single slow observer plugin stalls `interaction_memory_task` and blocks tool resolution across the entire system.

This RFC replaces the unbounded watch-channel polling with a deterministic Oneshot Return synchronization (Rule 18) and eliminates head-of-line blocking in observer fan-out.

---

## 2. Executive Summary

- **Context:** `synapto/crates/core/src/interactions/recent.rs`, `cognitive/side.rs`, and `cognitive/direct.rs` manage recent interaction state, in-flight tool tracking, and observer notifications.
- **Motivation:** Eliminate the infinite wait deadlock in `cognitive_side_task`, the data race in `cognitive_direct_task`, and head-of-line observer blocking in `interaction_memory_task`.
- **Overarching Goals:** Achieve deterministic, deadlock-free state synchronization between the cognitive loops and `interaction_memory_task` using the Oneshot Return Pattern (Rule 18).
- **Purpose:** Refactor `resolve_in_flight_tool_tx` to carry a `oneshot::Sender<()>` acknowledgement, persist `in_flight_tools`/`resolved_tools` across serialization, remove legacy backward-compatibility fallback code in `recent.rs`, and make observer fan-out non-blocking.
- **Problem:**
  1. **Infinite Deadlock in `cognitive_side_task` (`crates/core/src/cognitive/side.rs:265-287` & `interactions/recent.rs:236-262`):** If `resolve_in_flight_tool` does not find `tool_call_id`, it logs a warning (`recent.rs:259`) and never updates `interaction_memory_tx`, causing `while let Ok(()) = rx.changed().await` (`side.rs:275`) to hang forever.
  2. **Race Condition in `cognitive_direct_task` (`crates/core/src/cognitive/direct.rs:372-383, 425`):** Sends `ToolCallId` to `resolve_in_flight_tool_tx` without waiting for `interaction_memory_task` to process it before borrowing `interaction_memory_rx` at `direct.rs:425`.
  3. **Sequential Observer Head-of-Line Blocking (`crates/core/src/interactions/recent.rs:339-346`):** Loops over `observers_tx` awaiting `tx.send(vec![interaction.clone()]).await` sequentially. If one observer's channel fills, `interaction_memory_task` freezes.
  4. **Speaking Semaphore Contention (`crates/core/src/cognitive/direct.rs:124-135, 718` vs `speaking_coordinator.rs:47`):** `cognitive_direct_task` holds `_cycle_permit` on `cognitive_speaking_semaphore` while sending `CognitiveOutputSpeech` and executing side-effect commands, blocking `speaking_coordinator` when the first TTS audio chunk arrives early.
- **Solution:**
  1. Change `resolve_in_flight_tool_tx` to `mpsc::Sender<ResolveInFlightToolsRequest>` containing `call_ids: Vec<ToolCallId>` and `reply_tx: oneshot::Sender<()>` (Rule 18). Both `cognitive_direct_task` and `cognitive_side_task` simply `reply_rx.await`, which completes deterministically whether each ID was found or already rolled out.
  2. Drop `_cycle_permit` in `cognitive_direct_task` immediately before or right after dispatching `CognitiveOutputSpeech` so `speaking_coordinator` never deadlocks or races for the permit.
  3. Dispatch observer notifications concurrently or via `try_send` / spawned send so a slow observer never blocks `interaction_memory_task`.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/crates/core/src/cognitive/side.rs:224-231, 265-287`**:
   ```rust
   let mut rx = interaction_memory_rx.clone();
   while let Ok(()) = rx.changed().await {
       let all_resolved = {
           let mem = rx.borrow();
           expected_ids.iter().all(|id| {
               mem.iter().any(|interaction| interaction.resolved_tools.contains(id))
           })
       };
       if all_resolved {
           break;
       }
   }
   ```
2. **`synapto/crates/core/src/interactions/recent.rs:236-262`**:
   ```rust
   async fn resolve_in_flight_tool(...) {
       let mut found = false;
       for interaction in interaction_memory.iter_mut().rev() { ... }
       if found {
           interaction_memory_tx.send(interaction_memory.clone())...;
       } else {
           tracing::warn!("Could not find in-flight tool call ID {}...", tool_call_id);
       }
   }
   ```
3. **`synapto/crates/core/src/interactions/recent.rs:339-346`**:
   ```rust
   for tx in &observers_tx {
       if let Err(e) = tx.send(vec![interaction.clone()]).await {
           tracing::error!("{}", e);
       }
   }
   ```

---

## 4. Structural Hierarchy

- **Mission:** Deadlock-Free State Synchronization in `interaction_memory_task`.
- **Goal:** Guarantee bounded, deterministic completion of tool resolution updates and non-blocking observer fan-out.
- **Atomic Responsibilities:**
  - **`ResolveInFlightToolsRequest` (`crates/core/src/interactions/recent.rs`):** Transitions all matching `ToolCallId`s from `in_flight_tools` to `resolved_tools`, updates `interaction_memory_tx` once for the batch, and always sends `()` on `reply_tx`.
  - **`ObserverDispatcher` (`crates/core/src/interactions/recent.rs`):** Sends new interactions to observer channels without blocking `interaction_memory_task`'s primary `tokio_select!` loop.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 18 (Oneshot Return Pattern):** Use `reply_tx: oneshot::Sender<()>` on `ResolveInFlightToolsRequest` instead of polling a shared `watch` channel for expected IDs.

### 5.2 Request Schema
```rust
// synapto/crates/core/src/interactions/recent.rs
pub struct ResolveInFlightToolsRequest {
    pub call_ids: Vec<ToolCallId>,
    pub reply_tx: tokio::sync::oneshot::Sender<()>,
}
```

### 5.3 JSON Contract Examples

#### Compiled Prompt Context Input (`interaction_memory`)
```json
{
  "interaction_memory": [
    {
      "timestamp": "2026-10-17T10:00:00Z",
      "in_flight_tools": [],
      "resolved_tools": ["call_01", "call_02"]
    }
  ]
}
```

#### LLM Structured Output Command Payload (`CognitiveDirectCommands`)
```json
{
  "reasoning": "All in-flight tools are marked resolved.",
  "commands": {
    "say": "I have checked both files for you."
  }
}
```

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For dispatching `ObservedInteraction` to `observers_tx` in `synapto/crates/core/src/interactions/recent.rs:339-346`:

**[ Option A: Concurrent `join_all` Send Across Observers ]**
- **Action:** Await `futures::future::join_all(observers_tx.iter().map(|tx| tx.send(vec![interaction.clone()])))` so one slow observer does not serialize behind another, while still applying backpressure if an observer's 100-item buffer is full.
- **Pros:** Preserves guaranteed delivery of every interaction to every memory observer without spawning detached tasks per turn.
- **Cons:** If an observer permanently deadlocks and fills its 100-item queue, backpressure still propagates.

**[ Option B: Non-Blocking `try_send` with Overflow Warning Log ]**
- **Action:** Use `tx.try_send(vec![interaction.clone()])` (or increase buffer size to 512) and log `tracing::error!` if an observer's queue is full, guaranteeing `interaction_memory_task` can never be blocked by a misbehaving observer plugin.
- **Pros:** Guarantees 100% isolation of the core cognitive loops from slow or stalled observer plugins.
- **Cons:** A severely stalled observer (>512 turns behind) drops an observation instead of stalling the core.

**My Recommendation:** I recommend **Option B** (with a 512-item channel buffer) so a stalled background observer plugin can never freeze live cognitive loops.
Please select Option A or Option B for this RFC.
