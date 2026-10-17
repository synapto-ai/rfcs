# [MVP] RFC: Streamlined Batch Plugin Registration

## 1. Motivation and Goal

Currently, the `Plugin::register` method requires plugins to make individual method calls on `PluginRegistry` for each provider, tool, command, and observer. Plugins with multiple capabilities must repeatedly clone `self` and sub-components.

The goal of this RFC is to introduce batch registration methods (`register_context_providers`, `register_tools`, `register_commands`) on `PluginRegistry`.

## 2. Executive Summary

- **Context:** Plugins register components into `PluginRegistry` during startup.
- **Problem:** Repetitive line-by-line registration and `.clone()` boilerplate across plugins (e.g. `TaskMemoryPlugin`, `DocumentsPlugin`).
- **Solution:**
  1. Add batch registration methods accepting iterators or slices to `PluginRegistry`.
  2. Simplify `Plugin::register` implementations across all plugins.

## 3. Side-by-Side API Comparison

### 1. `TaskMemoryPlugin` Registration

```rust
// CURRENT (Repetitive registration calls)
fn register<R: PluginRegistry + ?Sized>(self: Arc<Self>, registry: &mut R) {
    registry.register_interaction_observer(self.clone());
    registry.register_context_provider(self.tasks_provider.clone());
    registry.register_context_provider(self.goals_provider.clone());
    registry.register_context_provider(self.missions_provider.clone());
    registry.register_command(self.command_executor.clone());
}
```

```rust
// PROPOSED (Batch registration)
fn register<R: PluginRegistry + ?Sized>(self: Arc<Self>, registry: &mut R) {
    registry.register_interaction_observer(self.clone());
    registry.register_context_providers([
        self.tasks_provider.clone(),
        self.goals_provider.clone(),
        self.missions_provider.clone(),
    ]);
    registry.register_command(self.command_executor.clone());
}
```

### 2. `DocumentsPlugin` Registration

```rust
// CURRENT (Individual tool registrations)
fn register<R: synapto_interface::plugin::PluginRegistry + ?Sized>(
    self: std::sync::Arc<Self>,
    registry: &mut R,
) {
    registry.register_documents(self.clone());
    registry.register_context_provider(self.provider.clone());
    registry.register_tool(ReadDocumentPluginTool {
        read_document_tx: self.read_document_tx.clone(),
    });
    registry.register_tool(ReadUrlPluginTool {
        read_url_tx: self.read_url_tx.clone(),
    });
}
```

```rust
// PROPOSED (Batch tool registration)
fn register<R: synapto_interface::plugin::PluginRegistry + ?Sized>(
    self: std::sync::Arc<Self>,
    registry: &mut R,
) {
    registry.register_documents(self.clone());
    registry.register_context_provider(self.provider.clone());
    registry.register_tools([
        ReadDocumentPluginTool { read_document_tx: self.read_document_tx.clone() },
        ReadUrlPluginTool { read_url_tx: self.read_url_tx.clone() },
    ]);
}
```

## 4. Key Simplifications

1. **Reduced Boilerplate:** Groups multiple component registrations into declarative arrays.
2. **Backward Compatibility:** Keeps single-item registration methods while adding ergonomic batch extensions.
3. **Clearer Intent:** Groups related capabilities visually within the plugin registration block.

## 5. Migration Plan

1. Add default blanket methods `register_context_providers`, `register_tools`, and `register_commands` to `PluginRegistry` in `synapto-interface`.
2. Update plugins across `synapto-ai` and `allusio-ai` to use batch registration syntax.
