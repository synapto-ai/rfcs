# Retrospective: Multi-Tool Turn Barrier and Working Memory Alignment

- **RFC:** [`implemented/2026-10-17-multi-tool-turn-barrier-and-working-memory-alignment.md`](../implemented/2026-10-17-multi-tool-turn-barrier-and-working-memory-alignment.md)
- **Date:** `2026-10-19`
- **Classification:** `Flaw`

## 1. Architectural Evolution & Key Decisions

### 1.1 Turn-Scoped Batch Barrier (`ResolvedToolBatch`)
- **Initial State:** `RegistryToolExecutor::execute` spawned an independent task for each tool call in a turn and sent individual `(ToolOutput, ToolCall)` items over `tool_resolved_tx`. Any multi-tool turn woke the cognitive loop on the fastest tool completion while slower sibling tools were still running, and overwritten `resolved_tools` assignments in `direct.rs` caused lost tool responses and misaligned `WorkingMemoryStore` tail slicing.
- **Implemented Resolution (Main Agent):** `RegistryToolExecutor::execute` now spawns a single turn-coordinator task that executes all tool calls from the turn concurrently via `futures::future::join_all` (retaining the 60-second per-tool timeout) and sends `ResolvedToolBatch` (`Vec<(ToolOutput, ToolCall)>`) as one atomic message over `tool_resolved_tx`.

### 1.2 Post-Commit Tool Dispatch (Option B vs. Option A)
- **Rejected Approach (Option A):** Keeping `self.tools_state.execute(...)` inside `LLMClient::call_inner` and attempting to drain stale tool batches from `tool_resolved_rx` after a barge-in cancellation. Rejected because background tools would still be spawned for a discarded turn and fast/error tool completions could send `resolve_in_flight_tool_tx` before `process_llm_output` sent the `Interaction` containing `in_flight_tools` to `new_interaction_tx`.
- **Selected Approach (Option B — Lead Architect Selection):** Removed automatic tool execution from `LLMClient::call_inner` (`synapto-llm`) and exposed `LLMClient::execute_tools(&self, ctx, tool_calls)`. Both `cognitive_direct_task` and `cognitive_side_task` now invoke `llm_client.execute_tools(request, tool_calls).await` only after `tokio_select!` completes without barge-in interruption and `process_llm_output` records the `Interaction` on `new_interaction_tx`.

### 1.3 Separate Scenario Tests for Multi-Tool Barrier and Barge-In Race
- **Direction (Lead Architect):** Rather than combining the `user_says` barge-in race test into the existing `user_writes` multi-tool barrier scenario, we maintained two independent scenario tests:
  1. `multi_tool_turn_barrier_and_working_memory_alignment` (`tests/scenarios/multi-tool-turn-barrier-and-working-memory-alignment/scenario.yaml`) testing parallel multi-tool turn barrier resolution and working memory alignment on the side cognitive loop.
  2. `multi_tool_barge_in_race` (`tests/scenarios/multi-tool-barge-in-race/scenario.yaml`) testing speech barge-in interruption (`speech_detected.notify()`) coupled with parallel multi-tool execution on the direct cognitive loop.

## 2. Attribution
- **Lead Architect:** Selected Option B at the Phase 2 Architectural Fork Checkpoint and directed separating the barge-in race scenario into its own dedicated scenario test rather than replacing the original `user_writes` scenario.
- **Main Agent:** Designed and implemented `ResolvedToolBatch`, `LLMClient::execute_tools`, `WorkingMemoryStore` batch alignment, `consecutive_tool_turns` priority ordering, and the unit and scenario tests.
- **Review Subagent:** Conducted the zero-context Phase 3 architectural verification against `synapto/docs/src/ARCHITECTURE.md`.

## 3. Process Observations
- When an RFC covers multiple distinct execution modalities (such as `cognitive_side_task` via `user_writes` and `cognitive_direct_task` barge-in via `user_says`), keeping each modality in its own atomic scenario test isolates failure modes and prevents one test from masking regressions in the other.
