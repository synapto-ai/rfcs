# [MVP] [DRAFT] Pluggable Credentials Provider and Ambient Cloud Identity Resolution via Capability Archetypes

> **Draft Status:** Phase 6 — All 10 sections drafted (foundational credentials architecture already partially implemented in `synapto-interface::credentials`); awaiting Lead Architect review to either archive in `implemented/` or approve remaining items.
> **Next Step on Resume:** Reconcile DoD checklist against `synapto/crates/interface/src/credentials.rs` and `2026-10-17-strict-credentials-and-explicit-config-compliance.md`, and confirm state transition with the Lead Architect.

## 1. User-Centric Motivation & Goal
Users and operators want secure, simple system deployments without managing static secret files. In cloud environments like Google Cloud Platform (GCP), storing static service account JSON keys in configuration files creates security risks and operational friction. When an operator runs an assistant on a GCP Virtual Machine (VM) with an attached service account, the assistant must use ambient credentials from the VM metadata server automatically. When static credentials or API keys are required for other clouds or local development (such as Speechmatics, OpenAI, or ElevenLabs), the system must support them uniformly. A pluggable, type-safe credentials provider allows developers to compose authentication providers at compile-time (for example: `(GoogleCredentials, SpeechmaticsCredentials)`) while keeping core crates clean, vendor-neutral, and strictly decoupled.

## 2. Executive Summary
- **Context:** The system integrates with multiple external cloud APIs (Google Vertex AI, Google Cloud Speech-to-Text, Google Cloud Text-to-Speech, Google Chat, Speechmatics). Currently, authentication logic is duplicated inside individual plugin crates.
- **Motivation:** Duplicate authentication code creates maintenance overhead and security vulnerabilities. Plugins require static JSON private keys and cannot use ambient VM metadata server identities (Application Default Credentials / ADC). Core crates also lack a pluggable mechanism to inject authentication providers.
- **Overarching Goals:** Introduce standard protocol-neutral capability archetype traits (`ProvideBearerToken<Target>`, `ProvideApiKey<Target>`) in `synapto-interface`. Decouple concrete cloud authentication providers into integration crates (`integrations/credentials-providers/`). Enable ambient cloud identity resolution alongside explicit static configuration with clean Developer Experience (DX) and strict vendor neutrality.
- **Purpose:** Implement capability archetype traits, scoped credentials configuration loading, and `CredentialsHandle` in `synapto-interface` and `synapto-core`. Extend `Synapto` type-level composition to accept credentials providers, and implement dedicated providers starting with `GoogleCredentials` and `SpeechmaticsCredentials`.
- **Problem:**
  - `synapto-llm`, `stt-google`, `tts-google`, `task-memory`, and `synapto-plugin-google-chat` each implement isolated credential schemas and token signing logic.
  - If a service account is not provided in configuration, components fail instead of querying the GCP metadata server (`http://169.254.169.254`).
  - `Synapto` bundle signature does not support pluggable credentials providers.
  - `CoreConfig` contains vendor-specific fields (`google_service_account_credentials`, `gemini_api_key`), violating the Principle of Locality.
  - `synapto-interface` must remain strictly vendor-neutral without platform-specific target types.
- **Solution:**
  - Define standard protocol-neutral traits (`ProvideBearerToken<Target>`, `ProvideApiKey<Target>`, `ProvideBasicAuth<Target>`) and `CredentialsHandle` in `synapto-interface` with zero vendor-specific types.
  - Add `load_credentials_config` to `ConfigProvider` to provide scoped configuration resolution for credentials providers (e.g. `credentials.google.GoogleCredentials`).
  - Define vendor target descriptors (e.g. `GoogleCloudTarget`, `SpeechmaticsTarget`) strictly inside their respective integration crates.
  - Add the `CR` generic parameter to `Synapto`:
    ```rust
    Synapto::<
        (ConfigJson<DataDir>, DotEnv, Env),
        Storage,
        FilePromptProvider<DataDir>,
        (GoogleCredentials, SpeechmaticsCredentials),
    >::run::<(
        HostAudioInputPlugin,
        HostAudioOutputPlugin,
        GoogleChatPlugin,
        SpeechmaticsSttPlugin,
    )>()
    ```
  - Implement `GoogleCredentials` in `integrations/credentials-providers/google` with support for static JSON keys and ambient GCP Compute Engine metadata server ADC.
  - Implement `SpeechmaticsCredentials` in `integrations/credentials-providers/speechmatics` with support for scoped configuration and environment variables.

## 3. Current State Analysis
- **Scattered Data Models & Vendor Pollution in Core:**
  - `synapto-core/src/config.rs`: `CoreConfig` contains hardcoded `google_vertex_ai_location`, `google_project_id`, `gemini_api_key`, and `google_service_account_credentials`.
  - `synapto-core/src/google_credentials.rs`: Defines `GoogleServiceAccountCredentials(serde_json::Value)` inside core.
  - `synapto-plugins/google-chat/src/google_auth/oauth_client.rs`: Custom `GoogleServiceAccountCredentials` with RSA-SHA256 JWT generation.
  - `stt-google`, `tts-google`, `task-memory`: Duplicate definitions of Google service account wrappers.
- **Hard Requirement for Static Keys:**
  - `synapto-llm` requires `google_service_account_credentials` or `gemini_api_key`. If absent, initialization does not attempt metadata server resolution.
  - `synapto-plugin-google-chat` requires an explicit private key to sign JWT tokens.
  - Deployment scripts (`setup_gcp_project.sh`) generate static private key JSON files and upload them to the VM via SOPS-encrypted configuration files.
- **Rigid Composition Signature:**
  - `Synapto<C, S, PR>` only allows configuring `ConfigProvider`, `StorageConnection`, and `PromptProvider`. Credentials cannot be configured pluggably at the bundle level.
- **Flaws:**
  - **Violation of Principle of Locality:** `CoreConfig` holds cloud vendor-specific keys instead of scoping configuration to the respective integration provider.
  - **Security Risk (Static Credentials):** Key files stored on disk increase the attack surface and violate ISO 27001 recommendations for ambient identities.
  - **High Duplication (DRY Violation):** Multiple crates duplicate token exchange logic and token cache implementations.
  - **Inverted Responsibility:** Plugins handle low-level HTTP token exchanges and credential parsing directly rather than focusing strictly on their domain capabilities.

## 4. Structural Hierarchy
- **Mission:** Pluggable, Type-Safe, Vendor-Neutral Capability Archetypes for Authentication.
- **Goal:** Decouple plugins from specific credential storage, enable type-level credentials composition in bundles, and provide clean ambient identity resolution with scoped configuration.
- **Atomic Responsibilities:**
  - **`CredentialTarget` (Marker Trait in `synapto-interface`):** Domain marker trait bounding credential target descriptors to enforce Rule 24 compliance.
  - **`ProvideBearerToken<Target: CredentialTarget>` (Capability Trait in `synapto-interface`):** Protocol-neutral trait for resolving scoped bearer tokens.
  - **`ProvideApiKey<Target: CredentialTarget>` (Capability Trait in `synapto-interface`):** Protocol-neutral trait for resolving static or environment API keys.
  - **`ProvideBasicAuth<Target: CredentialTarget>` (Capability Trait in `synapto-interface`):** Protocol-neutral trait for basic HTTP credentials.
  - **`PluggableCredentialsProvider` (Factory Trait in `synapto-interface`):** Factory trait implemented by credentials provider types to register their capabilities into `CredentialsBuilder`.
  - **`CredentialsConfigResolver` / `ConfigProvider` (`synapto-core`):** Resolves scoped configuration for credentials providers under `credentials.<crate_name>.<provider_name>`.
  - **`CredentialsHandle` (Shared Registry in `synapto-interface`):** Type-erased container holding registered credential resolvers indexed by target `TypeId`, exposed via `PluginInitContext` and shared with LLM executors.
  - **`CredentialsTuple` (Composition in `synapto-core`):** Compile-time tuple trait for initializing credentials providers into `CredentialsHandle`.
  - **`GoogleCredentials` & `GoogleCloudTarget` (Integration in `integrations/credentials-providers/google`):** Pluggable provider and target descriptor implementing `ProvideBearerToken<GoogleCloudTarget>` via scoped configuration or ambient GCP metadata server ADC.
  - **`SpeechmaticsCredentials` & `SpeechmaticsTarget` (Integration in `integrations/credentials-providers/speechmatics`):** Pluggable provider and target descriptor implementing `ProvideApiKey<SpeechmaticsTarget>` via scoped configuration or environment variables.

## 5. Data Models

### 5.1 Vendor-Neutral Capability Archetype Traits (`synapto-interface`)
```rust
use async_trait::async_trait;
use synapto_interface::secrets::Secret;
use std::any::TypeId;
use std::collections::HashMap;
use std::sync::Arc;

/// Domain marker trait for credential target descriptors.
/// Enforces Rule 24 compliance and bounds credential registry entries.
pub trait CredentialTarget: Send + Sync + 'static {}

/// Capability trait for scoped bearer tokens (e.g. Google Cloud, AWS STS, Azure AD, OAuth2).
#[async_trait]
pub trait ProvideBearerToken<Target: CredentialTarget>: Send + Sync + 'static {
    async fn resolve_bearer_token(&self, target: &Target) -> Result<Secret<String>, String>;
}

/// Capability trait for API keys (e.g. Speechmatics, OpenAI, Anthropic, ElevenLabs).
#[async_trait]
pub trait ProvideApiKey<Target: CredentialTarget>: Send + Sync + 'static {
    async fn resolve_api_key(&self, target: &Target) -> Result<Secret<String>, String>;
}

/// Capability trait for basic authentication (username and secret password).
#[async_trait]
pub trait ProvideBasicAuth<Target: CredentialTarget>: Send + Sync + 'static {
    async fn resolve_basic_auth(&self, target: &Target) -> Result<(String, Secret<String>), String>;
}

#[async_trait]
trait ErasedBearerResolver: Send + Sync + 'static {
    async fn resolve_erased(&self, target: &(dyn std::any::Any + Send + Sync)) -> Result<Secret<String>, String>;
}

struct TypedBearerResolver<Target, P> {
    provider: Arc<P>,
    _marker: std::marker::PhantomData<Target>,
}

#[async_trait]
impl<Target: CredentialTarget, P: ProvideBearerToken<Target>> ErasedBearerResolver for TypedBearerResolver<Target, P> {
    async fn resolve_erased(&self, target: &(dyn std::any::Any + Send + Sync)) -> Result<Secret<String>, String> {
        let typed_target = target.downcast_ref::<Target>().ok_or_else(|| {
            format!("Invalid target type: expected {}", std::any::type_name::<Target>())
        })?;
        self.provider.resolve_bearer_token(typed_target).await
    }
}

#[async_trait]
trait ErasedApiKeyResolver: Send + Sync + 'static {
    async fn resolve_erased(&self, target: &(dyn std::any::Any + Send + Sync)) -> Result<Secret<String>, String>;
}

struct TypedApiKeyResolver<Target, P> {
    provider: Arc<P>,
    _marker: std::marker::PhantomData<Target>,
}

#[async_trait]
impl<Target: CredentialTarget, P: ProvideApiKey<Target>> ErasedApiKeyResolver for TypedApiKeyResolver<Target, P> {
    async fn resolve_erased(&self, target: &(dyn std::any::Any + Send + Sync)) -> Result<Secret<String>, String> {
        let typed_target = target.downcast_ref::<Target>().ok_or_else(|| {
            format!("Invalid target type: expected {}", std::any::type_name::<Target>())
        })?;
        self.provider.resolve_api_key(typed_target).await
    }
}

/// Builder used by credential providers to register their typed resolvers.
#[derive(Default)]
pub struct CredentialsBuilder {
    bearer_resolvers: HashMap<TypeId, Arc<dyn ErasedBearerResolver>>,
    api_key_resolvers: HashMap<TypeId, Arc<dyn ErasedApiKeyResolver>>,
}

impl CredentialsBuilder {
    pub fn register_bearer<Target: CredentialTarget, P: ProvideBearerToken<Target>>(
        &mut self,
        provider: Arc<P>,
    ) -> Result<(), String> {
        let type_id = TypeId::of::<Target>();
        if self.bearer_resolvers.contains_key(&type_id) {
            return Err(format!(
                "Credentials resolver already registered for target: {}",
                std::any::type_name::<Target>()
            ));
        }
        let resolver = TypedBearerResolver {
            provider,
            _marker: std::marker::PhantomData,
        };
        self.bearer_resolvers.insert(type_id, Arc::new(resolver));
        Ok(())
    }

    pub fn register_api_key<Target: CredentialTarget, P: ProvideApiKey<Target>>(
        &mut self,
        provider: Arc<P>,
    ) -> Result<(), String> {
        let type_id = TypeId::of::<Target>();
        if self.api_key_resolvers.contains_key(&type_id) {
            return Err(format!(
                "Credentials resolver already registered for target: {}",
                std::any::type_name::<Target>()
            ));
        }
        let resolver = TypedApiKeyResolver {
            provider,
            _marker: std::marker::PhantomData,
        };
        self.api_key_resolvers.insert(type_id, Arc::new(resolver));
        Ok(())
    }

    pub fn build(self) -> CredentialsHandle {
        CredentialsHandle {
            bearer_resolvers: Arc::new(self.bearer_resolvers),
            api_key_resolvers: Arc::new(self.api_key_resolvers),
        }
    }
}

/// Type-erased registry container for runtime credential resolution.
#[derive(Clone, Default)]
pub struct CredentialsHandle {
    bearer_resolvers: Arc<HashMap<TypeId, Arc<dyn ErasedBearerResolver>>>,
    api_key_resolvers: Arc<HashMap<TypeId, Arc<dyn ErasedApiKeyResolver>>>,
}

impl CredentialsHandle {
    /// Resolves a scoped bearer token against registered providers.
    pub async fn resolve_bearer_token<Target: CredentialTarget>(
        &self,
        target: &Target,
    ) -> Result<Secret<String>, String> {
        let type_id = TypeId::of::<Target>();
        if let Some(resolver) = self.bearer_resolvers.get(&type_id) {
            resolver.resolve_erased(target).await
        } else {
            Err(format!(
                "No registered credentials provider implements ProvideBearerToken<{}>",
                std::any::type_name::<Target>()
            ))
        }
    }

    /// Resolves an API key against registered providers.
    pub async fn resolve_api_key<Target: CredentialTarget>(
        &self,
        target: &Target,
    ) -> Result<Secret<String>, String> {
        let type_id = TypeId::of::<Target>();
        if let Some(resolver) = self.api_key_resolvers.get(&type_id) {
            resolver.resolve_erased(target).await
        } else {
            Err(format!(
                "No registered credentials provider implements ProvideApiKey<{}>",
                std::any::type_name::<Target>()
            ))
        }
    }
}

/// Factory trait for pluggable credential providers initialized by Synapto.
pub trait PluggableCredentialsProvider: Send + Sync + 'static {
    type Config: serde::de::DeserializeOwned + Default + Send + Sync;

    fn register_provider<C: synapto_interface::config::ConfigProvider>(
        config: Self::Config,
        builder: &mut CredentialsBuilder,
    ) -> Result<(), String>;
}
```

### 5.2 Scoped Configuration Resolution (`synapto-core`)
`ConfigProvider` is extended to support scoped credentials configuration:
```rust
pub trait ConfigProvider: Send + Sync + Sized + 'static {
    // ... existing methods ...

    /// Returns the credentials configuration for a specific provider.
    fn load_credentials_config(&self, crate_name: &str, provider_type_name: &str) -> serde_json::Value;

    /// Retrieves and deserializes the configuration for a specific credentials provider.
    fn get_credentials_config<T: serde::de::DeserializeOwned>(
        &self,
        crate_name: &str,
        provider_type_name: &str,
    ) -> T {
        let val = self.load_credentials_config(crate_name, provider_type_name);
        serde_json::from_value(val).unwrap_or_else(|e| {
            panic!(
                "Failed to parse config for credentials provider '{}::{}': {}",
                crate_name, provider_type_name, e
            )
        })
    }
}
```

### 5.3 Plugin Boundary Contract (`synapto-interface/src/plugin.rs`)
```rust
pub struct PluginInitContext<'a> {
    llm_executor: crate::llm::LlmExecutor,
    plugin_config: &'a serde_json::Value,
    storage: crate::storage::StorageHandle,
    plugin_namespace: &'a str,
    credentials: crate::credentials::CredentialsHandle,
}

impl<'a> PluginInitContext<'a> {
    pub fn credentials(&self) -> crate::credentials::CredentialsHandle {
        self.credentials.clone()
    }
}
```

### 5.4 Type-Safe Composition Signature (`synapto-core`)
```rust
pub trait CredentialsTuple: Send + Sync + 'static {
    fn build_handle<C: synapto_interface::config::ConfigProvider>(
        config_provider: &C,
    ) -> Result<synapto_interface::credentials::CredentialsHandle, String>;
}

impl CredentialsTuple for () {
    fn build_handle<C: synapto_interface::config::ConfigProvider>(
        _config_provider: &C,
    ) -> Result<synapto_interface::credentials::CredentialsHandle, String> {
        Ok(synapto_interface::credentials::CredentialsHandle::default())
    }
}

macro_rules! impl_credentials_tuple {
    ($($T:ident),+) => {
        impl<$($T: synapto_interface::credentials::PluggableCredentialsProvider),+> CredentialsTuple for ($($T,)+) {
            fn build_handle<C: synapto_interface::config::ConfigProvider>(
                config_provider: &C,
            ) -> Result<synapto_interface::credentials::CredentialsHandle, String> {
                let mut builder = synapto_interface::credentials::CredentialsBuilder::default();
                $(
                    let full_path = std::any::type_name::<$T>();
                    let crate_name = full_path.split("::").next().unwrap_or("").replace('-', "_");
                    let type_name = full_path.split("::").last().unwrap_or("");
                    let config: <$T as synapto_interface::credentials::PluggableCredentialsProvider>::Config =
                        config_provider.get_credentials_config(&crate_name, type_name);
                    $T::register_provider(config, &mut builder)?;
                )+
                Ok(builder.build())
            }
        }
    };
}

impl_credentials_tuple!(P1);
impl_credentials_tuple!(P1, P2);
impl_credentials_tuple!(P1, P2, P3);
impl_credentials_tuple!(P1, P2, P3, P4);

pub struct Synapto<
    C: config::ConfigProvider,
    S: storage::StorageConnection + storage::KeyValueStore + storage::RecordStore,
    PR: prompt_provider::CognitivePromptProvider = prompt_provider::EmptyPromptProvider,
    CR: CredentialsTuple = (),
> {
    // ...
    credentials: synapto_interface::credentials::CredentialsHandle,
}
```

### 5.5 Concrete Credentials Provider Implementations

#### 1. Google Credentials Provider (`integrations/credentials-providers/google`)
```rust
use async_trait::async_trait;
use serde::{Deserialize, Serialize};
use synapto_interface::credentials::{
    CredentialsBuilder, PluggableCredentialsProvider, ProvideBearerToken,
};
use synapto_interface::secrets::Secret;
use std::collections::{BTreeSet, HashMap};
use std::sync::Arc;
use tokio::sync::RwLock;

/// Target descriptor for Google Cloud authentication, defined in this integration crate.
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct GoogleCloudTarget {
    pub scopes: Vec<String>,
}

impl synapto_interface::credentials::CredentialTarget for GoogleCloudTarget {}

#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct GoogleCredentialsConfig {
    #[serde(default)]
    pub service_account_key: Option<Secret<String>>,
    #[serde(default)]
    pub project_id: Option<String>,
}

struct CachedToken {
    token: Secret<String>,
    expires_at: std::time::Instant,
}

pub struct GoogleCredentials {
    config: GoogleCredentialsConfig,
    token_cache: RwLock<HashMap<BTreeSet<String>, CachedToken>>,
    http_client: reqwest::Client,
}

#[async_trait]
impl ProvideBearerToken<GoogleCloudTarget> for GoogleCredentials {
    async fn resolve_bearer_token(
        &self,
        target: &GoogleCloudTarget,
    ) -> Result<Secret<String>, String> {
        let normalized_scopes: BTreeSet<String> = target.scopes.iter().cloned().collect();

        // 1. Check in-memory scope cache with 60-second buffer
        {
            let cache = self.token_cache.read().await;
            if let Some(entry) = cache.get(&normalized_scopes) {
                if entry.expires_at > std::time::Instant::now() + std::time::Duration::from_secs(60) {
                    return Ok(entry.token.clone());
                }
            }
        }

        // 2. Fetch fresh token (Static Key or Ambient GCP Metadata Server)
        let (token, ttl) = if let Some(ref key_secret) = self.config.service_account_key {
            let account = gcp_auth::CustomServiceAccount::from_json(key_secret.expose_secret())
                .map_err(|e| format!("Invalid Google Service Account JSON: {}", e))?;
            let scopes_ref: Vec<&str> = target.scopes.iter().map(|s| s.as_str()).collect();
            let token_resp = account.token(&scopes_ref).await
                .map_err(|e| format!("Failed to generate token from static key: {}", e))?;
            (Secret::new(token_resp.as_str().to_string()), std::time::Duration::from_secs(3600))
        } else {
            // Ambient resolution: query GCP Compute Engine Metadata Server
            let mut url = reqwest::Url::parse(
                "http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token",
            ).map_err(|e| e.to_string())?;

            if !target.scopes.is_empty() {
                url.query_pairs_mut().append_pair("scopes", &target.scopes.join(","));
            }

            let response = self.http_client
                .get(url)
                .header("Metadata-Flavor", "Google")
                .timeout(std::time::Duration::from_millis(1500))
                .send()
                .await
                .map_err(|e| format!("GCP Metadata Server unreachable: {}", e))?;

            if !response.status().is_success() {
                return Err(format!("GCP Metadata Server returned HTTP {}", response.status()));
            }

            let json: serde_json::Value = response.json().await
                .map_err(|e| format!("Failed to parse metadata token response: {}", e))?;
            let access_token = json["access_token"].as_str()
                .ok_or_else(|| "Missing access_token in metadata response".to_string())?;
            let expires_in_secs = json["expires_in"].as_u64().unwrap_or(3600);

            (Secret::new(access_token.to_string()), std::time::Duration::from_secs(expires_in_secs))
        };

        // 3. Save to cache
        let mut cache = self.token_cache.write().await;
        cache.insert(
            normalized_scopes,
            CachedToken {
                token: token.clone(),
                expires_at: std::time::Instant::now() + ttl,
            },
        );

        Ok(token)
    }
}

impl PluggableCredentialsProvider for GoogleCredentials {
    type Config = GoogleCredentialsConfig;

    fn register_provider<C: synapto_interface::config::ConfigProvider>(
        config: Self::Config,
        builder: &mut CredentialsBuilder,
    ) -> Result<(), String> {
        let provider = Arc::new(Self {
            config,
            token_cache: RwLock::new(HashMap::new()),
            http_client: reqwest::Client::new(),
        });

        builder.register_bearer::<GoogleCloudTarget, Self>(provider)?;
        Ok(())
    }
}
```

#### 2. Speechmatics Credentials Provider (`integrations/credentials-providers/speechmatics`)
```rust
use async_trait::async_trait;
use serde::{Deserialize, Serialize};
use synapto_interface::credentials::{
    CredentialsBuilder, PluggableCredentialsProvider, ProvideApiKey,
};
use synapto_interface::secrets::Secret;
use std::sync::Arc;

/// Target descriptor for Speechmatics authentication, defined in this integration crate.
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct SpeechmaticsTarget;

impl synapto_interface::credentials::CredentialTarget for SpeechmaticsTarget {}

#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct SpeechmaticsCredentialsConfig {
    #[serde(default)]
    pub api_key: Option<Secret<String>>,
}

pub struct SpeechmaticsCredentials {
    config: SpeechmaticsCredentialsConfig,
}

#[async_trait]
impl ProvideApiKey<SpeechmaticsTarget> for SpeechmaticsCredentials {
    async fn resolve_api_key(
        &self,
        _target: &SpeechmaticsTarget,
    ) -> Result<Secret<String>, String> {
        if let Some(ref key) = self.config.api_key {
            if !key.expose_secret().is_empty() {
                return Ok(key.clone());
            }
        }
        if let Ok(env_key) = std::env::var("SPEECHMATICS_API_KEY") {
            if !env_key.is_empty() {
                return Ok(Secret::new(env_key));
            }
        }
        Err("Speechmatics API key is not configured. Set 'credentials.speechmatics.api_key' in configuration or SPEECHMATICS_API_KEY environment variable.".to_string())
    }
}

impl PluggableCredentialsProvider for SpeechmaticsCredentials {
    type Config = SpeechmaticsCredentialsConfig;

    fn register_provider<C: synapto_interface::config::ConfigProvider>(
        config: Self::Config,
        builder: &mut CredentialsBuilder,
    ) -> Result<(), String> {
        let provider = Arc::new(Self { config });
        builder.register_api_key::<SpeechmaticsTarget, Self>(provider)?;
        Ok(())
    }
}
```

### 5.6 Concrete Plugin Implementations

#### Example A: Google Chat Plugin
```rust
use synapto_interface::plugin::{Plugin, PluginInitContext, PluginRegistry};
use synapto_interface::credentials::CredentialsHandle;
use credentials_provider_google::GoogleCloudTarget;
use std::sync::Arc;

pub struct GoogleChatPlugin {
    credentials: CredentialsHandle,
    spaces: Vec<String>,
}

#[async_trait::async_trait]
impl Plugin for GoogleChatPlugin {
    async fn create(context: &PluginInitContext<'_>) -> Result<Self, String> {
        let config: GoogleChatConfig = context.optional_config()?.unwrap_or_default();
        Ok(Self {
            credentials: context.credentials(),
            spaces: config.spaces,
        })
    }

    fn register<R: PluginRegistry + ?Sized>(self: Arc<Self>, registry: &mut R) {
        registry.register_chat(self);
    }
}

impl GoogleChatPlugin {
    async fn send_chat_message(&self, space: &str, text: &str) -> Result<(), String> {
        let target = GoogleCloudTarget {
            scopes: vec!["https://www.googleapis.com/auth/chat.bot".to_string()],
        };
        let token = self.credentials.resolve_bearer_token(&target).await?;

        let client = reqwest::Client::new();
        client.post(format!("https://chat.googleapis.com/v1/{}/messages", space))
            .bearer_auth(token.expose_secret())
            .json(&serde_json::json!({ "text": text }))
            .send()
            .await
            .map_err(|e| e.to_string())?;

        Ok(())
    }
}
```

#### Example B: Speechmatics STT Plugin
```rust
use synapto_interface::plugin::{Plugin, PluginInitContext, PluginRegistry};
use synapto_interface::credentials::CredentialsHandle;
use credentials_provider_speechmatics::SpeechmaticsTarget;
use std::sync::Arc;

pub struct SpeechmaticsSttPlugin {
    credentials: CredentialsHandle,
}

#[async_trait::async_trait]
impl Plugin for SpeechmaticsSttPlugin {
    async fn create(context: &PluginInitContext<'_>) -> Result<Self, String> {
        Ok(Self {
            credentials: context.credentials(),
        })
    }

    fn register<R: PluginRegistry + ?Sized>(self: Arc<Self>, registry: &mut R) {
        registry.register_stt(self);
    }
}

impl SpeechmaticsSttPlugin {
    async fn run_recognition(&self) -> Result<(), String> {
        let api_key = self.credentials.resolve_api_key(&SpeechmaticsTarget).await?;
        // Connect WebSocket / REST client using api_key in authorization header...
        Ok(())
    }
}
```

### 5.7 Scoped JSON Configuration Contract Examples

#### Ambient Identity Mode (Keyless GCP VM)
```json
{
  "audience": "reasonably intelligent human",
  "plugins": {
    "google_chat": {
      "GoogleChatPlugin": {
        "spaces": ["spaces/AAAAAAAAAAA"]
      }
    }
  }
}
```

#### Multi-Provider Configuration with Scoped Keys
```json
{
  "audience": "reasonably intelligent human",
  "credentials": {
    "google": {
      "GoogleCredentials": {
        "project_id": "my-gcp-project",
        "service_account_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
      }
    },
    "speechmatics": {
      "SpeechmaticsCredentials": {
        "api_key": "sm_api_key_abcdef123456"
      }
    }
  }
}
```

## 6. Component Architecture
- **Initialization Flow:**
  1. The composition bundle initializes `Synapto::<C, S, PR, CR>::run::<P>()`.
  2. `CR::build_handle(&config_provider)` extracts each provider's scoped configuration (`config_provider.get_credentials_config::<P::Config>(&crate_name, type_name)`) and registers resolvers into `CredentialsBuilder`.
  3. `GoogleCredentials` checks if `service_account_key` is configured. If present, it initializes static JWT signing. If absent, it sets up ambient GCP metadata resolution:
     - Endpoint: `http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token`
     - Header: `Metadata-Flavor: Google`
     - Probe Timeout: 1.5 seconds (executed lazily during first asynchronous token resolution).
  4. `CredentialsHandle` is stored in `Synapto`, passed into `synapto-llm`, and injected into `PluginInitContext` for all registered plugins.
- **Runtime Execution Flow:**
  1. When a plugin or `synapto-llm` sends an API request, it requests a token via `context.credentials().resolve_bearer_token(&GoogleCloudTarget { scopes }).await`.
  2. `GoogleCredentials` checks its compound scope token cache (`RwLock<HashMap<BTreeSet<String>, CachedToken>>`). If a valid token exists for the requested scopes (with a 60-second expiration buffer), it returns the cached token.
  3. If expired or missing, the provider queries the GCP metadata server (or signs a new JWT via static private key), updates the scope cache, and returns the token.
  4. The caller attaches the resolved credential to the HTTP authorization header.

## 7. Architectural Diagrams

### Token Resolution Topography
```mermaid
flowchart TD
    subgraph Bundle [Composition Bundle in main.rs]
        Init[Synapto::run] --> BuildCreds["CR::build_handle (GoogleCredentials, SpeechmaticsCredentials)"]
    end

    subgraph ConfigResolution [Scoped Config Resolution]
        BuildCreds --> LoadGoogle["load_credentials_config(credentials.google.GoogleCredentials)"]
        BuildCreds --> LoadSpeechmatics["load_credentials_config(credentials.speechmatics.SpeechmaticsCredentials)"]
    end

    subgraph GoogleProvider [integrations/credentials-providers/google]
        LoadGoogle --> CheckConfig{Static Key in Scoped Config?}
        CheckConfig -- Yes --> StaticEngine[Static JWT Signer]
        CheckConfig -- No --> AmbientEngine[GCP Metadata Client]
        StaticEngine --> GoogleCache["Scope Token Cache: RwLock HashMap"]
        AmbientEngine --> GoogleCache
    end

    subgraph SpeechmaticsProvider [integrations/credentials-providers/speechmatics]
        LoadSpeechmatics --> SpeechmaticsEngine[Config/Env API Key Provider]
    end

    subgraph RuntimeConsumers [Runtime Consumers]
        LLM[synapto-llm / Vertex AI]
        Chat[synapto-plugin-google-chat]
        STT[synapto-plugin-stt-google]
        SmSTT[synapto-plugin-speechmatics]
    end

    GoogleCache --> Handle[CredentialsHandle in PluginInitContext]
    SpeechmaticsEngine --> Handle

    Handle -->|resolve_bearer_token GoogleCloudTarget| LLM
    Handle -->|resolve_bearer_token GoogleCloudTarget| Chat
    Handle -->|resolve_bearer_token GoogleCloudTarget| STT
    Handle -->|resolve_api_key SpeechmaticsTarget| SmSTT

    AmbientEngine -.->|Lazy HTTP GET Metadata-Flavor: Google| MetadataServer[GCP Metadata Server 169.254.169.254]
```

## 8. Robustness & Pragmatic Security Trade-offs
- **ISO/IEC 27001 Compliance (Control A.8.24 - Key Management):** Ambient VM identity eliminates static private keys on persistent disks and in deployment repositories.
- **Strict Principle of Locality:** `CoreConfig` is stripped of vendor fields; credentials configuration is scoped exclusively under `credentials.<provider_crate>.<provider_struct>`.
- **Zero Vendor Pollution in Interface:** `synapto-interface` contains only protocol archetypes (`ProvideBearerToken`, `ProvideApiKey`, `ProvideBasicAuth`).
- **Compound Scope Token Caching:** The token cache is keyed by sorted scope sets (`BTreeSet<String>`), preventing token cross-talk and permission mismatches between distinct plugins.
- **Lazy Metadata Resolution:** Metadata network calls occur asynchronously during token resolution, preventing blocking operations during synchronous application boot.
- **Graceful Error Handling:** If the GCP metadata server is unreachable (for example, in a local non-GCP development environment without static keys), `resolve_bearer_token()` returns an informative `Err(String)` explaining that neither static credentials nor ambient GCP metadata are available.
- **Accepted Risks (MVP):**
  - **Retry and Backoff Logic:** Network failures to the GCP metadata server will fail immediately without exponential backoff or jitter.
  - **Lock Contention:** Multi-threaded token resolution uses standard `tokio::sync::RwLock` without lock-free atomic pointer swaps.
  - **Dynamic Hot-Reloading:** Credential providers initialize during application startup. Modifying credentials at runtime requires an application restart.

## 9. Systemic Consequences

### 9.1 Audited Codebase Dependencies
The following files were identified via mechanical search and depend on credentials, `CoreConfig`, or `PluginInitContext`:
- `synapto-ai/synapto/crates/interface/src/plugin.rs`: `PluginInitContext` definition.
- `synapto-ai/synapto/crates/core/src/google_credentials.rs`: Deprecated static credentials struct removed from core.
- `synapto-ai/synapto/crates/core/src/config.rs`: Removed vendor fields (`google_service_account_credentials`, `gemini_api_key`) from `CoreConfig`.
- `synapto-ai/synapto/crates/core/src/config/provider.rs`: Added `load_credentials_config` and `get_credentials_config`.
- `synapto-ai/synapto/crates/core/src/lib.rs`: `Synapto` composition definition and plugin runner.
- `synapto-ai/synapto/crates/llm/src/lib.rs`: `ConcreteLlmExecutor` Vertex AI authentication.
- `synapto-ai/synapto/integrations/plugins/stt-google/src/lib.rs`: Google STT client initialization.
- `synapto-ai/synapto/integrations/plugins/tts-google/src/lib.rs`: Google TTS client initialization.
- `synapto-ai/synapto/integrations/plugins/task-memory/src/lib.rs`: Google Vertex AI / Gemini integration.
- `allusio-ai/synapto-plugins/google-chat/src/lib.rs`: Google Chat plugin configuration.
- `allusio-ai/synapto-plugins/google-chat/src/google_auth/oauth_client.rs`: Custom JWT token exchange.
- `allusio-ai/synapto-plugins/google-chat/src/read.rs`: Google Chat REST & Pub/Sub requests.
- `allusio-ai/synapto-plugins/google-chat/src/reactions.rs`: Google Chat reaction REST requests.
- `allusio-ai/org-assistant/src/main.rs`: Bundle composition root.
- `allusio-ai/org-assistant/scripts/setup_gcp_project.sh`: Deployment script.
- `allusio-ai/org-assistant/scripts/prepare_gcp_vm.sh`: VM provisioning script.

### 9.2 Breaking Changes
- `CoreConfig` no longer contains `google_service_account_credentials` or `gemini_api_key`. Credentials configuration moves under `credentials.<crate_name>.<provider_name>`.
- `PluginInitContext` carries `credentials() -> CredentialsHandle`.
- `Synapto` accepts the fourth generic parameter `CR: CredentialsTuple = ()`.

## 10. Definition of Done (DoD)
- [ ] 1. Create `synapto_interface::credentials` module with `CredentialTarget`, `ProvideBearerToken<Target>`, `ProvideApiKey<Target>`, `ProvideBasicAuth<Target>`, `PluggableCredentialsProvider`, `CredentialsBuilder`, and `CredentialsHandle`.
- [ ] 2. Update `synapto_interface::plugin::PluginInitContext` with `credentials()` accessor.
- [ ] 3. Update `synapto-core/src/config/provider.rs` with `load_credentials_config` and `get_credentials_config`.
- [ ] 4. Remove vendor fields from `synapto-core/src/config.rs` (`CoreConfig`) and remove `google_credentials.rs`.
- [ ] 5. Add `CR: CredentialsTuple = ()` generic parameter and tuple macro implementations to `Synapto` in `synapto-core`.
- [ ] 6. Create `integrations/credentials-providers/google` crate implementing `GoogleCredentials` and `GoogleCloudTarget`.
- [ ] 7. Update `synapto-llm` to resolve Vertex AI authorization tokens via `CredentialsHandle`.
- [ ] 8. Migrate `synapto-plugin-google-chat` to resolve tokens via `context.credentials()`.
- [ ] 9. Update `allusio-ai/org-assistant/src/main.rs` bundle composition to include `GoogleCredentials`.
- [ ] 10. Update `scripts/prepare_gcp_vm.sh` and VM provisioning documentation.
- [ ] 11. Verify `cargo check` and `cargo test` pass across workspace crates.
