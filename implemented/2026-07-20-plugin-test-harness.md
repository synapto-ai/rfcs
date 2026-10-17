# [MVP] RFC: Localized Plugin Test Harness via Static Bundling

## 1. User-Centric Motivation & Goal

Currently, to run scenario-based tests using our YAML test framework, we must rely on a monolithic `test-bundle` that mocks everything. If a developer is working on the `documents` plugin, they cannot easily write a scenario that tests their *real* plugin implementation integrated with the system's mocked peripherals.

The goal of this RFC is to allow developers to drop a `scenario.yaml` directly into their plugin's `scenarios/` folder and run it against their actual plugin code. This provides robust, end-to-end confidence for individual components without spinning up the entire production system or relying solely on isolated unit tests.

## 2. Executive Summary

- **Context:** The project uses a YAML-driven scenario coordinator in `synapto/test-harness`. However, this coordinator and the accompanying `Mock` plugins are locked inside a binary crate (`src/main.rs`).
- **Motivation:** We need to verify plugin logic (e.g., RAG retrieval in `documents`) using real channel orchestration and state lifecycles, but with controlled (mocked) I/O and external boundaries.
- **Overarching Goals:** Establish a decentralized, localized, scenario-driven integration testing pattern for all Synapto plugins, utilizing our standard static bundling architecture.
- **Purpose:** Refactor `test-harness` to export its core components and establish a template for plugin-specific test bundles.
- **Problem:** The `test-harness` is heavily coupled as a singular application. Other crates cannot import `ScenarioCoordinator`, `ScenarioStep`, or the pre-built `MockChatPlugin`, etc.
- **Solution:** Extract the generic components of `test-harness` into a `lib.rs`. Then, inside target plugins (e.g., `ai/plugins/documents`), define a local static test bundle (`tests/test_bundle.rs`) that imports the harness library, composes the real plugin with required mocks, and executes the local scenarios.

## 3. Current State Analysis

Presently, `synapto/test-harness/src/main.rs` contains:
- `ScenarioCoordinator`, `ScenarioManifest`, `ScenarioStep` schemas.
- Implementations for `MockAudioInputPlugin`, `MockDocumentsPlugin`, `MockChatPlugin`, etc.
- The `Synapto::run::<...>()` entry point that statically wires all these mocks together.
- A global `ACTIVE_COORDINATOR: OnceLock<Arc<ScenarioCoordinator>>` that the mock plugins use to communicate with the scenario driver.

**Flaws & Tight Coupling:**
- **Trapped Logic:** Because these are in `main.rs`, no other crate in the workspace can import the mocks or the scenario runner.
- **Inverted Control for Testing:** The test-harness dictates the entire bundle. A plugin cannot inject itself into the harness without modifying the global `test-harness` codebase, which ruins isolation and turns the test-harness into a messy kitchen sink of conditional logic if we try to support testing every plugin there.

## 4. Structural Hierarchy

- **`synapto-test-harness` (Library):** 
  - **Constraints:** Must not contain concrete plugin logic. Purely a provider of the scenario execution engine and standardized boundary mocks.
  - **Atomic Responsibility:** Parsing YAML scenarios, driving the step state machine, and providing dummy `Synapto` plugins (`MockChat`, `MockAudio`) for dependency injection.
- **`ai/plugins/<plugin-name>/examples/test_bundle.rs` (Binary):**
  - **Constraints:** Must only exist as a `dev-dependency` orchestration layer.
  - **Atomic Responsibility:** Assembling a bespoke `Synapto::run` type signature that combines the local *real* plugin with the imported *mock* plugins from the test harness library.

## 5. Data Models

The fundamental data models (like `ScenarioStep` and `ScenarioManifest`) remain unchanged but are shifted to a public module boundary.

### Semantic Schemas (Test Harness)

```rust
pub struct ScenarioManifest {
    pub name: String,
    pub timeout_secs: u64,
    pub steps: Vec<ScenarioStep>,
}

pub enum ScenarioStep {
    UserWrites { text: String, attachments: Vec<String> },
    UserSays { transcript: String },
    AwaitResponse { 
        assert_contains: Option<String>,
        assert_all: Option<Vec<String>>,
        assert_any: Option<Vec<String>>,
        case_sensitive: Option<bool>,
        timeout_secs: Option<u64>,
    },
    PlayAudio { audio_stream: String },
    Wait { millis: u64 },
}
```

### Static Bundle Definition (Consumer)
The key structural boundary is the **Static Bundle Definition** inside the plugin's test binary. For example, in `documents/examples/test_bundle.rs`, the execution signature will look like this:

```rust
// Localized test bundle definition relying on SSoT bundling protocol
Synapto::<
    datadir_ephemeral::EphemeralDir,
    (synapto::config::DotEnv, synapto::config::Env),
    storage_local::LocalStorage,
    synapto::prompt_provider::EmptyPromptProvider,
>::run::<(
    test_harness::MockAudioInputPlugin,
    documents::DocumentsPlugin, // <-- THE REAL PLUGIN
    test_harness::MockChatPlugin,
    test_harness::MockSlowReadPlugin,
    test_harness::MockTtsPlugin,
    test_harness::MockSttPlugin,
    test_harness::MockDiarizationPlugin,
)>()
.await
```

### JSON Contract Examples
Although this RFC primarily shifts existing code across module boundaries, the fundamental data contract remains the scenario YAML payload (which deserializes cleanly to JSON).

```json
{
  "name": "document-retrieval",
  "timeout_secs": 30,
  "steps": [
    {
      "UserWrites": {
        "text": "What is in the TACR document?",
        "attachments": ["/tmp/tacr.pdf"]
      }
    },
    {
      "AwaitResponse": {
        "assert_contains": "architecture"
      }
    }
  ]
}
```

No new LLM models or system state variants are introduced. The `!LLMSafe` boundaries defined by the target plugins remain intact since the true plugin is executed within the standard Synapto bootup sequence.

## 6. Component Architecture

### Execution Flow
1. The developer executes `cargo run --example test-bundle -- scenarios/my-scenario.yaml` from within their plugin directory.
2. The `test_bundle.rs` binary parses the YAML and initializes the `ScenarioCoordinator`.
3. The global `ACTIVE_COORDINATOR` lock is initialized (as it currently works today in `test-harness`).
4. A background Tokio task is spawned to drive the `ScenarioCoordinator::drive` execution loop.
5. The main thread invokes the bespoke `Synapto::run::<...>()` bundle.
6. The `Mock` plugins rely on the `ACTIVE_COORDINATOR` global state to receive simulated input (e.g., text, audio) and capture generated output (e.g., TTS audio, agent messages) to satisfy assertions.
7. The *real* plugin processes these events transparently over standard Synapto channels (e.g., `State`, `InteractionMemory`).

## 7. Architectural Diagrams

```mermaid
flowchart TD
    subgraph Plugin Test Crate [ai/plugins/documents]
        TBR(test_bundle.rs)
        YAML(scenario.yaml)
    end
    
    subgraph test-harness [synapto/test-harness (lib)]
        SC[ScenarioCoordinator Task]
        Mocks(MockAudioInput, MockChat, MockTTS)
    end
    
    subgraph Synapto Runtime [synapto]
        Runtime(Synapto::run)
        Channels(Standard Channels)
    end
    
    RealPlugin(DocumentsPlugin)

    TBR -->|Reads| YAML
    TBR -->|Initializes| SC
    TBR -->|Spawns| Runtime
    
    Runtime -.->|Loads| RealPlugin
    Runtime -.->|Loads| Mocks
    
    SC <==>|Global OnceLock| Mocks
    Mocks <==>|Standard Synapto Interfaces| Channels
    RealPlugin <==>|Standard Synapto Interfaces| Channels
```

## 8. Robustness & Pragmatic Security Trade-offs

- **Hazard: Global State Pollution.** The `test-harness` relies on `ACTIVE_COORDINATOR` (a global `OnceLock`). This means tests cannot easily be run in parallel within the same process.
  - **Accepted Risk (MVP):** Since we are invoking distinct `cargo run --bin test-bundle` processes for scenario testing, intra-process test collision is not an issue. We accept this global state dependency for now to keep the MVP strictly focused on structural code movement rather than redesigning the entire mock orchestration logic.
- **Hazard: Channel Deadlocks.** If a mock plugin fails to read from a channel or the real plugin panics, the `ScenarioCoordinator` might hang indefinitely waiting for an `AwaitResponse`.
  - **Accepted Risk (MVP):** The `ScenarioManifest` provides a global `timeout_secs`, which eventually kills the test. We accept that debugging deadlocks might still be tricky, but complex drop guards or channel introspection are out of scope for this library extraction.

## 9. Systemic Consequences

- **Breaking Changes:** No breaking changes to existing Synapto architecture.
- **Impact on test-harness:** `test-harness` will be refactored from a pure binary into a hybrid library/binary (`lib.rs` and `main.rs`). The global `test-bundle` will still work exactly as before, but its internals will be exposed.
- **Impact on plugins:** Plugins will now require `synapto-test-harness` as a `dev-dependency` and will carry a small `test_bundle.rs` binary.

## 10. Definition of Done (DoD)

- [ ] Move `plugins`, `ScenarioCoordinator`, `ScenarioManifest`, `ScenarioStep`, and `ACTIVE_COORDINATOR` logic from `synapto/test-harness/src/main.rs` to `synapto/test-harness/src/lib.rs`.
- [ ] Export necessary components publicly from `lib.rs`.
- [ ] Ensure the global `test-bundle` (`synapto/test-harness/src/main.rs`) still compiles and runs existing scenarios.
- [ ] Add `test-harness` as a `dev-dependency` to `ai/plugins/documents/Cargo.toml`.
- [ ] Create `ai/plugins/documents/examples/test_bundle.rs` combining `DocumentsPlugin` and imported mocks.
- [ ] Create a `scenario.yaml` in `ai/plugins/documents/scenarios/` that tests RAG retrieval.
- [ ] Execute `cargo run --example test-bundle -- scenarios/document-summary/scenario.yaml` successfully from the documents directory.