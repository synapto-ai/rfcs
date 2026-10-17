# [MVP] Retrospective: Pluggable Singleton LLM Provider and Core Decoupling

## 1. Context & Evolution
The implementation successfully decoupled LLM execution from `synapto-core`. Previously, `synapto-core` hardcoded Google Gemini and Vertex AI execution via `ConcreteLlmExecutor` in `synapto-llm`, polluting `Config` with Google Cloud fields and preventing alternative provider implementations.

Through RFC `pluggable-singleton-llm-provider.md`, the architecture established `LlmProvider` in `synapto-interface::llm` and updated `SynaptoBuilder` to enforce singular mandatory LLM provider configuration at compile time via typestate markers `NoLlm` and `WithLlm<L>`.

## 2. Iterative Architectural Review & Corrections
During zero-context subagent reviews, several critical issues were discovered and proactively remediated prior to final implementation:

1. **Chicken-and-Egg Lifecycle Paradox in `Synapto` Construction:**
   - *Problem:* `LlmSetup::setup(&mut synapto)` originally attempted to configure the LLM executor on an existing mutable `Synapto` instance. However, `Synapto` could not construct itself without an already-initialized non-empty `LlmExecutor`.
   - *Solution:* Refactored `LlmSetup::setup(&C, &CredentialsHandle)` to run before `Synapto` instantiation and introduced `Synapto::with_config_and_llm(config_provider, credentials, llm_executor, error_rx, tracing)`.
   - *Attribution:* Zero-Context Subagent (Review 1 & 2).

2. **Crate Boundary Segregation (`synapto-llm` vs `synapto-llm-google`):**
   - *Problem:* Initial draft contemplated removing `synapto-llm` from `synapto-core`, which would break core cognitive prompt and tool compilation.
   - *Solution:* Split responsibilities cleanly: `synapto-llm` remains in core as the vendor-neutral prompt DSL and client abstraction (`LLMClient`, `Instruction`, `ToolExecutor`), while `ConcreteLlmExecutor` was moved to `integrations/llm-providers/google` as `GoogleLlm`.
   - *Attribution:* Zero-Context Subagent (Review 1).

3. **Strict Rule 27 Compliance (Unified Credentials Provider):**
   - *Problem:* Early draft retained `gcp_auth` and static API key fields in `GoogleLlmConfig`.
   - *Solution:* Removed `gcp_auth` and static key fields; `GoogleLlm` resolves bearer tokens exclusively via `CredentialsHandle` using `GoogleCloudTarget`.
   - *Attribution:* Zero-Context Subagent (Review 2).

4. **Scenario Testing Compatibility (`MockLlm`):**
   - *Problem:* Deserializing `()` from `{}` caused runtime parsing panics across tests.
   - *Solution:* Implemented `MockLlmConfig` deriving `Deserialize, Default`, allowing zero-config tests to boot cleanly.
   - *Attribution:* Zero-Context Subagent (Review 3).

## 3. Scope & Verification
- `synapto` workspace compiled cleanly (`cargo check --workspace --tests`).
- `synapto-test` scenario test suite verified (`cargo test -p synapto-test --test scenario_tests`).
- `personal-assistant` bundle verified (`cargo check --config dev-config.toml`).
- `kid-assistant` bundle verified (`cargo check`).
- `bundles` workspace (home-assistant, rpg, teacher) verified (`cargo check`).
- `allusio-ai/synapto-plugins` scenario test suites verified (`cargo test --test scenario_tests`).
- `ARCHITECTURE.md` updated with Rule 33 and updated Rule 31 builder steps.
