# [MVP] [DRAFT] RFC: Episodic Memory Persistence and Rollout Controller Registration

> **Classification:** `Flaw` (Complete Loss of Episodic Memory Persistence & Unwired Rollout)
> **Draft Status:** Reproduced (`rfcs/reproduced/`) — Failing scenario test `rfc_2026_10_17_episodic_memory_persistence_and_rollout_registration` verified in `synapto/crates/test/tests/scenario_tests.rs`; awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-episodic-memory-persistence-and-rollout-registration.md`.

## 1. User-Centric Motivation & Goal

Users rely on the episodic memory plugin (`synapto-plugin-episodic-memory`) to summarize conversations into historical Sessions, Progressions, and Continuums that persist across assistant restarts, while allowing old raw interactions to roll out of the active context window once summarized.

Currently, `synapto-plugin-episodic-memory` **never writes a single record to the database**: `store.upsert_record(...)` is completely missing from `session.rs`, `progression.rs`, and `continuum.rs`, and all three files call `store.trim_records_before(..., "REPLACE_ME_TODO")`. Furthermore, `EpisodicMemoryPlugin` never implements `RolloutController` or calls `registry.register_rollout_controller(...)`, meaning the core switchboard ignores its rollout receiver.

This RFC implements the missing database upserts, replaces `"REPLACE_ME_TODO"` with the actual retention window cutoff key, and registers `EpisodicMemoryPlugin` as a `RolloutController`.

---

## 2. Executive Summary

- **Context:** `synapto/integrations/plugins/episodic-memory/src/{lib.rs, session.rs, progression.rs, continuum.rs}`.
- **Motivation:** Episodic memory currently operates exclusively in volatile RAM, attempts to trim records using a literal `"REPLACE_ME_TODO"` string on every turn, and fails to wire its rollout channel into the core switchboard.
- **Overarching Goals:** Make episodic memory durable across restarts and properly integrated with the core interaction rollout lifecycle.
- **Purpose:** Add `store.upsert_record` calls for created/updated sessions, progressions, and continuums, compute real cutoff keys for `store.trim_records_before`, and implement/register `RolloutController` on `EpisodicMemoryPlugin`.
- **Problem:**
  1. **Missing `upsert_record` in `session.rs:280-324`:** Mutates `session_memory` in RAM on `NewSession` and `UpdatedSession`, never calls `store.upsert_record("sessions", ...)`, and calls `store.trim_records_before("sessions", "REPLACE_ME_TODO").await` (`L319-324`).
  2. **Missing `upsert_record` in `progression.rs:200-237`:** Mutates `progression_memory` in RAM on `NewProgression` and `UpdatedProgression`, never calls `store.upsert_record("progressions", ...)`, and calls `store.trim_records_before("progressions", "REPLACE_ME_TODO").await` (`L232-237`).
  3. **Missing `upsert_record` in `continuum.rs:186-223`:** Mutates `continuum_memory` in RAM on `NewContinuum` and `UpdatedContinuum`, never calls `store.upsert_record("continuums", ...)`, and calls `store.trim_records_before("continuums", "REPLACE_ME_TODO").await` (`L218-223`).
  4. **Unwired `RolloutController` in `lib.rs:126, 130, 143-144, 174-182`:** `EpisodicMemoryPlugin` exposes `pub session_interaction_rollout_rx: watch::Receiver<Timestamp>` as a struct field instead of implementing `RolloutController` and calling `registry.register_rollout_controller(self.clone())`.
- **Solution:**
  1. Call `store.upsert_record` using the record's RFC3339 `started_at` timestamp string as the key whenever a `Session`, `Progression`, or `Continuum` is created or updated.
  2. When trimming in-memory vectors to their retention window (`session_memory`, `progression_memory`, `continuum_memory`), pass the `started_at.to_string()` of the first (oldest retained) record in the window to `store.trim_records_before`.
  3. Implement `RolloutController` for `EpisodicMemoryPlugin` and register it in `Plugin::register`.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/integrations/plugins/episodic-memory/src/session.rs:280-324`**:
   ```rust
   SessionOperation::NewSession(new_session) => {
       tracing::info!("Creating new session: {:?}", new_session);
       session_memory.push(new_session);
   }
   SessionOperation::UpdatedSession(updated_session) => {
       tracing::info!("Updating session: {:?}", updated_session);
       if let Some(last_session) = session_memory.last_mut() {
           *last_session = updated_session;
       } else {
           session_memory.push(updated_session);
       }
   }
   ...
   if let Err(e) = store
       .trim_records_before("sessions", "REPLACE_ME_TODO")
       .await
   {
       tracing::error!("Failed to write session memory: {:?} at line {}", e, line!());
   }
   ```
2. **`synapto/integrations/plugins/episodic-memory/src/progression.rs:200-237`**:
   Identical pattern calling `store.trim_records_before("progressions", "REPLACE_ME_TODO").await` at `L232-237` with zero `upsert_record` calls.
3. **`synapto/integrations/plugins/episodic-memory/src/continuum.rs:186-223`**:
   Identical pattern calling `store.trim_records_before("continuums", "REPLACE_ME_TODO").await` at `L218-223` with zero `upsert_record` calls.
4. **`synapto/integrations/plugins/episodic-memory/src/lib.rs:124-182`**:
   `Plugin::register` (`L174-182`) registers `InteractionObserver` and three `ContextProvider`s, omitting `registry.register_rollout_controller(self.clone())`.

---

## 4. Structural Hierarchy

- **Mission:** Durable Episodic Memory Hierarchy (`Interaction -> Session -> Progression -> Continuum`).
- **Goal:** Persist all episodic tiers to `RecordStore` and participate in core interaction rollout.
- **Atomic Responsibilities:**
  - **`EpisodicMemoryPlugin` (`RolloutController`):** Transfers ownership of `session_interaction_rollout_rx` to the core switchboard via `RolloutController::start`.
  - **`session_memory_task` / `progression_memory_task` / `continuum_memory_task`:** Upsert new/updated records by their `started_at` key and trim records older than the first retained element's `started_at`.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 18 & Section 6 (Inversion of Control / Channel Handover):** Rollout receivers must be registered via `RolloutController`, never exposed as public struct fields on plugins.

### 5.2 Record Key Convention
Each `Session`, `Progression`, and `Continuum` uses its `started_at: Timestamp` (formatted via `.to_string()`, which is lexicographically ordered ISO-8601 / RFC3339 UTC) as the `RecordStore` primary key:
```rust
let key = record.started_at.to_string();
store.upsert_record("sessions", &key, record.clone()).await?;
if let Some(oldest_retained) = session_memory.first() {
    store.trim_records_before("sessions", &oldest_retained.started_at.to_string()).await?;
}
```

### 5.3 JSON Contract Examples

#### Compiled Prompt Context Input (`context_plugins.session_memory`)
```json
{
  "context_plugins": {
    "session_memory": {
      "sessions": [
        {
          "started_at": "2026-10-17T08:00:00Z",
          "ended_at": "2026-10-17T08:15:00Z",
          "summary": "Discussed architecture review findings."
        }
      ]
    }
  }
}
```

#### Session LLM Structured Output Payload (`SessionLLMOutput`)
```json
{
  "reasoning": "Summarizing the completed interaction batch into the active session.",
  "session_operation": {
    "UpdatedSession": {
      "started_at": "2026-10-17T08:00:00Z",
      "ended_at": "2026-10-17T08:25:00Z",
      "summary": "Discussed architecture review findings and created 18 atomic RFCs."
    }
  }
}
```

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For handling `UpdatedSession` / `UpdatedProgression` / `UpdatedContinuum` when the LLM returns a modified `started_at` timestamp:

**[ Option A: Preserve Original `started_at` Key on `Updated*` Operations ]**
- **Action:** When applying `UpdatedSession` (or `UpdatedProgression` / `UpdatedContinuum`) to `last_mut()`, preserve the existing `last_session.started_at` as the immutable primary key and update `ended_at` and `summary`, then call `store.upsert_record("sessions", &last_session.started_at.to_string(), ...)`.
- **Pros:** Prevents orphaned database records if the LLM slightly alters `started_at` during an update operation, guaranteeing the upsert overwrites the exact existing record in `RecordStore`.
- **Cons:** Ignores any LLM change to `started_at` on an update operation (which is semantically immutable once a session starts).

**[ Option B: Delete Old Key and Insert New Key If `started_at` Changed ]**
- **Action:** Compare `last_session.started_at` with `updated_session.started_at`; if different, call `store.delete_record` on the old key before calling `store.upsert_record` on the new key.
- **Pros:** Allows the LLM to rewrite `started_at`.
- **Cons:** Extra database delete call and non-atomic key migration if the LLM hallucinates a timestamp change.

**My Recommendation:** I recommend **Option A** because a session's start time is immutable once created, and preserving `started_at` guarantees idempotent single-key upserts.
Please select Option A or Option B for this RFC.
