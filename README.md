# Synapto Architecture RFCs

This repository is the Single Source of Truth for architectural Requests for Comments (RFCs) and retrospectives across the Synapto framework (`synapto`), domain plugins (`synapto-plugins`), and deployable assistant bundles (`org-assistant`).

## Lifecycle & Directory Structure

Every RFC file is prefixed with its creation date (`YYYY-MM-DD-<slug>.md`) and resides in the directory matching its current lifecycle state:

- **`draft/`**: Active proposals and drafts undergoing design, Architectural Fork evaluation, or review (`[DRAFT]` in H1 title).
- **`reproduced/`**: `Flaw` RFCs whose primary failure has been deterministically reproduced via an `xfail` (`#[should_panic(expected = "Scenario Failed")]`) scenario test in `synapto/crates/test` (`[DRAFT]` in H1 title).
- **`approved/`**: Frozen RFCs formally approved by the Lead Architect (`[DRAFT]` removed), ready for or undergoing implementation.
- **`implemented/`**: Completed RFCs verified against their Definition of Done (DoD), passing scenario tests (`#[should_panic]` removed and test/scenario renamed to canonical `<slug>` without the `rfc_<date>_` / `YYYY-MM-DD-` prefix), and the Solution Verification Protocol.
- **`retrospectives/`**: Post-implementation retrospectives (`YYYY-MM-DD-<slug>-retro.md`) capturing architectural evolution, rejected options, and mid-execution pivots.

Every RFC addresses **one specific problem** (`1 Problem = 1 RFC`) and is classified as:
- **`Flaw`**: Fixes a runtime bug, concurrency race, deadlock, data loss, resource leak, or security vulnerability (`draft/` -> `reproduced/` -> `approved/` -> `implemented/`).
- **`Guideline`**: Enforces an architectural boundary, interface purity rule, configuration explicitness rule, strong typing rule, or documentation invariant (`draft/` -> `approved/` -> `implemented/`).
- **`Feature`**: Introduces a net-new capability or subsystem (`draft/` -> `approved/` -> `implemented/`).

See [`AGENTS.md`](AGENTS.md) for the complete RFC lifecycle protocol, Failure Scenario Reproduction rules (Phase 5.5), and the Zero-Context Resumability Mandate.

---

## RFC Index

### 1. Draft (`draft/` — `Guideline` & `Feature` Proposals)

| Date | RFC | Type | Title | Draft Status | Target Crates |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `2026-07-20` | [`2026-07-20-api-surface-minimization.md`](draft/2026-07-20-api-surface-minimization.md) | `Guideline` | API Surface Minimization | Phase 6 (Awaiting Approval) | `synapto`, `synapto-interface` |
| `2026-07-20` | [`2026-07-20-data-dir-strategy.md`](draft/2026-07-20-data-dir-strategy.md) | `Guideline` | Refactor DataDirProvider to Mandatory Trait-Based Strategy | Phase 6 (Awaiting Approval) | `synapto`, `datadir-providers` |
| `2026-07-20` | [`2026-07-20-plugin-startup-macro.md`](draft/2026-07-20-plugin-startup-macro.md) | `Feature` | Declarative Plugin Startup Macro | Phase 6 (Awaiting Approval) | `synapto-derive`, `synapto-interface` |
| `2026-09-12` | [`2026-09-12-unified-credentials-provider.md`](draft/2026-09-12-unified-credentials-provider.md) | `Feature` | Pluggable Credentials Provider and Ambient Cloud Identity Resolution | Phase 6 (Reconcile / Archive) | `synapto-interface`, `credentials-providers` |
| `2026-09-19` | [`2026-09-19-decision-integration-phases.md`](draft/2026-09-19-decision-integration-phases.md) | `Feature` | Integration Roadmap for Decision Executor: Preflight & Turn Gating | Phase 6 (Awaiting Approval) | `synapto`, `decision-providers`, memory plugins |
| `2026-09-25` | [`2026-09-25-contextual-prompt-roles.md`](draft/2026-09-25-contextual-prompt-roles.md) | `Feature` | Contextual Prompt Roles & Situational Prompt Routing | Phase 6 (Awaiting Approval) | `synapto`, `prompt-providers` |
| `2026-10-17` | [`2026-10-17-decouple-genai-from-synapto-interface.md`](draft/2026-10-17-decouple-genai-from-synapto-interface.md) | `Guideline` | Decouple `genai` and `futures` from `synapto-interface` | Phase 2 (Fork Checkpoint) | `synapto-interface`, `synapto-llm`, `synapto-llm-google` |
| `2026-10-17` | [`2026-10-17-structural-llm-safe-derive-and-visibility-boundaries.md`](draft/2026-10-17-structural-llm-safe-derive-and-visibility-boundaries.md) | `Guideline` | Structural `#[derive(LLMSafe)]` Field Verification and `pub(crate)` Visibility Boundaries | Phase 2 (Fork Checkpoint) | `synapto-derive`, `synapto-interface`, `synapto`, memory plugins |
| `2026-10-17` | [`2026-10-17-unified-credentials-provider-migration.md`](draft/2026-10-17-unified-credentials-provider-migration.md) | `Guideline` | Unified Credentials Provider Migration and Environment Variable Elimination | Phase 2 (Fork Checkpoint) | `google-mail`, `llm-providers/google`, `credentials-providers/*`, `org-assistant` |
| `2026-10-17` | [`2026-10-17-explicit-configuration-and-orthogonal-namespaces.md`](draft/2026-10-17-explicit-configuration-and-orthogonal-namespaces.md) | `Guideline` | Explicit Configuration Schemas and Orthogonal Configuration Namespaces | Phase 2 (Fork Checkpoint) | `synapto`, `synapto-interface`, all plugin/provider configs |
| `2026-10-17` | [`2026-10-17-non-blocking-plugin-start-and-async-io-offloading.md`](draft/2026-10-17-non-blocking-plugin-start-and-async-io-offloading.md) | `Guideline` | Non-Blocking Plugin Task Spawning and Async I/O Offloading | Phase 2 (Fork Checkpoint) | `stt-*`, `tts-*`, `call-recorder`, `skills`, `mumble`, `google-mail` |
| `2026-10-17` | [`2026-10-17-interface-encapsulation-typestate-and-strong-typing.md`](draft/2026-10-17-interface-encapsulation-typestate-and-strong-typing.md) | `Guideline` | Interface Encapsulation, Builder Typestate, Opaque Resource IDs, and Strong Typing | Phase 2 (Fork Checkpoint) | `synapto-interface`, `synapto`, `documents`, `semantic-memory`, `google-meet` |
| `2026-10-17` | [`2026-10-17-error-visibility-lint-compliance-and-architecture-doc-sync.md`](draft/2026-10-17-error-visibility-lint-compliance-and-architecture-doc-sync.md) | `Guideline` | Error Visibility, Workspace Lint Compliance, and Architecture Documentation Synchronization | Phase 2 (Fork Checkpoint) | `synapto-test`, `ARCHITECTURE.md`, `synapto-interface`, storage/STT/memory plugins |

---

### 2. Reproduced (`reproduced/` — `Flaw` RFCs with Verified `#[should_panic(expected = "Scenario Failed")]` Tests)

| Date | RFC | Type | Title | Scenario Test (`synapto-test`) | Draft Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `2026-09-20` | [`2026-09-20-runtime-stability-and-smells-remediation.md`](reproduced/2026-09-20-runtime-stability-and-smells-remediation.md) | `Flaw` | Framework Stability and Runtime Reliability Hardening | `rfc_2026_09_20_runtime_stability_and_smells_remediation` | Phase 6 (Awaiting Approval) |
| `2026-10-17` | [`2026-10-17-broadcast-lag-resilience-and-direct-audio-wiring.md`](reproduced/2026-10-17-broadcast-lag-resilience-and-direct-audio-wiring.md) | `Flaw` | Broadcast Lag Resilience and Direct Audio Stream Wiring | `rfc_2026_10_17_broadcast_lag_resilience_and_direct_audio_wiring` | Phase 2 (Fork Checkpoint) |
| `2026-10-17` | [`2026-10-17-deterministic-in-flight-tool-resolution-sync.md`](reproduced/2026-10-17-deterministic-in-flight-tool-resolution-sync.md) | `Flaw` | Deterministic In-Flight Tool Resolution Synchronization and Non-Blocking Observer Fan-Out | `rfc_2026_10_17_deterministic_in_flight_tool_resolution_sync` | Phase 2 (Fork Checkpoint) |
| `2026-10-17` | [`2026-10-17-episodic-memory-persistence-and-rollout-registration.md`](reproduced/2026-10-17-episodic-memory-persistence-and-rollout-registration.md) | `Flaw` | Episodic Memory Persistence and Rollout Controller Registration | `rfc_2026_10_17_episodic_memory_persistence_and_rollout_registration` | Phase 2 (Fork Checkpoint) |
| `2026-10-17` | [`2026-10-17-semantic-memory-batch-accumulation-and-startup-barrier.md`](reproduced/2026-10-17-semantic-memory-batch-accumulation-and-startup-barrier.md) | `Flaw` | Semantic Memory Batch Accumulation and Startup Consolidation Barrier | `rfc_2026_10_17_semantic_memory_batch_accumulation_and_startup_barrier` | Phase 2 (Fork Checkpoint) |
| `2026-10-17` | [`2026-10-17-behavioral-memory-opaque-keys-and-surrealdb-query-binding.md`](reproduced/2026-10-17-behavioral-memory-opaque-keys-and-surrealdb-query-binding.md) | `Flaw` | Behavioral Memory Opaque Record Keys and SurrealDB Parameterized Vector Queries | `rfc_2026_10_17_behavioral_memory_opaque_keys_and_surrealdb_query_binding` | Phase 2 (Fork Checkpoint) |
| `2026-10-17` | [`2026-10-17-llm-schema-ref-inlining-and-boundary-sanitization.md`](reproduced/2026-10-17-llm-schema-ref-inlining-and-boundary-sanitization.md) | `Flaw` | LLM JSON Schema `$ref` Inlining and Boundary Sanitization | `rfc_2026_10_17_llm_schema_ref_inlining_and_boundary_sanitization` | Phase 2 (Fork Checkpoint) |
| `2026-10-17` | [`2026-10-17-tool-output-json-sandboxing-and-documents-plugin-fixes.md`](reproduced/2026-10-17-tool-output-json-sandboxing-and-documents-plugin-fixes.md) | `Flaw` | Tool Output JSON Sandboxing and Documents Plugin Remediation | `rfc_2026_10_17_tool_output_json_sandboxing_and_documents_plugin_fixes` | Phase 2 (Fork Checkpoint) |
| `2026-10-17` | [`2026-10-17-local-storage-concurrency-locks-and-ephemeral-dir-leak.md`](reproduced/2026-10-17-local-storage-concurrency-locks-and-ephemeral-dir-leak.md) | `Flaw` | LocalStorage Concurrency Synchronization and EphemeralDir Leak Fix | `rfc_2026_10_17_local_storage_concurrency_locks_and_ephemeral_dir_leak` | Phase 2 (Fork Checkpoint) |
| `2026-10-17` | [`2026-10-17-telemetry-reloadable-filter-and-async-gui-error-channel.md`](reproduced/2026-10-17-telemetry-reloadable-filter-and-async-gui-error-channel.md) | `Flaw` | Telemetry Reloadable Filter Fix and Async GUI Error Channel | `rfc_2026_10_17_telemetry_reloadable_filter_and_async_gui_error_channel` | Phase 2 (Fork Checkpoint) |
| `2026-10-17` | [`2026-10-17-process-lifecycle-browser-cleanup-and-durable-pubsub-ack.md`](reproduced/2026-10-17-process-lifecycle-browser-cleanup-and-durable-pubsub-ack.md) | `Flaw` | External Process Lifecycle, Browser Cleanup, and Durable Pub/Sub Acknowledgement | `rfc_2026_10_17_process_lifecycle_browser_cleanup_and_durable_pubsub_ack` | Phase 2 (Fork Checkpoint) |

---

### 3. Approved (`approved/`)

| Date | RFC | Type | Title | Target Crates |
| :--- | :--- | :--- | :--- | :--- |
| _None currently queued_ | | | | |

---

### 4. Implemented (`implemented/`)

| Date | RFC | Title | Retrospective |
| :--- | :--- | :--- | :--- |
| `2026-07-20` | [`2026-07-20-plugin-test-harness.md`](implemented/2026-07-20-plugin-test-harness.md) | Localized Plugin Test Harness via Static Bundling | - |
| `2026-07-20` | [`2026-07-20-split-document-provider.md`](implemented/2026-07-20-split-document-provider.md) | Split Document Provider Functionality from ChatPlugin | - |
| `2026-07-23` | [`2026-07-23-mcp-connector.md`](implemented/2026-07-23-mcp-connector.md) | Model Context Protocol (MCP) Integration Plugin | - |
| `2026-09-04` | [`2026-09-04-google-bot-migration.md`](implemented/2026-09-04-google-bot-migration.md) | Google First-Class Bot Migration for Google Plugins and Deployment Scripts | [`2026-09-04-google-bot-migration-retro.md`](retrospectives/2026-09-04-google-bot-migration-retro.md) |
| `2026-09-09` | [`2026-09-09-consolidated-engine-registries.md`](implemented/2026-09-09-consolidated-engine-registries.md) | Consolidated Engine Registries Container | - |
| `2026-09-09` | [`2026-09-09-opaque-llm-executor.md`](implemented/2026-09-09-opaque-llm-executor.md) | Encapsulate LLM Execution via Opaque LlmExecutor Handle | - |
| `2026-09-09` | [`2026-09-09-opaque-storage-handle.md`](implemented/2026-09-09-opaque-storage-handle.md) | Encapsulate Storage Subsystem via Opaque Storage Handle | - |
| `2026-09-09` | [`2026-09-09-simplified-llm-prompt-definition.md`](implemented/2026-09-09-simplified-llm-prompt-definition.md) | Direct Typed LLM Client Construction | - |
| `2026-09-09` | [`2026-09-09-streamlined-plugin-registration.md`](implemented/2026-09-09-streamlined-plugin-registration.md) | Streamlined Batch Plugin Registration | - |
| `2026-09-17` | [`2026-09-17-optional-decision-plugin.md`](implemented/2026-09-17-optional-decision-plugin.md) | Optional Decision Subsystem and First-Class Decision Plugin Architecture | [`2026-09-18-optional-decision-plugin-retro.md`](retrospectives/2026-09-18-optional-decision-plugin-retro.md) |
| `2026-09-18` | [`2026-09-18-fluent-bundle-builder.md`](implemented/2026-09-18-fluent-bundle-builder.md) | Fluent Builder Pattern for Bundle Composition and Dedicated Singleton Decision Provider | - |
| `2026-09-19` | [`2026-09-19-active-working-memory.md`](implemented/2026-09-19-active-working-memory.md) | Active Working Memory for Tool Outputs and Multi-Turn Reasoning | - |
| `2026-09-24` | [`2026-09-24-pluggable-singleton-llm-provider.md`](implemented/2026-09-24-pluggable-singleton-llm-provider.md) | Pluggable Singleton LLM Provider and Core Decoupling | [`2026-09-24-pluggable-singleton-llm-provider-retro.md`](retrospectives/2026-09-24-pluggable-singleton-llm-provider-retro.md) |
| `2026-10-17` | [`2026-10-17-multi-tool-turn-barrier-and-working-memory-alignment.md`](implemented/2026-10-17-multi-tool-turn-barrier-and-working-memory-alignment.md) | Multi-Tool Turn Barrier and Working Memory Alignment | [`2026-10-19-multi-tool-turn-barrier-and-working-memory-alignment-retro.md`](retrospectives/2026-10-19-multi-tool-turn-barrier-and-working-memory-alignment-retro.md) |
