# [MVP] [DRAFT] RFC: Interface Encapsulation, Builder Typestate, Opaque Resource IDs, and Strong Typing

> **Classification:** `Guideline` (Rules 15, 16, 17, 22, 23, 26, & 31 Architectural Hygiene)
> **Draft Status:** Phase 2 — Awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-interface-encapsulation-typestate-and-strong-typing.md`.

## 1. User-Centric Motivation & Goal

Synapto's architecture rules enforce compile-time safety and clean module boundaries:
- **Rule 15 (Strong Typing & Newtype Pattern):** No primitive obsession (`String`, `u32`, `i64`) for domain IDs, and no mystery boolean parameters (`bool`) in function signatures.
- **Rule 17 (Opaque Resource Identifier Protocol):** `DocumentId`s exposed to the LLM must be opaque identifiers without filename or path metadata leakage.
- **Rule 23 (Direct Wiring Principle):** No unnecessary intermediate channel forwarding loops.
- **Rule 26 (Encapsulated Handles):** Public API boundaries must hide internal `RwLock`s and `Arc<dyn Trait>` behind opaque handles.
- **Rule 31 (Fluent Bundle Builder Pattern):** Builder stages must enforce single registration at compile time via typestate, and singleton plugin registrations must behave consistently.

Currently, several crates violate these structural guidelines: `ToolRegistryBuilder` and `CommandRegistryBuilder` expose `pub RwLock` fields, `ToolHandle`/`DecisionHandle`/`LlmExecutor` expose `Arc<dyn ...>`, `synapto-interface` embeds speaker alignment heuristics (`speech_to_text.rs:102-207`), `SynaptoBuilder` only enforces single-call typestate on `.decision()` and `.llm()`, `PluginRegistry` silently overwrites duplicate singleton plugins on 6 roles while panicking on 4 roles, `Synapto` holds dead state (`current_context_tx`, unused camera frame clones, write-only `plugins` map, global `DYNAMIC_CAPABILITIES`), `synapto-plugins/documents` derives `DocumentId` from `original_filename` and spawns a redundant channel forwarding loop, and multiple plugins use mystery booleans and raw primitive IDs.

This RFC brings all affected crates into full compliance with Rules 15, 16, 17, 22, 23, 26, and 31.

---

## 2. Executive Summary

- **Context:** `synapto/crates/{interface, core}`, `synapto/integrations/*`, `synapto-plugins/{documents, stt-moonshine}`, and `org-assistant/integrations/{google-chat, google-mail, google-meet}`.
- **Motivation:** Enforce compile-time encapsulation, typestate completeness, opaque resource identifiers, and strong domain typing across the workspace.
- **Overarching Goals:** Eliminate encapsulation leaks, dead core state, filename-leaking `DocumentId`s, redundant channel bridges, mystery booleans, and primitive obsession.
- **Purpose:** Remediate all Rule 15, 16, 17, 22, 23, 26, and 31 guideline violations identified in the architectural audit.
- **Problem (Complete Audit Inventory):**
  1. **Rule 26 Encapsulation Leaks in `synapto-interface` & Providers:**
     - `synapto/crates/interface/src/tool.rs:182` (`pub tools: std::sync::RwLock<...>`) and `command.rs:48-49` (`pub commands: std::sync::RwLock<...>`) accessed directly by `crates/core/src/cognitive/{direct.rs:148-153, side.rs:86-91}`.
     - `ToolHandle` (`tool.rs:108-118`: `from_arc`, `into_inner`, `inner`), `DecisionHandle::set_arc_backend` (`decision.rs:108`), `LlmExecutor::from_arc` (`llm.rs:120`), `DecisionProvider::raw_decision_executor` (`decision.rs:125`, `typesafe/src/lib.rs:119`), and `LlmProvider::raw_llm_executor` (`llm.rs:137`, `google/src/lib.rs:219`) expose `Arc<dyn ...>` across crate boundaries.
  2. **Section 6 Interface Purity & Asymmetric Capability Registration in `synapto-interface`:**
     - `synapto/crates/interface/src/speech_to_text.rs:102-207`: Embeds speaker alignment implementation (`CosineSimilarity`, `WordOverlap`, `InternalSpeaker`, `"Maybe {}"` formatting) inside `synapto-interface`.
     - `synapto/crates/interface/src/plugin.rs:127-131`: `register_call` takes `capability: Option<&'static str>` manually instead of reading `P::CAPABILITY` (`plugin.rs:149`).
  3. **Rule 31 Incomplete Typestate & Inconsistent Duplicate Role Handling in `crates/core`:**
     - `synapto/crates/core/src/lib.rs:457-501`: `.configs()`, `.storage()`, `.prompt()`, `.credentials()`, and `.plugins()` can be called multiple times on `SynaptoBuilder`, silently overwriting previous types; `ConfigProvider` is implemented for `()` (`config/provider.rs:212`).
     - `synapto/crates/core/src/lib.rs:1200, 1287, 1402, 1420, 1460, 1472`: `register_chat`, `register_documents`, `register_diarization`, `register_call`, `register_gui`, and `register_camera` silently overwrite `self.<role>_spawner = Some(...)` instead of panicking on duplicate registration like `register_audio_input`/`output`/`stt`/`tts` (`lib.rs:1137, 1151, 1166, 1184`).
  4. **Dead State & Process-Global Static in `crates/core`:**
     - `synapto/crates/core/src/lib.rs:599-600, 634, 657, 734-737, 957-974`: `self.plugins_names`, `self.plugins`, and `self.current_context_tx` (whose receiver is immediately dropped at `L657`) are dead/unread.
     - `synapto/crates/core/src/lib.rs:872-879` & `cognitive/direct.rs:421-424`: `video_rx` wakes `cognitive_direct_task` on every frame only to clone and drop `let _video_frame = ...`.
     - `synapto/crates/core/src/lib.rs:1104-1124` & `cognitive.rs:151-165`: `DYNAMIC_CAPABILITIES` is a process-global `OnceLock<Mutex<Vec<String>>>`.
     - `synapto/crates/core/src/users.rs:15-25` & `cognitive/types.rs:222-226`: `Users::get_by_speaker_id` stub makes `LLMUser::Distinguishable` unreachable.
  5. **Rule 17 (ORI) & Rule 23 (Direct Wiring) Violations:**
     - `synapto-plugins/documents/src/service.rs:718-732`: Generates `DocumentId` from `req.original_filename.replace(['/', '\\'], "_")` with an O(N) DB loop instead of an opaque identifier.
     - `synapto-plugins/documents/src/service.rs:636, 647-656`: Spawns a redundant task forwarding `add_document_rx` to `internal_add_document_rx`.
     - `synapto/integrations/plugins/{behavioral-memory/src/lib.rs:168-177, episodic-memory/src/lib.rs:51-60, 78-87, 105-114, semantic-memory/src/lib.rs:111-120, task-memory/src/lib.rs:63-74, 102-113, 141-152}`: Spawns 8 intermediate tasks in `ContextProvider::subscribe()` just to erase `watch::Receiver<T>` to `watch::Receiver<()>`.
  6. **Rule 15 (Strong Typing, Newtypes, & No Mystery Booleans) & Rule 16 (`{Subject}{Direction}{Type}`):**
     - Mystery booleans: `org-assistant/integrations/google-meet/src/lib.rs:55, 376` (`setup_browser(..., load_state: bool)`, `turn_microphone(page, on: bool)`), `synapto-plugins/stt-moonshine/src/tokenizer.rs:67` (`decode(..., skip_specials: bool)`), `synapto/integrations/prompt-providers/{embedded/src/lib.rs:35, file/src/lib.rs:42}` (`get_dynamic_instructions(..., _is_initial_run: bool)`), `synapto/integrations/plugins/skills/src/tool.rs:46-77` (inspects structured JSON keys instead of fulltext scan).
     - Primitive obsession: `synapto/integrations/plugins/semantic-memory/src/lib.rs:35-40, 57-74, 188, 195, 235-237, 271` (raw `u32` for `Insight.id`, `Activity.id`, and raw `i64` for timestamps), `synapto/integrations/plugins/task-memory/src/tasks.rs:31` (raw `i64` timestamp), `synapto-plugins/documents/src/{plugin.rs:76, 131, service.rs:28, 53-138}` (raw `String` instead of `DocumentId`, `Url`, `DocumentStatus`), `org-assistant/integrations/google-mail/src/{tools.rs:12-23, gmail.rs:3-11}` (6 consecutive `&str`/`String` arguments), `org-assistant/integrations/google-chat/src/types.rs:4-9` (raw `String` IDs).
- **Solution:**
  1. Make `ToolRegistryBuilder::tools` and `CommandRegistryBuilder::commands` private and expose safe accessor methods (`get_command(&self, name: &str)`). Encapsulate `Arc` construction inside `ToolHandle::new<T: ErasedTool>(tool: T)`, `DecisionHandle::new<D: RawDecisionExecutor>(exec: D)`, and `LlmExecutor::new<L: RawLlmExecutor>(exec: L)`.
  2. Move speaker alignment heuristics (`CosineSimilarity`, `WordOverlap`, `InternalSpeaker`) from `synapto-interface/src/speech_to_text.rs` to `synapto/crates/core/src/speech_to_text/speaker_transcript_alignment.rs`.
  3. Restrict `.configs()`, `.storage()`, `.prompt()`, `.credentials()`, and `.plugins()` on `SynaptoBuilder` to `NoConfig`, `NoStorage`, `NoPrompt`, `NoCredentials`, and `NoPlugins` typestate markers, remove `impl ConfigProvider for ()`, panic consistently on duplicate singleton plugin registration, and remove dead fields (`current_context_tx`, `plugins`, `plugins_names`, `DYNAMIC_CAPABILITIES`) from `Synapto`.
  4. Generate opaque `DocumentId`s (`doc_<16-hex>`) in `synapto-plugins/documents` and remove the redundant `add_document_rx` forwarding task.
  5. Replace boolean parameters with descriptive enums and wrap raw IDs/timestamps in domain newtypes.

---

## 3. Current State Analysis (Exact File & Line Inventory)

See the complete file and line inventory in Section 2 (items 1 through 6).

---

## 4. Structural Hierarchy

- **Mission:** Compile-Time Type Safety, Handle Encapsulation, and Interface Cleanliness.
- **Goal:** Eliminate all Rule 15, 16, 17, 22, 23, 26, and 31 violations across `synapto`, `synapto-plugins`, and `org-assistant`.
- **Atomic Responsibilities:**
  - **`SynaptoBuilder` (`crates/core/src/lib.rs`):** Enforces single-call typestate transitions across all 7 builder methods.
  - **`ToolHandle` / `DecisionHandle` / `LlmExecutor` (`crates/interface/src/`):** Wrap concrete implementations internally without exposing `Arc<dyn ...>` in public signatures.
  - **`DocumentService` (`synapto-plugins/documents/src/service.rs`):** Generates opaque `DocumentId`s (`doc_<16-hex>`) and uses strongly typed `DocumentId`, `DocumentStatus`, and `Url` fields.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rules 15, 16, 17, 22, 23, 26, 31 (`synapto/docs/src/ARCHITECTURE.md`).**

### 5.2 Typestate Builder & Opaque Handle Signatures
```rust
// synapto/crates/core/src/lib.rs
pub struct NoPrompt;
pub struct NoCredentials;
pub struct NoPlugins;

impl<S, PR, CR, D, L, P> SynaptoBuilder<NoConfig, S, PR, CR, D, L, P> {
    pub fn configs<C: ConfigProvider>(self) -> SynaptoBuilder<C, S, PR, CR, D, L, P> { ... }
}

// synapto/crates/interface/src/llm.rs
impl LlmExecutor {
    pub fn new<E: RawLlmExecutor + 'static>(executor: E) -> Self {
        Self(Arc::new(executor))
    }
}
```

### 5.3 JSON Contract Examples

#### Opaque `DocumentId` in Prompt Context (`context_plugins.documents`)
```json
{
  "context_plugins": {
    "documents": {
      "available_documents": [
        {
          "document_id": "doc_a1b2c3d4e5f60718",
          "original_filename": "quarterly_report.pdf",
          "summary": "Q3 financial metrics and projections."
        }
      ]
    }
  }
}
```

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For eliminating the 8 intermediate forwarding tasks in `ContextProvider::subscribe()` (`synapto/crates/interface/src/context.rs:36, 146-156` and memory plugins):

**[ Option A: Have `ContextRegistryBuilder::register` Observe `watch::Receiver<T>` Directly via a Generic Subscription Helper ]**
- **Action:** Provide a helper `ContextSubscription::from_watch(rx: watch::Receiver<T>)` (or trait method on `ContextProvider`) so `ContextRegistryBuilder` awaits `rx.changed().await` directly in its single notification loop without each plugin spawning its own `watch::Receiver<T> -> watch::Receiver<()>` intermediate task.
- **Pros:** Eliminates 8 spawned forwarding tasks across `behavioral-memory`, `episodic-memory`, `semantic-memory`, and `task-memory` and satisfies Rule 23.
- **Cons:** Minor adjustment to `ContextProvider::subscribe` return type (`Option<ContextSubscription>`).

**[ Option B: Keep `Option<watch::Receiver<()>>` in `ContextProvider::subscribe` ]**
- **Action:** Keep `watch::Receiver<()>` as the return type of `ContextProvider::subscribe()`.
- **Pros:** No change to `ContextProvider::subscribe` signature.
- **Cons:** Requires every `ContextProvider` holding a typed `watch::Receiver<T>` to spawn an extra forwarding task just to erase `T` to `()`.

**My Recommendation:** I recommend **Option A** because it eliminates 8 redundant background tasks and removes boilerplate from every memory plugin.
Please select Option A or Option B for this RFC.
