# [MVP] [DRAFT] RFC: Error Visibility, Workspace Lint Compliance, and Architecture Documentation Synchronization

> **Classification:** `Guideline` (Rule 13 Error Visibility, `CONTRIBUTION.md` Lints, & Documentation SSoT)
> **Draft Status:** Phase 2 — Awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-error-visibility-lint-compliance-and-architecture-doc-sync.md`.

## 1. User-Centric Motivation & Goal

`synapto/CONTRIBUTION.md` (Lines 84-165), `ARCHITECTURE.md` (Rule 13), and the Error Visibility Mandate require that:
1. No `Result` is silently discarded (`if let Ok(...)`, `.unwrap_or_default()`, `drop(...)`, `.ok()`, or `let _ =`) without logging via `tracing::error!`.
2. Production code never uses `.expect(...)` on `Result` or non-static `Option` panics, and never discards error context with `|_|`.
3. Every crate in the workspace enforces `[lints] workspace = true` and compiles on stable Rust without warnings.
4. Codebase comments/logs use English (ASD-STE100), and `ARCHITECTURE.md` / `crates/interface/*.md` accurately reflect the current codebase structure.

Currently, 14 crates across `synapto`, `synapto-plugins`, and `org-assistant` contain silent error drops, forbidden `.expect(...)` calls, `drop(...)` on I/O results, `println!` calls, Czech comments/logs, disabled workspace lints in `synapto-test`, and outdated diagrams/module paths in `ARCHITECTURE.md` and `crates/interface/src/*.md`.

This RFC remediates all silent error suppression, lint violations, and documentation drift across the workspaces.

---

## 2. Executive Summary

- **Context:** All crates in `synapto`, `synapto-plugins`, and `org-assistant`, plus `synapto/docs/src/ARCHITECTURE.md` and `synapto/crates/interface/{README.md, src/*.md}`.
- **Motivation:** Ensure end-to-end error visibility in logs, 100% Clippy/lint compliance across every crate (including `synapto-test`), and accurate architectural documentation.
- **Overarching Goals:** Zero silent error drops, zero forbidden `.expect()` / `.unwrap()` / `|_|` in library code, and 100% synchronized documentation.
- **Purpose:** Fix all silent error drops, `.expect()` calls, `synapto-test` lint/nightly issues, non-English comments, and outdated documentation files.
- **Problem (Complete Audit Inventory):**
  1. **Silent Error Suppression (`if let Ok(...)`, `.unwrap_or_default()`, `drop(...)`, `.ok()`, `|_|`):**
     - `synapto/crates/interface/src/context.rs:119`: `if let Ok(val) = res` in `gather_contexts` drops `ContextProvider::context()` errors without logging.
     - `synapto/crates/interface/src/storage.rs:39`: `.unwrap_or_else(|_| unreachable!("TypeId mismatch in StorageRegistry"))` discards the `Arc<dyn Any>` dynamic type (`CONTRIBUTION.md:147-159`).
     - `synapto/crates/core/src/cognitive/types.rs:485-488`: `tool.erased_is_available(...).await.unwrap_or(false)` drops errors without logging.
     - `synapto/crates/core/src/{lib.rs:547, cognitive/direct.rs:134, 313, 537, 657, cognitive/side.rs:438, 524, cognitive/types.rs:58, 96, 160, 185}` & `crates/telemetry/src/{tracing.rs:322, rerun.rs:130, 304, 308, 312}`: `.ok()` without `.inspect_err(...)`.
     - `synapto/integrations/storage-providers/firestore/src/lib.rs:185, 230`: `if let Ok(obj) = ...` and `if let Ok(item) = item_res` silently drop deserialization and stream errors.
     - `synapto/integrations/storage-providers/surrealdb/src/lib.rs:153`: `if let Ok(parsed) = serde_json::from_value::<T>(val)` silently drops deserialization errors.
     - `synapto/integrations/decision-providers/typesafe/src/lib.rs:87`: `.unwrap_or_else(|_| "Unknown error".to_string())` discards the error with `|_|`.
     - `synapto/integrations/plugins/mcp/src/lib.rs:274, 286, 301, 306, 317, 329, 334, 344, 348, 350, 352, 382, 414, 507, 577, 582, 584, 588, 595, 619`: Uses `drop(...)` on 17 I/O and channel results, `if let Ok(resp)` on JSON-RPC parsing (`L301`), and `.ok()` on tool schema parsing (`L414, 619`).
     - `synapto/integrations/plugins/stt-google/src/lib.rs:238, 240, 515, 517`: `if let Ok(response) = client.streaming_recognize(...)` and `while let Ok(Some(reco_result)) = response_stream.message().await` silently swallow gRPC status errors.
     - `synapto/integrations/plugins/stt-elevenlabs/src/lib.rs:132, 189, 270` & `stt-speechmatics/src/lib.rs:202, 241`: Silently drop JSON and WebSocket errors.
     - `synapto/integrations/plugins/semantic-memory/src/lib.rs:299-308, 534-545`: `if let Ok(records) = store.get_ordered_records(...)` silently drops DB load errors.
     - `synapto/integrations/plugins/task-memory/src/{tasks.rs:192, goals.rs:69, missions.rs:70}`: `.get_ordered_records(...).await.unwrap_or_default()` masks DB read failures.
     - `org-assistant/integrations/google-chat/src/{read.rs:60-62, 233-241, reactions.rs:31-36, 78-80}`: `let Ok(...) = ... else { continue / return; }` silently drops Pub/Sub JSON, space lookup, HTTP, and credential errors.
  2. **Forbidden `.expect(...)`, `.unwrap()`, `println!`, and Non-English Text:**
     - `.expect("Missing value")` in `episodic-memory/src/lib.rs:197-216`, `linux-host-audio/src/{capture.rs:172-173, speaker.rs:133-134, 237-238}`, `semantic-memory/src/plugin.rs:290, 296`, `task-memory/src/lib.rs:279, 285, 291`, `behavioral-memory/src/lib.rs:225`.
     - `synapto/crates/test/{Cargo.toml:9-10, src/lib.rs:1, 109, 210, 218, 230, 442, 457, 458, src/plugins/*.rs, src/test_datadir.rs:16, 25}`: Commented-out `[lints] workspace = true`, nightly `#![feature(iter_array_chunks)]`, `.unwrap()`, and `let _ =`.
     - `synapto/integrations/plugins/mumble/src/lib.rs:309`: `println!` in production loop.
     - `synapto/integrations/plugins/semantic-memory/src/lib.rs:489, 498, 505, 509, 549, 610, 689, 698, 704`: Emoji in `tracing::info!` (`L704`) and redundant manual error prefixes.
     - `synapto-plugins/speaker-recognizer/src/lib.rs:79, 512-516`: Czech doc comment and Czech warning log.
  3. **Documentation Drift (`synapto/docs/src/ARCHITECTURE.md` & `synapto/crates/interface/`):**
     - `ARCHITECTURE.md:129, 138-139, 143-146, 158, 165, 176-186, 196-197, 216-218, 231-368`: References removed `src/google_text_to_speech.rs`, `src/elevenlabs_text_to_speech.rs`, `src/gui.rs`, `core/src/memories/`, `core/src/scenarist/`, `synapto_interface::core::Document`, `state_changes`, `CongnitiveOutputSpeech` typo, `ChatPlugin::start` `_add_document_tx` parameter, omits `llm` from `L179-185` (and `crates/core/src/config/provider.md:27-45`), and has an outdated Mermaid data flow diagram.
     - `crates/interface/{README.md:154, 282, 330-332, src/lib.rs:10, src/command.md:15, src/interaction.md:28, 36, 82, src/storage.md:121, src/tool.md:108-109}`: Outdated claims about `synapto_interface::sync` Rerun instrumentation, `synapto_interface::core` imports, and outdated trait signatures.
- **Solution:**
  1. Replace all silent error drops with `match` / `if let Err(e)` logging via `tracing::error!("{}", e)`.
  2. Replace `.expect(...)` in production code, enable `[lints] workspace = true` in `synapto-test/Cargo.toml`, replace `array_chunks` with `chunks_exact` on stable Rust, replace `println!` with `tracing::debug!`, remove the emoji in `semantic-memory`, and translate Czech comments/logs in `speaker-recognizer`.
  3. Update `synapto/docs/src/ARCHITECTURE.md`, `crates/core/src/config/provider.md`, and `crates/interface/{README.md, src/*.md}` to match the current codebase.

---

## 3. Current State Analysis (Exact File & Line Inventory)

See the complete file and line inventory in Section 2 (items 1, 2, and 3).

---

## 4. Structural Hierarchy

- **Mission:** End-to-End Diagnostic Visibility, Uniform Workspace Lints, and Accurate Documentation SSoT.
- **Goal:** Zero silent error suppression, zero Clippy/lint exemptions in library crates, and 100% accurate architectural documentation.
- **Atomic Responsibilities:**
  - **Error Logging Sites:** Log every `Err(e)` via `tracing::error!("{}", e)` (Rule 13).
  - **`synapto-test` (`crates/test/`):** Enforces `[lints] workspace = true` and compiles on stable Rust.
  - **Documentation (`docs/src/ARCHITECTURE.md`, `crates/interface/`):** Accurately documents the current plugin-based topology and trait signatures.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 13 (`ARCHITECTURE.md:31-40`):** Minimalist `tracing::error!("{}", e)` logging without silent drops.
- **`CONTRIBUTION.md` (Lines 84-165):** Mandatory Clippy compliance, no silent `Result` dropping, no `|_|` error discarding, no `.expect()` on `Result`.

### 5.2 Standard Error Logging Pattern
```rust
match store.get_ordered_records::<Task>("tasks", None, SortOrder::Ascending).await {
    Ok(records) => records,
    Err(e) => {
        tracing::error!("{}", e);
        Vec::new()
    }
}
```

### 5.3 JSON Contract Examples
_Not applicable (Logging, lint enforcement, and markdown documentation updates; no runtime JSON contract changes)._

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For `DummyLlmExecutor` (`synapto/crates/interface/src/plugin.rs:186`) and `DummyStore` (`synapto/crates/interface/src/storage.rs:184`) vs. `ARCHITECTURE.md:186` (*"Strict Mock Isolation (No Mocks in Core): Under no circumstances should 'Mock' or 'Dummy' providers... be defined within the `synapto` or `synapto-interface` crates—even for testing purposes"*):

**[ Option A: Move `DummyLlmExecutor`, `DummyStore`, `PluginTestContext`, and `EmptyPromptProvider` into `synapto-test` ]**
- **Action:** Move `PluginTestContext`, `DummyLlmExecutor`, and `DummyStore` out of `synapto-interface` and `EmptyPromptProvider` out of `synapto` into `synapto-test` (which is already the dedicated testing crate in `crates/test`), strictly enforcing `ARCHITECTURE.md:186`.
- **Pros:** Enforces Strict Mock Isolation (`ARCHITECTURE.md:186`) and removes test-only dummy structs from production `synapto-interface` and `synapto` binaries.
- **Cons:** Unit tests in plugins that use `PluginTestContext` import it from `synapto_test` (which is already in their `[dev-dependencies]`).

**[ Option B: Gate `PluginTestContext`, `DummyLlmExecutor`, and `DummyStore` Behind `#[cfg(any(test, feature = "test-utils"))]` or Amend `ARCHITECTURE.md:186` ]**
- **Action:** Keep `PluginTestContext` in `synapto-interface` (gated or documented as an exception in `ARCHITECTURE.md:186`).
- **Pros:** Zero import changes in plugin unit tests.
- **Cons:** Leaves dummy implementations inside `synapto-interface`.

**My Recommendation:** I recommend **Option A** because `synapto-test` is already in `[dev-dependencies]` across plugins, and moving `PluginTestContext`, `DummyLlmExecutor`, and `DummyStore` into `synapto-test` cleanly satisfies `ARCHITECTURE.md:186`.
Please select Option A or Option B for this RFC.
