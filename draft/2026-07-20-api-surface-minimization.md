# [DRAFT] RFC: API Surface Minimization

> **Draft Status:** Phase 6 — All 10 sections drafted; awaiting Lead Architect review and approval to move to `approved/2026-07-20-api-surface-minimization.md`.
> **Next Step on Resume:** Perform Pre-Implementation Recheck against current `PluginInitContext` in `synapto-interface` and seek explicit Phase 6 approval from the Lead Architect.

## 1. User-Centric Motivation & Goal

When developers build plugins, they shouldn't have to worry about accidentally interacting with low-level system internals.

The goal of this RFC is to create a robust, intermediate architectural step that strictly minimizes the API surface exposed to plugin developers during initialization. By introducing a generic, restrictive `PluginInitContext`, we reduce the cognitive load for developers and prevent unauthorized access to core internals, paving the way for future macro-driven automation with a clean, secure boundary.

## 2. Executive Summary

- **Context:** The `PluginContext` struct currently exposes internal system state (like global context subscription receivers) directly to the `Plugin::create` constructor.
- **Motivation:** Direct access to `PluginContext` violates the Principle of Least Privilege and pollutes the plugin API boundary with internals.
- **Overarching Goals:** Restrict the plugin's access to the Core engine solely to what is explicitly required (config, storage, LLM).
- **Purpose:** Introduce `PluginInitContext` to completely hide core internals.
- **Problem:** Exposing internal methods like `subscribe_context_updates` directly in the constructor payload creates poor Developer Experience (DX) and breaks encapsulation.
- **Solution:** Change the `Plugin::create` signature to accept a restrictive `&PluginInitContext`.

## 3. Current State Analysis

Presently, `Plugin::create` accepts the raw `PluginContext`:
```rust
async fn create(context: PluginContext) -> Result<Self, String>;
```
This raw context includes methods like `subscribe_context_updates`, which are internal routing features that plugins should not touch directly during initialization. Exposing these internals makes the API surface unnecessarily large and couples plugins to the core implementation details.

## 4. Structural Hierarchy

- **Core Orchestrator (`Synapto::run_internal`):**
  - **Constraints:** Unchanged. It still constructs the raw `PluginContext` but now wraps it in a `PluginInitContext` before passing it to `Plugin::create`.
- **`synapto_interface::plugin`:**
  - **Constraints:** Must define the restrictive `PluginInitContext` wrapper.
  - **Atomic Responsibility:** `PluginInitContext` is strictly responsible for providing config parsing, database connections, and LLM executors. It holds zero business logic and obscures all other internal Core systems.

## 5. Data Models

The API surface is minimized by the `PluginInitContext`:

```rust
// In synapto_interface::plugin

/// A restrictive proxy wrapper that explicitly prevents access to low-level 
/// core internals (e.g., routing keys, raw subscriptions).
pub struct PluginInitContext<'a> {
    inner: &'a PluginContext,
}

impl<'a> PluginInitContext<'a> {
    pub fn new(inner: &'a PluginContext) -> Self { Self { inner } }

    pub fn config<C: serde::de::DeserializeOwned>(&self) -> Result<C, String> {
        self.inner.config()
    }
    
    pub async fn store<S: crate::storage::StorageConnection>(&self) -> Result<S, String> {
        self.inner.store::<S>().await
    }
    
    pub fn llm_executor(&self) -> std::sync::Arc<dyn crate::llm::LlmExecutor> {
        self.inner.llm_executor()
    }
}
```

## 6. Component Architecture

### The Developer Experience
The only change for the plugin developer is the signature of `create` and the methods available on the context object. They continue to manage their own channels and state in the `start` methods as they do today until the macro solution is implemented.

```rust
pub struct GoogleChatPlugin {
    config: GoogleChatConfig,
    channels: std::sync::Arc<tokio::sync::Mutex<GoogleChatChannels>>,
}

#[async_trait]
impl Plugin for GoogleChatPlugin {
    // Note the minimized API signature!
    async fn create(context: &PluginInitContext<'_>) -> Result<Self, String> {
        let config: GoogleChatConfig = context.config()?;
        Ok(Self {
            config,
            channels: std::sync::Arc::new(tokio::sync::Mutex::new(GoogleChatChannels::default())),
        })
    }
}
```

## 7. Architectural Diagrams

```mermaid
flowchart TD
    subgraph Core
        CoreRun[Synapto::run_internal]
        RawCtx[Raw PluginContext]
        InitCtx[PluginInitContext Proxy]
        
        CoreRun --> RawCtx
        RawCtx --> InitCtx
    end
    
    subgraph Plugin API Boundary
        PCreate[Plugin::create]
        InitCtx -->|Strict API| PCreate
        PCreate -->|config()| Cfg[Config]
        PCreate -->|store()| DB[Database]
    end
```

## 8. Robustness & Pragmatic Security Trade-offs

- **Hazard: Over-restriction.** If a plugin genuinely requires an advanced internal feature during initialization, it is now blocked from accessing it.
  - **Accepted Risk:** This is the intended behavior. If a new capability is needed, it should be explicitly modeled as a generic capability trait or explicitly added to the `PluginInitContext` by the core architects, rather than implicitly accessed.

## 9. Systemic Consequences

- **Breaking Change:** This changes the `Plugin::create` trait signature across the entire workspace from `(context: PluginContext)` to `(context: &PluginInitContext<'_>)`. All existing plugins (e.g., `google-chat`, `task-memory`, `semantic-memory`, etc.) must be updated.
- **Architectural Cleanup:** Completely prevents developers from accidentally interacting with or depending on internal Core channels (`subscribe_context_updates`).

## 10. Definition of Done (DoD)

- [ ] Implement `PluginInitContext` in `synapto_interface::plugin` as a wrapper over `PluginContext`.
- [ ] Update the `Plugin` trait signature in `synapto_interface` to accept `&PluginInitContext<'_>`.
- [ ] Update `Synapto::run_internal` to wrap `PluginContext` before calling `Plugin::create`.
- [ ] Refactor all plugins in `synapto` and `synapto-plugins` workspaces to use the new `&PluginInitContext<'_>` signature.
