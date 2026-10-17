# [MVP] RFC: Encapsulate Storage Subsystem via Opaque Storage Handle

## 1. Motivation and Goal

Currently, the `synapto-interface` crate exposes internal storage mechanics across multiple separate types:
1. `StorageRegistry` (a low-level, type-erased resource map for database connection pooling).
2. `StorageConfigResolver` (a raw trait object for resolving storage JSON configurations).

These two components are stored as separate public `Arc` references in `PluginInitContext`, `StorageConnection`, and the Core runtime. This exposes internal wiring details to plugin developers and complicates constructor signatures.

The goal of this RFC is to encapsulate all storage resolution and connection pooling into a single, opaque `StorageHandle` struct in `synapto-interface`.

## 2. Executive Summary

- **Context:** Plugins initialize database connections via `context.store::<S>()` during `Plugin::create`.
- **Problem:**
  - `StorageRegistry` and `dyn StorageConfigResolver` expose low-level container internals directly on `PluginInitContext` and `StorageConnection::connect`.
  - Construction of `PluginInitContext` requires passing multiple storage primitives (`Arc<StorageRegistry>`, `Arc<dyn StorageConfigResolver>`).
- **Solution:**
  1. Combine `StorageRegistry` and `StorageConfigResolver` inside an opaque `StorageHandle` struct in `synapto-interface::storage`.
  2. Hide `StorageRegistry` and `StorageConfigResolver` behind private fields or `#[doc(hidden)]` internals.
  3. Simplify `StorageConnection::connect` to accept `&StorageHandle` and `plugin_namespace: &str`.
  4. Simplify `PluginInitContext` fields by replacing two storage parameters with one `StorageHandle`.

## 3. Current State Analysis

Currently, storage initialization across interfaces is structured as follows:

```rust
// In synapto_interface::storage
pub struct StorageRegistry {
    map: tokio::sync::Mutex<HashMap<TypeId, Arc<dyn Any + Send + Sync>>>,
}

pub trait StorageConfigResolver: Send + Sync + 'static {
    fn resolve_config(&self, crate_name: &str, storage_type_name: &str) -> Option<serde_json::Value>;
}

pub trait StorageConnection: Send + Sync + Sized + 'static {
    type Config: serde::de::DeserializeOwned + Send + Sync;

    async fn connect(
        config: Self::Config,
        storage_registry: std::sync::Arc<StorageRegistry>, // <-- Internal resource map exposed
        plugin_namespace: &str,
    ) -> Result<Self, String>;
}
```

```rust
// In synapto_interface::plugin
pub struct PluginInitContext<'a> {
    llm_executor: crate::llm::LlmExecutor,
    plugin_config: &'a serde_json::Value,
    storage: std::sync::Arc<crate::storage::StorageRegistry>, // <-- Leaked internal
    plugin_namespace: &'a str,
    storage_config_resolver: std::sync::Arc<dyn crate::storage::StorageConfigResolver>, // <-- Leaked internal
}
```

### Architectural Deficiencies

1. **Information Leakage:** Exposes `StorageRegistry` and dynamic configuration resolution logic outside the storage boundary.
2. **Constructor Bloat:** `PluginInitContext::new` requires 5 arguments, including multiple internal storage primitives.
3. **Coupled Connection Contract:** Storage adapters (`surrealdb`, `firestore`, `local`) directly depend on `Arc<StorageRegistry>` instead of a domain storage handle.

## 4. Structural Hierarchy

- **`StorageHandle` (Opaque Struct):**
  - **Location:** `synapto-interface::storage`.
  - **Responsibility:** Holds the shared connection pool registry and the configuration resolver. Provides high-level connection initialization and pooling without exposing collection maps or trait objects.
- **`StorageConnection` (Trait):**
  - **Location:** `synapto-interface::storage`.
  - **Responsibility:** Defines how storage adapters connect using a `StorageHandle`.
- **`PluginInitContext` (Struct):**
  - **Location:** `synapto-interface::plugin`.
  - **Responsibility:** Holds a single `StorageHandle` instance and delegates `context.store::<S>()` to it.

## 5. Data Models and API Changes

### `synapto-interface::storage`

```rust
/// Opaque handle encapsulating storage connection pooling and configuration resolution.
#[derive(Clone)]
pub struct StorageHandle {
    registry: std::sync::Arc<StorageRegistry>,
    resolver: std::sync::Arc<dyn StorageConfigResolver>,
}

impl StorageHandle {
    pub fn new(
        resolver: std::sync::Arc<dyn StorageConfigResolver>,
    ) -> Self {
        Self {
            registry: std::sync::Arc::new(StorageRegistry::default()),
            resolver,
        }
    }

    pub fn with_registry(
        registry: std::sync::Arc<StorageRegistry>,
        resolver: std::sync::Arc<dyn StorageConfigResolver>,
    ) -> Self {
        Self { registry, resolver }
    }

    /// Resolves configuration and establishes a scoped storage connection.
    pub async fn connect_store<S: StorageConnection>(&self, plugin_namespace: &str) -> Result<std::sync::Arc<S>, String> {
        let full_path = std::any::type_name::<S>();
        let crate_name = full_path
            .split("::")
            .next()
            .unwrap_or("")
            .to_string()
            .replace('-', "_");
        let base_path = full_path.split('<').next().unwrap_or(full_path);
        let storage_type_name = base_path.split("::").last().unwrap_or("").to_string();

        let config_val = self
            .resolver
            .resolve_config(&crate_name, &storage_type_name)
            .unwrap_or_else(|| serde_json::json!({}));

        let config: S::Config = serde_json::from_value(config_val).map_err(|e| {
            format!(
                "Failed to parse config for storage '{}::{}': {}",
                crate_name, storage_type_name, e
            )
        })?;

        let store = S::connect(config, self, plugin_namespace).await?;
        Ok(std::sync::Arc::new(store))
    }

    /// Accesses or initializes a pooled storage resource safely.
    pub async fn get_or_init_pool<T: StorageProviderPool, F, Fut>(
        &self,
        init: F,
    ) -> Result<std::sync::Arc<T>, String>
    where
        F: FnOnce() -> Fut,
        Fut: std::future::Future<Output = Result<T, String>>,
    {
        self.registry.get_or_init(init).await
    }
}

#[async_trait]
pub trait StorageConnection: Send + Sync + Sized + 'static {
    type Config: serde::de::DeserializeOwned + Send + Sync;

    async fn connect(
        config: Self::Config,
        storage_handle: &StorageHandle,
        plugin_namespace: &str,
    ) -> Result<Self, String>;
}
```

### `synapto-interface::plugin::PluginInitContext`

```rust
pub struct PluginInitContext<'a> {
    llm_executor: crate::llm::LlmExecutor,
    plugin_config: &'a serde_json::Value,
    storage: crate::storage::StorageHandle,
    plugin_namespace: &'a str,
}

impl<'a> PluginInitContext<'a> {
    pub fn new(
        llm_executor: crate::llm::LlmExecutor,
        plugin_config: &'a serde_json::Value,
        storage: crate::storage::StorageHandle,
        plugin_namespace: &'a str,
    ) -> Self {
        Self {
            llm_executor,
            plugin_config,
            storage,
            plugin_namespace,
        }
    }

    pub async fn store<S: crate::storage::StorageConnection>(&self) -> Result<std::sync::Arc<S>, String> {
        self.storage.connect_store::<S>(self.plugin_namespace).await
    }
}
```

## 6. Side-by-Side API Comparison

### 1. `StorageConnection` Trait Definition

```rust
// CURRENT (Exposes raw Arc<StorageRegistry>)
#[async_trait]
pub trait StorageConnection: Send + Sync + Sized + 'static {
    type Config: serde::de::DeserializeOwned + Send + Sync;

    async fn connect(
        config: Self::Config,
        storage_registry: std::sync::Arc<StorageRegistry>,
        plugin_namespace: &str,
    ) -> Result<Self, String>;
}
```

```rust
// PROPOSED (Uses domain StorageHandle)
#[async_trait]
pub trait StorageConnection: Send + Sync + Sized + 'static {
    type Config: serde::de::DeserializeOwned + Send + Sync;

    async fn connect(
        config: Self::Config,
        storage: &StorageHandle,
        plugin_namespace: &str,
    ) -> Result<Self, String>;
}
```

### 2. Storage Provider Implementation (SurrealDB Example)

```rust
// CURRENT (SurrealDB Provider)
#[async_trait]
impl<P: DataDirProvider> StorageConnection for SurrealStorage<P> {
    type Config = EmptyStorageConfig;

    async fn connect(
        _config: Self::Config,
        storage_registry: std::sync::Arc<synapto_interface::storage::StorageRegistry>,
        plugin_namespace: &str,
    ) -> Result<Self, String> {
        let provider = storage_registry
            .get_or_init::<SurrealProvider<P>, _, _, String>(|| async move {
                SurrealProvider::<P>::new().await
            })
            .await?;

        provider.connection_for(plugin_namespace).await
    }
}
```

```rust
// PROPOSED (SurrealDB Provider)
#[async_trait]
impl<P: DataDirProvider> StorageConnection for SurrealStorage<P> {
    type Config = EmptyStorageConfig;

    async fn connect(
        _config: Self::Config,
        storage: &StorageHandle,
        plugin_namespace: &str,
    ) -> Result<Self, String> {
        let provider = storage
            .get_or_init_pool(|| async move {
                SurrealProvider::<P>::new().await
            })
            .await?;

        provider.connection_for(plugin_namespace).await
    }
}
```

### 3. `PluginInitContext` Constructor and Fields

```rust
// CURRENT (5 fields, leaked container internals)
pub struct PluginInitContext<'a> {
    llm_executor: crate::llm::LlmExecutor,
    plugin_config: &'a serde_json::Value,
    storage: std::sync::Arc<crate::storage::StorageRegistry>,
    plugin_namespace: &'a str,
    storage_config_resolver: std::sync::Arc<dyn crate::storage::StorageConfigResolver>,
}

impl<'a> PluginInitContext<'a> {
    pub fn new(
        llm_executor: crate::llm::LlmExecutor,
        plugin_config: &'a serde_json::Value,
        storage: std::sync::Arc<crate::storage::StorageRegistry>,
        plugin_namespace: &'a str,
        storage_config_resolver: std::sync::Arc<dyn crate::storage::StorageConfigResolver>,
    ) -> Self;
}
```

```rust
// PROPOSED (4 clean fields, zero leaked internals)
pub struct PluginInitContext<'a> {
    llm_executor: crate::llm::LlmExecutor,
    plugin_config: &'a serde_json::Value,
    storage: crate::storage::StorageHandle,
    plugin_namespace: &'a str,
}

impl<'a> PluginInitContext<'a> {
    pub fn new(
        llm_executor: crate::llm::LlmExecutor,
        plugin_config: &'a serde_json::Value,
        storage: crate::storage::StorageHandle,
        plugin_namespace: &'a str,
    ) -> Self;
}
```

### Key Simplifications

1. **Removed Generic Parameter Noise**: `storage.get_or_init_pool(...)` infers pool types automatically without explicit `<Type, _, _, String>` type annotations.
2. **Encapsulated Configuration Dispatch**: `PluginInitContext` no longer stores or exposes `dyn StorageConfigResolver`.
3. **Reduced Constructor Arity**: `PluginInitContext::new` decreases from 5 parameters to 4 parameters.
4. **No Raw Pointer / Arc Boilerplate**: Replaces `std::sync::Arc<StorageRegistry>` with a standard `&StorageHandle` reference.

## 7. Component Architecture

```mermaid
flowchart TD
    subgraph Core [synapto-core]
        Resolver[ConfigProvider Storage Resolver]
        Handle[StorageHandle]
        Resolver --> Handle
    end

    subgraph Interface [synapto-interface]
        Context[PluginInitContext]
        Handle --> Context
    end

    subgraph Plugins [Plugins]
        P[Plugin::create]
        Context -->|context.store::<S>()| P
    end

    subgraph StorageAdapters [Storage Adapters]
        Adapter[SurrealDB / Firestore]
        P -->|connect(config, handle)| Adapter
        Adapter -->|handle.get_or_init_pool()| Handle
    end
```

## 8. Migration and Implementation Plan

1. **Phase 1: Update `synapto-interface::storage`**:
   - Define `StorageHandle` with connection and pool delegation.
   - Update `StorageConnection::connect` signature to accept `&StorageHandle`.
   - Update `PluginInitContext` to hold `StorageHandle`.
2. **Phase 2: Update Storage Adapters**:
   - Update `synapto-storage-surrealdb`, `synapto-storage-firestore`, and `synapto-datadir-local` implementations of `StorageConnection::connect` to use `storage_handle.get_or_init_pool(...)`.
3. **Phase 3: Update `synapto-core`**:
   - Update `Synapto` initialization to instantiate `StorageHandle` from the configuration provider.
   - Pass `StorageHandle` into `PluginInitContext::new`.
4. **Phase 4: Update Tests and Fixtures**:
   - Update all test harnesses to use `StorageHandle::new(...)`.
