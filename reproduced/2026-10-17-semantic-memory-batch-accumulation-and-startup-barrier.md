# [MVP] [DRAFT] RFC: Semantic Memory Batch Accumulation and Startup Consolidation Barrier

> **Classification:** `Flaw` (Dropped Interactions & Startup Race Condition in `semantic-memory`)
> **Draft Status:** Reproduced (`rfcs/reproduced/`) — Failing scenario test `rfc_2026_10_17_semantic_memory_batch_accumulation_and_startup_barrier` verified in `synapto/crates/test/tests/scenario_tests.rs`; awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-semantic-memory-batch-accumulation-and-startup-barrier.md`.

## 1. User-Centric Motivation & Goal

The semantic memory plugin (`synapto-plugin-semantic-memory`) extracts long-term factual insights and activities from ongoing conversations and runs a consolidation task to resolve unclear interactions.

Currently, two bugs break semantic memory in production:
1. **Interaction Batch Drop Bug:** `insight_memory_task` creates a brand-new `Vec` (`let mut process_interactions = Vec::new();`) inside the `while let Some(interactions) = interaction_rx.recv().await` loop, and then checks `let drain_count: i16 = process_interactions.len() as i16 - 8; if drain_count >= 0`. Because the core sends 1 interaction per turn (`vec![interaction]`), `process_interactions.len()` is always `1`, `drain_count` is `-7`, and **every single interaction is discarded at the end of the loop iteration** without ever extracting insights or advancing the rollout timestamp.
2. **Startup Task Race Condition:** `SemanticMemoryPlugin` implements both `RetrospectiveConsolidationPlugin` and `InteractionObserver`. The core spawns both `.start()` methods concurrently via `tokio::spawn`. `RetrospectiveConsolidationPlugin::start` puts `not_clear_memory_rx` into a `Mutex<Option<...>>`, while `InteractionObserver::start` immediately calls `.take()` on that mutex. Whenever Tokio schedules `InteractionObserver::start` first, the mutex is `None` and `semantic_consolidation_task` is **silently never spawned**.

This RFC fixes interaction accumulation across loop iterations and replaces the racy `Mutex<Option<...>>` with a deterministic `oneshot` barrier.

---

## 2. Executive Summary

- **Context:** `synapto/integrations/plugins/semantic-memory/src/{lib.rs, plugin.rs}`.
- **Motivation:** Restore factual insight extraction on normal turn-by-turn conversations and guarantee `semantic_consolidation_task` always spawns regardless of Tokio task scheduling order.
- **Overarching Goals:** Deterministic startup wiring and lossless interaction buffering in `synapto-plugin-semantic-memory`.
- **Purpose:** Move `process_interactions` outside the `recv()` loop in `insight_memory_task` and synchronize `RetrospectiveConsolidationPlugin::start` with `InteractionObserver::start` via a `tokio::sync::oneshot` channel.
- **Problem:**
  1. **Local Vector Reset on Every `recv()` (`synapto/integrations/plugins/semantic-memory/src/lib.rs:313-363`):** `let mut process_interactions = Vec::new();` is inside the `while` loop (`L317`). Unless a single `mpsc` message contains >= 8 interactions, `drain_count` (`L358`) is negative and the interactions are lost when the loop iteration ends.
  2. **Non-Deterministic Startup Race (`synapto/integrations/plugins/semantic-memory/src/plugin.rs:228-246, 348-372`):** `RetrospectiveConsolidationPlugin::start` (`L231-245`) and `InteractionObserver::start` (`L349-372`) race on `self.not_clear_memory_rx: Arc<Mutex<Option<...>>>` and `self.resolve_not_clear_tx: Arc<Mutex<Option<...>>>`.
- **Solution:**
  1. Declare `let mut process_interactions: Vec<ObservedInteraction> = Vec::new();` outside the `while let Some(interactions) = interaction_rx.recv().await` loop in `insight_memory_task`, and drain processed interactions via `process_interactions.drain(0..=drain_count as usize)` after extraction succeeds.
  2. Use a `tokio::sync::oneshot` channel pair (`consolidation_init_tx`, `consolidation_init_rx`) on `SemanticMemoryPlugin` (matching how `rollout_tx` is already handed over at `plugin.rs:175, 257, 315`) so `InteractionObserver::start` awaits the channels from `RetrospectiveConsolidationPlugin::start` deterministically.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/integrations/plugins/semantic-memory/src/lib.rs:313-363`**:
   ```rust
   while let Some(interactions) = interaction_rx.recv().await {
       if interactions.is_empty() {
           continue;
       }
       let mut process_interactions = Vec::new();
       for interaction in interactions {
           process_interactions.push(interaction);
       }
       ...
       let drain_count: i16 = process_interactions.len() as i16 - 8;
       if drain_count >= 0 {
           let to_process: Vec<ObservedInteraction> = process_interactions
               .iter()
               .take((drain_count + 1) as usize)
               .cloned()
               .collect();
           ...
       }
   }
   ```
2. **`synapto/integrations/plugins/semantic-memory/src/plugin.rs:228-246, 348-372`**:
   ```rust
   // L231-245 (RetrospectiveConsolidationPlugin::start):
   if let Ok(mut guard) = self.not_clear_memory_rx.lock() {
       *guard = Some(not_clear_memory_rx);
   }
   if let Ok(mut guard) = self.resolve_not_clear_tx.lock() {
       *guard = Some(resolve_not_clear_tx);
   }
   // L349-372 (InteractionObserver::start):
   let not_clear_rx = self.not_clear_memory_rx.lock().ok().and_then(|mut g| g.take());
   let resolve_tx = self.resolve_not_clear_tx.lock().ok().and_then(|mut g| g.take());
   if let (Some(not_clear_rx), Some(resolve_tx)) = (not_clear_rx, resolve_tx) {
       tokio::spawn(async move {
           semantic_consolidation_task(...).await;
       });
   }
   ```

---

## 4. Structural Hierarchy

- **Mission:** Lossless Semantic Insight Extraction and Deterministic Consolidation Startup.
- **Goal:** Ensure every interaction is buffered until processed and `semantic_consolidation_task` always starts when registered.
- **Atomic Responsibilities:**
  - **`insight_memory_task` (`semantic-memory/src/lib.rs`):** Accumulates `ObservedInteraction`s across `recv()` calls into a persistent buffer and drains processed items after updating storage and advancing `insight_interaction_rollout_tx`.
  - **`SemanticMemoryPlugin` (`semantic-memory/src/plugin.rs`):** Synchronizes `RetrospectiveConsolidationPlugin::start` and `InteractionObserver::start` without race conditions.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 20 (Encapsulated Task Orchestration):** Internal plugin task spawning must be deterministic and immune to external caller task scheduling order.

### 5.2 Handover Struct
```rust
// synapto/integrations/plugins/semantic-memory/src/plugin.rs
struct ConsolidationChannels {
    not_clear_memory_rx: watch::Receiver<NotClearInteractionMemory>,
    resolve_not_clear_tx: mpsc::Sender<Timestamp>,
}
```

### 5.3 JSON Contract Examples

#### Compiled Prompt Context Input (`context_plugins.semantic_memory`)
```json
{
  "context_plugins": {
    "semantic_memory": {
      "activities": [
        {
          "id": 1,
          "name": "Architecture Audit",
          "description": "Reviewing Synapto core and plugin crates."
        }
      ],
      "insights": [
        {
          "id": 10,
          "activity_id": 1,
          "content": "Every RFC in the repository must address a single specific problem."
        }
      ]
    }
  }
}
```

#### Insight Extraction LLM Structured Output (`InsightLLMOutput`)
```json
{
  "reasoning": "Extracting the new architectural rule established by the user.",
  "new_insights": [
    {
      "activity_id": 1,
      "content": "Every RFC in the repository must address a single specific problem."
    }
  ],
  "obsoleted_insight_ids": [],
  "not_clear_interaction_timestamps": []
}
```

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For spawning `semantic_consolidation_task` in `synapto/integrations/plugins/semantic-memory/src/plugin.rs`:

**[ Option A: Spawn `semantic_consolidation_task` Directly Inside `RetrospectiveConsolidationPlugin::start` ]**
- **Action:** Share the initialized in-memory state handles (`Arc<Mutex<...>>` or channels for `insight_memory_tx`, `activity_memory_rx`, `store`, `consolidation_llm_client`) on `SemanticMemoryPlugin` and spawn `semantic_consolidation_task` directly inside `RetrospectiveConsolidationPlugin::start` when the core calls it, removing `not_clear_memory_rx` and `resolve_not_clear_tx` from `InteractionObserver::start` altogether.
- **Pros:** Each trait's `start()` method spawns its own corresponding task independently; if `RetrospectiveConsolidationPlugin` is not registered, `InteractionObserver::start` never blocks waiting for a oneshot that won't fire.
- **Cons:** Requires sharing the `insight_memory` / `activity_memory` state handles between the two tasks on `SemanticMemoryPlugin`.

**[ Option B: Hand Over `ConsolidationChannels` via `tokio::sync::oneshot` in `InteractionObserver::start` ]**
- **Action:** Pass `ConsolidationChannels` from `RetrospectiveConsolidationPlugin::start` to `InteractionObserver::start` over a `oneshot` channel (matching `rollout_tx` at `plugin.rs:175`).
- **Pros:** Minimal structural change to `SemanticMemoryPlugin` and `semantic_consolidation_task` arguments.
- **Cons:** Requires `RetrospectiveConsolidationPlugin` to always be registered whenever `InteractionObserver` is registered on `SemanticMemoryPlugin` (which `Plugin::register` at `plugin.rs:219-225` always does).

**My Recommendation:** I recommend **Option B** for MVP because `SemanticMemoryPlugin::register` (`plugin.rs:219-225`) unconditionally registers both traits together and already uses the exact same `oneshot` pattern for `rollout_tx`.
Please select Option A or Option B for this RFC.
