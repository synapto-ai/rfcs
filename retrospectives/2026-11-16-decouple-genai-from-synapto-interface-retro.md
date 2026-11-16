# Retrospective: Decouple `genai` and `futures` from `synapto-interface`

- **RFC:** [`implemented/2026-10-17-decouple-genai-from-synapto-interface.md`](../implemented/2026-10-17-decouple-genai-from-synapto-interface.md)
- **Date:** 2026-11-16
- **Classification:** `Guideline`

## 1. Trigger & Initial Motivation

A requested dependency upgrade of `genai` to `0.7.5` inside `synapto/integrations/llm-providers/google` exposed tight cross-crate coupling: `synapto-interface` re-exported `genai` (`pub use genai;`) and exposed `genai::chat` types in `RawLlmOptions` and `RawLlmExecutor`, while `synapto-llm` also depended directly on `genai = "0.6"`. Upgrading `genai` in the Google provider alone was impossible without breaking or coupling the core crates. The Lead Architect directed updating and executing the existing draft RFC (`2026-10-17-decouple-genai-from-synapto-interface.md`) first.

## 2. Evolution of the RFC Design

1. **Reconciling Stale Phase 2 Draft State (Main Agent):**
   - The original `2026-10-17` draft assumed `RawLlmExecutor` and `RawDecisionExecutor` still used `futures::future::BoxFuture` and put the Phase 2 Architectural Fork on `#[async_trait]` vs `BoxFuture`.
   - Codebase inspection showed both traits had already migrated to `#[async_trait]`, leaving `futures::future::join_all` in `synapto/crates/interface/src/context.rs:115` as the sole `futures` usage, and showed `synapto-llm` also depended on `genai = "0.6"`.
   - The Main Agent updated Sections 1-5 to reflect the actual codebase and formulated a new Phase 2 fork between **Option A (Minimal YAGNI Contract)** and **Option B (Full Chat-History & Response Metadata Parity)**.

2. **Fork Decision (Lead Architect):**
   - The Lead Architect selected **Option A**, removing the unused `RawLlmOptions.messages` field and limiting `RawLlmResponse` to `texts: Vec<String>` and `tool_calls: Vec<ToolCall>`.

3. **Zero-Context Review Refinements (Subagent):**
   - **Vendor Neutrality (`provider_metadata`):** The first review pass flagged `thought_signatures: Option<Vec<String>>` on `synapto_interface::llm::ToolCall` as a Gemini-specific field leaking across `synapto-interface`. Replacing it with opaque `provider_metadata: Option<serde_json::Value>` confined `thought_signatures` semantics to `synapto-llm-google`.
   - **Strong Typing (`ToolCallId`):** The first review pass identified that `synapto_interface::tool::ToolCallId` already existed, so `ToolCall.call_id` was typed as `ToolCallId` instead of `String`, removing redundant `ToolCallId(...)` wrapping in `direct.rs` and `side.rs`.
   - **`None` vs `Some(Value::Null)` Serialization Guard:** The second review pass caught that `serde_json::to_value(&tc.thought_signatures)` would serialize `None` into `Some(Value::Null)` instead of `None`, polluting `ToolCallFmt` logs and equality checks. The final implementation uses `tc.thought_signatures.as_ref().map(serde_json::to_value).transpose()`, returning explicit `Err` on serialization or deserialization failure.

## 3. Rejected Options

- **Option B (Full Chat-History & Metadata Parity):** Rejected because `RawLlmOptions.messages` was `None` at all call sites and `usage`, `stop_reason`, `reasoning_content`, and `response_id` on `ChatResponse` were never read by `synapto-llm`. Adding unused DTOs (`LlmChatMessage`, `LlmChatRole`, `LlmContentPart`) violated YAGNI.
- **Retaining `thought_signatures` as a first-class field on `ToolCall`:** Rejected under `ARCHITECTURE.md` Rule 34 and Section 6 (`Platform-Agnostic Capabilities`).
- **Upgrading `genai` across `synapto-interface` and `synapto-llm` before decoupling:** Rejected by the Lead Architect to preserve interface boundary purity first.

## 4. Implementation & Verification Reconciliation

- All 12 DoD items matched the implemented code without drift.
- **Hidden `tokio/fs` Feature Unification Fix:** Testing `org-assistant` in dev mode (`../dev-config.toml`) and standalone `synapto` crate tests revealed that `synapto/crates/core/src/storage/local_storage.rs`, `synapto/crates/test/src/lib.rs`, and `synapto/integrations/plugins/skills/src/model.rs` used `tokio::fs`, which was previously enabled only transitively through `genai`'s dependency tree. Added `features = ["fs"]` explicitly to `tokio` in `synapto/crates/core/Cargo.toml`, `synapto/crates/test/Cargo.toml`, and `synapto/integrations/plugins/skills/Cargo.toml`.
- Minor verification note: running `cargo clippy --workspace --all-targets --all-features` timed out compiling `synapto-telemetry` (`tracy-client-sys` / `re_sdk`), so Clippy verification was completed via `cargo clippy --workspace --all-targets -- -D warnings` plus `cargo clippy -p synapto-llm --all-targets --features rerun -- -D warnings`. Similarly, `cargo check --config ../dev-config.toml` in `org-assistant` must be run from `org-assistant/assistant` (or `org-assistant-data`) because relative patch paths in `dev-config.toml` are anchored one directory below the `org-assistant` root.

## 5. Process Evolution Notes

- No changes to `rfcs/AGENTS.md` are required; the Zero-Context Resumability Mandate and Phase 3 zero-context subagent review caught real boundary leaks (`thought_signatures`, `ToolCallId`, `Some(Value::Null)`) prior to implementation.
