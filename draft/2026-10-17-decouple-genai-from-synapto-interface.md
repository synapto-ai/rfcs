# [MVP] [DRAFT] RFC: Decouple `genai` and `futures` from `synapto-interface`

> **Classification:** `Guideline` (Interface Boundary Purity & Zero Third-Party Leakage)
> **Draft Status:** Phase 2 — Awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-decouple-genai-from-synapto-interface.md`.

## 1. User-Centric Motivation & Goal

`synapto-interface` is the contract crate shared by every plugin, provider, and core crate in the ecosystem. Its documentation (`synapto/crates/interface/README.md:5`) mandates that `synapto-interface` must be free of third-party dependencies so plugin crates remain fast to compile and immune to breaking changes in external client libraries.

Currently, `synapto/crates/interface/Cargo.toml:23-24` depends directly on `genai = "0.6"` and `futures = "0.3.32"`, and `synapto/crates/interface/src/llm.rs:1` publicly re-exports `pub use genai;`. Furthermore, `RawLlmOptions` (`llm.rs:46-52`) and `RawLlmExecutor` (`llm.rs:57-65`) expose `genai::chat::{ChatMessage, ChatResponse, ReasoningEffort, Tool, ToolCall}` directly in their signatures, contradicting the doc comment on `RawLlmExecutor` (`llm.rs:54`: `"completely decoupled from any specific client library"`).

This RFC removes `genai` and `futures` from `synapto-interface` and replaces `genai::chat::*` types on `RawLlmExecutor` with native `synapto-interface::llm` DTOs.

---

## 2. Executive Summary

- **Context:** `synapto/crates/interface/{Cargo.toml, src/llm.rs}`, `synapto/crates/llm/src/lib.rs`, and `synapto/integrations/llm-providers/google/src/lib.rs`.
- **Motivation:** Eliminate third-party LLM client library leakage across the `synapto-interface` boundary.
- **Overarching Goals:** Make `synapto-interface` 100% vendor-neutral and client-library-agnostic.
- **Purpose:** Remove `genai` and `futures` from `synapto/crates/interface/Cargo.toml`, define native `LlmChatMessage`, `LlmTool`, `LlmChatResponse`, `ToolCall`, and `ReasoningEffort` types in `synapto_interface::llm`, and confine `genai` conversions strictly to `synapto-llm-google` (or `synapto-llm`).
- **Problem:**
  1. **Direct `genai` Dependency & Re-Export (`synapto/crates/interface/Cargo.toml:23`, `src/llm.rs:1, 46-65`):** `pub use genai;` exposes the entire `genai` crate through `synapto_interface::llm::genai`.
  2. **Direct `futures` Dependency (`synapto/crates/interface/Cargo.toml:24`, `src/decision.rs:1`, `src/llm.rs:56`, `src/plugin.rs:189`):** Used only for `futures::future::BoxFuture`, which is a 1-line type alias `pub type BoxFuture<'a, T> = std::pin::Pin<Box<dyn std::future::Future<Output = T> + Send + 'a>>;` in `std` (or via `#[async_trait]`).
- **Solution:**
  1. Define `BoxFuture<'a, T>` using `std::pin::Pin` and `std::future::Future` (or `#[async_trait]`) in `synapto-interface` and remove `futures` from `crates/interface/Cargo.toml`.
  2. Define native `ReasoningEffort`, `ToolCall`, `LlmTool`, `LlmChatMessage`, and `LlmChatResponse` structs in `synapto_interface::llm` and remove `genai` from `crates/interface/Cargo.toml`.
  3. Perform the translation between `synapto_interface::llm::*` and `genai::chat::*` inside `synapto-llm-google`.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/crates/interface/Cargo.toml:23-24`**:
   ```toml
   genai = { version = "0.6" }
   futures = "0.3.32"
   ```
2. **`synapto/crates/interface/src/llm.rs:1, 46-65`**:
   ```rust
   pub use genai;
   ...
   pub struct RawLlmOptions {
       pub output_schema: Option<serde_json::Value>,
       pub reasoning_effort: Option<genai::chat::ReasoningEffort>,
   }

   pub trait RawLlmExecutor: Send + Sync {
       fn execute_chat<'a>(
           &'a self,
           model: &'a str,
           messages: Vec<genai::chat::ChatMessage>,
           tools: Option<Vec<genai::chat::Tool>>,
           options: &'a RawLlmOptions,
       ) -> futures::future::BoxFuture<'a, Result<genai::chat::ChatResponse, String>>;
   }
   ```
3. **External Consumers of `synapto_interface::llm::genai`**:
   - `synapto/crates/llm/src/lib.rs:5` (`pub use synapto_interface::llm::genai::chat::{ReasoningEffort, ToolCall};`)
   - `synapto/crates/core/src/cognitive/{direct.rs, side.rs, types.rs}`
   - `synapto/integrations/llm-providers/google/src/lib.rs:1`
   - `synapto-plugins/documents/src/service.rs:14, 817` (`synapto_llm::ReasoningEffort`)

---

## 4. Structural Hierarchy

- **Mission:** Pure, Zero-Third-Party Interface Contract (`synapto-interface`).
- **Goal:** Confine `genai` to concrete LLM provider implementations (`synapto-llm-google`).
- **Atomic Responsibilities:**
  - **`synapto_interface::llm` (`crates/interface/src/llm.rs`):** Defines pure `serde`/`schemars` DTOs (`ReasoningEffort`, `ToolCall`, `LlmTool`, `LlmChatMessage`, `LlmChatResponse`) and `RawLlmExecutor`.
  - **`synapto-llm-google` (`integrations/llm-providers/google/src/lib.rs`):** Maps `synapto_interface::llm::*` to `genai::chat::*`.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **`synapto/crates/interface/README.md` Line 5:** `synapto-interface` MUST be free of 3rd-party dependencies.
- **Rule 33 (Pluggable Singleton LLM Provider):** Decouple the concrete generative LLM client library from the core and interface crates.

### 5.2 Native `synapto_interface::llm` Types
```rust
// synapto/crates/interface/src/llm.rs
pub type BoxFuture<'a, T> = std::pin::Pin<Box<dyn std::future::Future<Output = T> + Send + 'a>>;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, JsonSchema)]
pub enum ReasoningEffort {
    None,
    Low,
    Medium,
    High,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize, JsonSchema)]
pub struct ToolCall {
    pub call_id: String,
    pub fn_name: String,
    pub fn_arguments: serde_json::Value,
    pub thought_signatures: Option<Vec<String>>,
}
```

### 5.3 JSON Contract Examples
_Not applicable (Internal Rust trait boundary refactoring; wire JSON formats remain identical)._

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For `RawLlmExecutor` and `RawDecisionExecutor` async method signatures in `synapto-interface`:

**[ Option A: Use `#[async_trait]` on `RawLlmExecutor` and `RawDecisionExecutor` ]**
- **Action:** Annotate `RawLlmExecutor` (`crates/interface/src/llm.rs:57`) and `RawDecisionExecutor` (`crates/interface/src/decision.rs:62`) with `#[async_trait]` (which is already a workspace dependency in `synapto-interface/Cargo.toml:20`) and remove `futures` from `synapto-interface/Cargo.toml`.
- **Pros:** Matches every other trait in `synapto-interface` (`Plugin`, `ChatPlugin`, `Tool`, `Command`, `ContextProvider`, `RecordStore`), eliminates manual `Box::pin(async move { ... })` boilerplate in providers, and removes `futures` from `synapto-interface`.
- **Cons:** None.

**[ Option B: Define `pub type BoxFuture<'a, T> = Pin<Box<dyn Future<Output = T> + Send + 'a>>` Manually ]**
- **Action:** Keep the manual `BoxFuture` return type using `std::pin::Pin` and `std::future::Future`.
- **Pros:** Removes `futures` crate without changing provider `Box::pin` implementations.
- **Cons:** Inconsistent with all other traits in `synapto-interface` that use `#[async_trait]`.

**My Recommendation:** I recommend **Option A** for consistency with all other traits in `synapto-interface`.
Please select Option A or Option B for this RFC.
