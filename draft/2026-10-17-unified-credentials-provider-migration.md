# [MVP] [DRAFT] RFC: Unified Credentials Provider Migration and Environment Variable Elimination

> **Classification:** `Guideline` (Rule 27 Unified Credentials Provider & Zero Environment Variables Mandate)
> **Draft Status:** Phase 2 — Awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-unified-credentials-provider-migration.md`.

## 1. User-Centric Motivation & Goal

Rule 27 (`ARCHITECTURE.md:98-100`) and the Zero Environment Variables Mandate (`AGENTS.md`) require that every plugin and provider obtain authentication credentials exclusively through `context.credentials()` (`CredentialsHandle`), and forbid reading secrets or operational parameters from ambient environment variables (`std::env::var`, `Env`, `DotEnv`).

Currently, `org-assistant/integrations/google-mail` defines `client_id`, `client_secret`, and `refresh_token` directly in `GoogleMailConfig` and implements its own custom `GoogleMailAuth` OAuth client without ever calling `context.credentials()`. Similarly, `synapto-llm-google` embeds `gemini_api_key` in `GoogleLlmConfig`, `synapto-plugin-mumble` embeds `password: Option<String>` in `MumbleConfig`, `synapto-plugin-mcp` embeds `headers` and `env` in `McpPluginConfig`, `synapto-credentials-google` includes a `subject` field for Domain-Wide Delegation (prohibited by `org-assistant/AGENTS.md` Rule 3.3), 7 crates read `std::env::var` at runtime, and `org-assistant/assistant/src/main.rs` mounts `DotEnv` and `Env` config providers in production.

This RFC migrates all plugins and LLM providers to resolve credentials exclusively through `CredentialsHandle`, removes the DWD `subject` field from `synapto-credentials-google`, removes all `std::env::var` lookups, and removes `DotEnv`/`Env` from `org-assistant/assistant`.

---

## 2. Executive Summary

- **Context:** `org-assistant/integrations/{google-mail, google-chat, google-meet}`, `org-assistant/assistant`, `synapto/integrations/credentials-providers/{google, elevenlabs, speechmatics}`, `synapto/integrations/llm-providers/google`, `synapto/integrations/plugins/{mumble, mcp, skills}`, and `synapto-plugins/speaker-recognizer`.
- **Motivation:** Enforce a Single Source of Truth for authentication credentials (Rule 27), eliminate Domain-Wide Delegation support (`org-assistant/AGENTS.md` Rule 3.3), and eliminate ambient environment variable fallbacks.
- **Overarching Goals:** 100% of secrets live under the `"credentials"` configuration domain and are resolved via `CredentialsHandle` using typed `CredentialTarget` descriptors, with zero `std::env::var` secret/parameter reads.
- **Purpose:** Migrate `google-mail`, `llm-providers/google`, `mumble`, and `mcp` to `CredentialsHandle`, remove `subject` from `GoogleCredentials`, delete `std::env::var` calls across 7 crates, and remove `DotEnv`/`Env` from `org-assistant/assistant`.
- **Problem:**
  1. **`org-assistant/integrations/google-mail/src/lib.rs:12-96, 105-116`:** `GoogleMailConfig` embeds `client_id`, `client_secret`, and `refresh_token`, and `GoogleMailAuth` implements custom OAuth token refresh bypassing `context.credentials()`.
  2. **`synapto/integrations/llm-providers/google/src/lib.rs:14-18, 76-90`:** `GoogleLlmConfig` embeds `pub gemini_api_key: Option<Secret<String>>` instead of resolving an API key target from `CredentialsHandle`.
  3. **`synapto/integrations/plugins/mumble/src/lib.rs:42-43, 260-263`:** `MumbleConfig` embeds plaintext `pub password: Option<String>` instead of resolving via `CredentialsHandle`.
  4. **`synapto/integrations/plugins/mcp/src/lib.rs:33-40, 189-196, 221, 445-452`:** `McpPluginConfig` embeds raw `env` and `headers` maps in `"plugins"` config.
  5. **`synapto/integrations/credentials-providers/google/src/lib.rs:18, 29-32, 194`:** Defines `pub subject: Option<String>` for Workspace Domain-Wide Delegation (prohibited by `org-assistant/AGENTS.md` Rule 3.3).
  6. **Ambient `std::env::var` Lookups Across 7 Crates:**
     - `synapto/integrations/credentials-providers/elevenlabs/src/lib.rs:40-49` (`ELEVENLABS_API_KEY`, `ELEVEN_API_KEY`)
     - `synapto/integrations/credentials-providers/speechmatics/src/lib.rs:43-47` (`SPEECHMATICS_API_KEY`)
     - `synapto/integrations/credentials-providers/google/src/lib.rs:179-183, 231-232` (`GOOGLE_API_BASE_URL`, `GOOGLE_METADATA_SERVER_URL`)
     - `org-assistant/integrations/google-chat/src/lib.rs:6-12` (`GOOGLE_API_BASE_URL`)
     - `org-assistant/integrations/google-meet/src/lib.rs:136, 201` (`PLAYWRIGHT_FIREFOX_EXECUTABLE_PATH`, `PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH`)
     - `synapto/integrations/plugins/skills/src/{discovery.rs:31-35, plugin.rs:28-30}` (`CLAUDE_CONFIG_DIR`, `std::env::current_dir()`)
     - `synapto-plugins/speaker-recognizer/src/lib.rs:121-123, 257-334` (16 clustering env vars under `feature = "research"`)
  7. **`org-assistant/assistant/src/main.rs:3, 159`:** Wires `(ConfigJson<CurrentDir>, DotEnv, Env)` instead of `(ConfigJson<CurrentDir>,)`.
- **Solution:**
  1. Add `GoogleUserOAuthTarget` (`ProvideBearerToken<GoogleUserOAuthTarget>`) and `GoogleGeminiApiKeyTarget` (`ProvideApiKey<GoogleGeminiApiKeyTarget>`) to `synapto-credentials-google`, remove `subject` from `GoogleCloudTarget`, and refactor `google-mail` and `synapto-llm-google` to resolve tokens/keys exclusively via `CredentialsHandle`.
  2. Remove `std::env::var` calls from `elevenlabs`, `speechmatics`, `google`, `google-chat`, `google-meet`, `skills`, and `speaker-recognizer`, moving non-secret URLs/paths into explicit plugin/provider config fields.
  3. Change `org-assistant/assistant/src/main.rs:159` to `.configs::<(ConfigJson<CurrentDir>,)>()`.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`org-assistant/integrations/google-mail/src/lib.rs:12-96, 105-116`**:
   `GoogleMailConfig` holds `client_id`, `client_secret`, `refresh_token`; `GoogleMailAuth` caches and refreshes OAuth tokens locally.
2. **`synapto/integrations/llm-providers/google/src/lib.rs:14-18, 76-90`**:
   `GoogleLlmConfig` holds `gemini_api_key: Option<Secret<String>>`.
3. **`synapto/integrations/credentials-providers/google/src/lib.rs:18, 29-32, 179-183, 194, 231-232`**:
   `GoogleCloudTarget` has `pub subject: Option<String>`; `resolve_bearer_token` reads `GOOGLE_API_BASE_URL` and `GOOGLE_METADATA_SERVER_URL` from `std::env::var`.
4. **`synapto/integrations/credentials-providers/elevenlabs/src/lib.rs:40-49`** & **`speechmatics/src/lib.rs:43-47`**:
   Read `ELEVENLABS_API_KEY`, `ELEVEN_API_KEY`, and `SPEECHMATICS_API_KEY` from `std::env::var`.
5. **`org-assistant/integrations/google-chat/src/lib.rs:6-12`**, **`google-meet/src/lib.rs:136, 201`**, **`skills/src/discovery.rs:31-35`**, **`speaker-recognizer/src/lib.rs:257-334`**:
   Read operational URLs, executable paths, directories, and clustering thresholds from `std::env::var`.
6. **`org-assistant/assistant/src/main.rs:3, 159`**:
   `use synapto::config::{ConfigJson, CurrentDir, DotEnv, Env};` and `.configs::<(ConfigJson<CurrentDir>, DotEnv, Env)>()`.

---

## 4. Structural Hierarchy

- **Mission:** Single Source of Truth for Credentials and Zero Ambient Environment Variable Fallbacks.
- **Goal:** Ensure all authentication flows through `CredentialsHandle` and no operational credential or parameter is read from environment variables.
- **Atomic Responsibilities:**
  - **`GoogleCredentials` (`synapto-credentials-google`):** Resolves `GoogleCloudTarget` (service account / metadata server ADC without DWD `subject`), `GoogleUserOAuthTarget` (user-scoped OAuth refresh token for `google-mail`), and `GoogleGeminiApiKeyTarget` (API key for `synapto-llm-google`).
  - **`GoogleMailPlugin` (`org-assistant/integrations/google-mail`):** Resolves bearer tokens via `self.credentials.resolve_bearer_token(&GoogleUserOAuthTarget).await`.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 27 (Unified Credentials Provider Mandate):** Plugins and providers obtain credentials exclusively via `CredentialsHandle`.
- **Zero Secrets in Command-Line Arguments and Environment Variables Mandate (`AGENTS.md`):** Never read credentials or operational configuration from ambient environment variables.
- **`org-assistant/AGENTS.md` Rule 3.3 (Zero Domain-Wide Delegation):** Workspace DWD is prohibited; mailbox access uses strictly user-scoped OAuth 2.0.

### 5.2 Credential Targets in `synapto-credentials-google`
```rust
// synapto/integrations/credentials-providers/google/src/lib.rs
#[derive(Debug, Clone, Default)]
pub struct GoogleCloudTarget;
impl CredentialTarget for GoogleCloudTarget {}

#[derive(Debug, Clone, Default)]
pub struct GoogleUserOAuthTarget;
impl CredentialTarget for GoogleUserOAuthTarget {}

#[derive(Debug, Clone, Default)]
pub struct GoogleGeminiApiKeyTarget;
impl CredentialTarget for GoogleGeminiApiKeyTarget {}

#[derive(Debug, Clone, Deserialize)]
pub struct GoogleUserOAuthCredentials {
    pub client_id: String,
    pub client_secret: Secret<String>,
    pub refresh_token: Secret<String>,
    pub token_uri: String,
}
```

### 5.3 JSON Contract Examples

#### Scoped Credentials Configuration (`config.json` excerpt)
```json
{
  "credentials": {
    "synapto_credentials_google": {
      "GoogleCredentials": {
        "user_oauth": {
          "client_id": "123456.apps.googleusercontent.com",
          "client_secret": "GOCSPX-...",
          "refresh_token": "1//0g...",
          "token_uri": "https://oauth2.googleapis.com/token"
        }
      }
    }
  }
}
```

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For handling `Env` and `DotEnv` `ConfigProvider` structs in `synapto/crates/core/src/config/{env.rs, dotenv.rs}`:

**[ Option A: Remove `DotEnv` and `Env` from `org-assistant/assistant` and Mark Them Test/Dev-Only in `synapto` ]**
- **Action:** Remove `DotEnv` and `Env` from `org-assistant/assistant/src/main.rs:159` (using strictly `ConfigJson<CurrentDir>`), and gate `Env` / `DotEnv` in `synapto` behind explicit documentation or test-only usage while removing all direct `std::env::var` calls from plugins and credentials providers.
- **Pros:** Enforces the Zero Environment Variables Mandate in `org-assistant` and across all plugins/providers without breaking `ARCHITECTURE.md` Section 7's description of `ConfigProvider` implementations in `synapto`.
- **Cons:** `Env` and `DotEnv` still exist as types in `synapto::config`.

**[ Option B: Delete `Env` and `DotEnv` `ConfigProvider` Implementations Entirely from `synapto` ]**
- **Action:** Delete `synapto/crates/core/src/config/env.rs` and `dotenv.rs` from `synapto` and remove all `SYNAPTO__*` environment variable config loading from the framework.
- **Pros:** Makes it impossible for any bundle to load configuration or secrets from environment variables.
- **Cons:** Removes `SYNAPTO__*` env config support described in `ARCHITECTURE.md` Rule 32 and Section 7 (`L118, 178`).

**My Recommendation:** I recommend **Option B** if the Zero Environment Variables Mandate applies universally across all `synapto` bundles, or **Option A** if `SYNAPTO__*` in `ARCHITECTURE.md` Rule 32 is still intended for local non-secret dev overrides outside `org-assistant`.
Please select Option A or Option B for this RFC.
