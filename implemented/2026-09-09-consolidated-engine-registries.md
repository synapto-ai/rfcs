# [MVP] RFC: Consolidated Engine Registries Container

## 1. Motivation and Goal

Previously, the Core runtime passed 3 separate registry references (`ContextRegistries`, `ToolRegistryBuilder`, `CommandRegistryBuilder`) across every internal cognitive task. This caused parameter explosion in `cognitive::start`, `cognitive_direct_task`, and `cognitive_side_task`.

The goal of this RFC is to consolidate these three disjoint builders into a single, unified `EngineRegistries` container.

## 2. Executive Summary

- **Context:** Core tasks required 3 separate `Arc` parameters (`registries`, `tools`, `commands`).
- **Problem:**
  - High parameter count across internal task signatures.
  - Redundant atomic reference count operations when spawning background tasks.
  - Violation of architectural Rule 26 (Encapsulated Handles over raw `Arc` collections).
- **Solution:**
  1. Define `EngineRegistries` in `synapto_interface::context`.
  2. Encapsulate `ContextRegistries`, `ToolRegistryBuilder`, and `CommandRegistryBuilder` inside `EngineRegistries`.
  3. Replace the 3 separate task parameters with a single `EngineRegistries` handle.

## 3. Structural Hierarchy

- **`EngineRegistries` (Struct):**
  - **Location:** `synapto-interface::context`.
  - **Responsibility:** Serves as the unified runtime container for all active cognitive context providers, dynamic tools, and command executors.
- **`ContextRegistries` (Struct):**
  - **Responsibility:** Manages temporal context streams (`Historical`, `Current`, `Prospective`).
- **`ToolRegistryBuilder` (Struct):**
  - **Responsibility:** Stores available static and dynamic tools.
- **`CommandRegistryBuilder` (Struct):**
  - **Responsibility:** Stores executable system commands.

## 4. Data Models and API Changes

### `synapto-interface::context`

```rust
/// Unified registries container for the cognitive engine.
#[derive(Clone, Default)]
pub struct EngineRegistries {
    pub context: std::sync::Arc<ContextRegistries>,
    pub tools: std::sync::Arc<crate::tool::ToolRegistryBuilder>,
    pub commands: std::sync::Arc<crate::command::CommandRegistryBuilder>,
}
```

### `synapto-core::cognitive`

```rust
pub(crate) async fn start<P: CognitivePromptProvider>(
    config: Config,
    llm_executor: synapto_interface::llm::LlmExecutor,
    trigger_cognitive_direct: CognitiveDirectTrigger,
    interrupt_cognitive_direct: CognitiveDirectInterrupt,
    cognitive_speaking_semaphore: Arc<tokio::sync::Semaphore>,
    text_rx: broadcast::Receiver<PeerInputText>,
    peer_input_speech_rx: mpsc::Receiver<PeerInputSpeech>,
    interaction_memory_rx: watch::Receiver<InteractionMemory>,
    cognitive_speech_tx: broadcast::Sender<CognitiveOutputSpeech>,
    new_interaction_tx: mpsc::Sender<Interaction>,
    video_rx: Option<watch::Receiver<CameraInputFrame>>,

    registries: synapto_interface::context::EngineRegistries,

    cognitive_output_text_tx: Option<mpsc::Sender<CognitiveOutputText>>,
    cognitive_state_tx: broadcast::Sender<CognitiveStateUpdate>,
    resolve_in_flight_tool_tx: mpsc::Sender<ToolCallId>,
);
```

## 5. Architectural Benefits

1. **Parameter Count Reduction**: Decreased task parameter counts across `cognitive.rs`, `direct.rs`, and `side.rs` by 2 arguments per signature.
2. **Reduced Atomic Operations**: Spawning internal tasks now clones a single handle.
3. **Encapsulation**: Keeps runtime dependencies unified under domain boundaries according to Rule 26.
