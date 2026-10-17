# [MVP] [DRAFT] RFC: Explicit Configuration Schemas and Orthogonal Configuration Namespaces

> **Classification:** `Guideline` (Rule 28 Explicit Configuration & Rule 32 Orthogonal Configuration Namespaces)
> **Draft Status:** Phase 2 — Awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-explicit-configuration-and-orthogonal-namespaces.md`.

## 1. User-Centric Motivation & Goal

Rule 28 (`ARCHITECTURE.md:101-103`) mandates that all configuration schemas be 100% explicit: plugins and core components must never use `#[serde(default)]`, `optional_config()`, or implicit runtime fallback values (`.unwrap_or("cs")`, hardcoded models, or hardcoded cloud regions), and every configuration struct must enforce `#[serde(deny_unknown_fields)]` (`ARCHITECTURE.md:178`). Rule 32 (`ARCHITECTURE.md:114-119, 179-185`) mandates that top-level configuration namespaces mirror the system's architectural roles 1:1.

Currently, `PluggableCredentialsProvider::Config` (`crates/interface/src/credentials.rs:260`) requires `+ Default`, forcing credentials providers to derive `Default` and `#[serde(default)]`. Over 15 crates across `crates/core`, `integrations/*`, `synapto-plugins/*`, and `org-assistant/integrations/*` use `#[serde(default)]`, `optional_config()`, or `.unwrap_or(...)` runtime fallbacks, omit `#[serde(deny_unknown_fields)]`, and `ConfigJson::load_core_config` (`crates/core/src/config/json.rs:29-39`) co-mingles root engine fields with the subsystem namespaces.

This RFC removes all `#[serde(default)]` attributes, `optional_config()`, `+ Default` trait bounds, and runtime configuration fallbacks across the entire workspace, adds `#[serde(deny_unknown_fields)]` to every configuration struct, and isolates root configuration namespaces per Rule 32.

---

## 2. Executive Summary

- **Context:** Configuration traits and structs across `synapto`, `synapto-plugins`, and `org-assistant`.
- **Motivation:** Eliminate silent configuration defaults (such as `stt-google` V2 silently defaulting `google_project_id` to `""` and language codes silently defaulting to Czech `"cs"` / `"cs-CZ"`), reject unknown/misspelled configuration keys at startup, and enforce orthogonal top-level configuration domains.
- **Overarching Goals:** 100% compliance with Rule 28, Rule 29, and Rule 32 across all crates.
- **Purpose:** Strip `#[serde(default)]` and runtime fallbacks, remove `PluginInitContext::optional_config` and `PluggableCredentialsProvider::Config: Default`, add `#[serde(deny_unknown_fields)]` everywhere, and partition root `config.json` domains cleanly.
- **Problem (Complete 24-Location Inventory):**
  1. `synapto/crates/interface/src/credentials.rs:260`: `type Config: serde::de::DeserializeOwned + Default + Send + Sync;`.
  2. `synapto/crates/interface/src/llm.rs:39-43`: `#[serde(default)] pub reasoning_effort: ReasoningEffort` on `ModelConfig`.
  3. `synapto/crates/interface/src/plugin.rs:65-75, 157-159` & `README.md:213-220`: `PluginInitContext::optional_config` and doc comments instructing developers to use `#[serde(default)]`.
  4. `synapto/crates/core/src/config.rs:16-44, 64-108` & `config/json.rs:29-39`: `#[serde(default)]` across `InitialRunConfig`, `CognitiveConfig`, and `Config`; root-level keys co-mingled with `cognitive`, `storage`, `credentials`, `decision`, `llm`, `plugins`.
  5. `synapto/crates/core/src/{cognitive.rs:128-129, cognitive/direct.rs:644-645, cognitive/side.rs:460-461}`: `serde_json::from_value(config.prompt.clone()).unwrap_or_default()` silently masks invalid prompt configs.
  6. `synapto/integrations/credentials-providers/{elevenlabs/src/lib.rs:16-20, google/src/lib.rs:37-53, 76-93, speechmatics/src/lib.rs:16-20, typesafe/src/lib.rs:17-28}`: `#[derive(Default)]` and `#[serde(default)]`.
  7. `synapto/integrations/datadir-providers/local/src/lib.rs:12-14`: `dirs::data_local_dir().unwrap_or_else(|| PathBuf::from("."))`.
  8. `synapto/integrations/llm-providers/google/src/lib.rs:128-130`: Hardcodes `.with_temperature(0.0).with_top_p(0.95)`.
  9. `synapto/integrations/plugins/behavioral-memory/src/plugin.rs:17-26`: `#[serde(default = "default_decision_preflight_threshold")]`.
  10. `synapto/integrations/plugins/semantic-memory/src/plugin.rs:23-32, 157-173`: `#[serde(default = "default_decision_preflight_threshold")]`; reuses `config.insight` for `activity` and `consolidation` LLM clients.
  11. `synapto/integrations/plugins/task-memory/src/tasks.rs:385, 387`: Hardcodes `0.75` decision threshold.
  12. `synapto/integrations/plugins/mcp/src/lib.rs:12-41, 179`: `#[derive(Default)]`, `#[serde(default)]` on 5 fields, `context.optional_config()?`.
  13. `synapto/integrations/plugins/mumble/src/lib.rs:29-44`: `#[derive(Default)]`, `#[serde(default = "default_host")]`, `#[serde(default = "default_port")]`.
  14. `synapto/integrations/plugins/skills/src/{config.rs:4-35, plugin.rs:27, context.rs:180}`: `#[serde(default)]` on all fields, `optional_config()?.unwrap_or_default()`, hardcoded `none_prob > 0.6`.
  15. `synapto/integrations/plugins/linux-host-audio/src/lib.rs:32, 52-54, 96, 116-118`: Const generic `DEFAULT_TARGET = ""` fallback.
  16. `synapto/integrations/plugins/stt-elevenlabs/src/lib.rs:99-102`: `unwrap_or("cs")` and hardcoded `model_id`, `commit_strategy`, `vad_threshold`.
  17. `synapto/integrations/plugins/stt-google/src/lib.rs:38-52, 134, 142, 386, 402`: `#[derive(Default)]`, `#[serde(default)]` on `version` and `google_project_id`, `unwrap_or_else(|| "cs-CZ")`, hardcoded `model: "latest_long"` and `location = "eu"`.
  18. `synapto/integrations/plugins/stt-speechmatics/src/lib.rs:123-125, 155-162`: `unwrap_or("cs")`, hardcoded `"eu2"`, `operating_point`, `max_delay`, `end_of_utterance_silence_trigger`.
  19. `synapto/integrations/plugins/tts-elevenlabs/src/lib.rs:81-84, 101-102`: `unwrap_or_else(|| "eleven_multilingual_v2")` and `unwrap_or_else(|| "cs")`.
  20. `synapto/integrations/plugins/tts-google/src/lib.rs:14-25, 89, 158-159`: `#[derive(Default)]`, `#[serde(default)]` on `relax_safety_filters`, hardcoded `<prosody rate="120%">` and `"sampleRateHertz": 16000`.
  21. `synapto-plugins/documents/src/service.rs:114-129, 151-156, 815-818, 832-845`: `#[serde(default)]` on `on_url_not_found`, wildcard `_ => 768`, hardcoded `"gemini-3-flash-preview"` and `max_characters: 2000` / `overlap: 200`.
  22. `synapto-plugins/speaker-recognizer/src/lib.rs:125-254, 1134-1137, 1159-1163`: `unwrap_or_else(|| PathBuf::from("speakers.json"))` and hardcoded `SpeakerConfigParams`.
  23. `synapto-plugins/stt-moonshine/src/{config.rs:6-56, lib.rs:39}`: `#[serde(default = "...")]` on all 4 fields and `optional_config()?.unwrap_or_default()`.
  24. `org-assistant/integrations/{call-recorder/src/lib.rs:9-14, 33-36, 85-89, google-chat/src/lib.rs:23-31, google-meet/src/lib.rs:21-42, 777-781}`: `#[derive(Default)]`, `#[serde(default)]`, and `.unwrap_or_else(...)` directory fallbacks.
- **Solution:**
  1. Remove `+ Default` from `PluggableCredentialsProvider::Config` and remove `PluginInitContext::optional_config`.
  2. Remove all `#[serde(default)]` attributes and `.unwrap_or(...)` operational fallbacks across all 24 locations, make all operational parameters explicit struct fields, and add `#[serde(deny_unknown_fields)]` to every configuration struct.
  3. Update all workspace `config.json` and test scenario configs to supply the explicit fields.

---

## 3. Current State Analysis (Exact File & Line Inventory)

See the 24-item inventory in Section 2 above. Every listed file and line range contains either `#[serde(default)]`, `#[derive(Default)]` on a config struct, `optional_config()`, or a runtime `.unwrap_or` / hardcoded operational parameter.

---

## 4. Structural Hierarchy

- **Mission:** Zero Implicit Defaults and Orthogonal Configuration Domains.
- **Goal:** Fail fast at startup on any missing or misspelled configuration field across `synapto`, `synapto-plugins`, and `org-assistant`.
- **Atomic Responsibilities:**
  - **`ConfigProvider` (`crates/core/src/config/`):** Validates root configuration domain keys strictly and deserializes each subsystem domain with `#[serde(deny_unknown_fields)]`.
  - **Plugin & Provider Config Structs:** Require every operational parameter explicitly without `#[serde(default)]`.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 28 (Explicit Configuration Mandate):** No `#[serde(default)]` or implicit runtime fallback values for operational parameters.
- **Rule 29 (Atomic Plugin Configuration Mandate):** Dedicated config struct per plugin with `#[serde(deny_unknown_fields)]`.
- **Rule 32 (Subsystem Configuration Domain Isolation):** Top-level configuration schema mirrors architectural roles 1:1.

### 5.2 Example Explicit Plugin Config (`GoogleSttConfig`)
```rust
// synapto/integrations/plugins/stt-google/src/lib.rs
#[derive(Debug, Clone, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct GoogleSttConfig {
    pub version: GoogleSttVersion,
    pub language_code: String,
    pub model: String,
    pub google_project_id: String,
    pub location: String,
}
```

### 5.3 JSON Contract Examples

#### Explicit Root Configuration (`config.json`)
```json
{
  "cognitive": {
    "audience": "organization",
    "barge_in": false,
    "disable_cognitive_direct": true,
    "disable_cognitive_side": false,
    "disable_ctrl_c": false,
    "plugin_init_timeout_secs": 30,
    "max_tool_turns": 5,
    "direct": {
      "model": "gemini-2.5-flash",
      "reasoning_effort": "None"
    },
    "side": {
      "model": "gemini-2.5-pro",
      "reasoning_effort": "Medium"
    }
  },
  "storage": {},
  "credentials": {},
  "decision": {},
  "llm": {},
  "plugins": {}
}
```

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For structuring the root configuration domains in `config.json` under Rule 32 (`synapto/crates/core/src/config.rs:66-96`):

**[ Option A: Consolidate Core Runtime & Prompt Settings Under `"cognitive"` (6 Top-Level Domains) ]**
- **Action:** Nest `audience`, `data_dir`, `barge_in`, `initial_run`, `disable_cognitive_direct`, `disable_cognitive_side`, `disable_ctrl_c`, `plugin_init_timeout_secs`, and `prompt` under `"cognitive"` so the root `config.json` has strictly the 6 domains listed in Rule 32 (`cognitive`, `storage`, `credentials`, `decision`, `llm`, `plugins`).
- **Pros:** Matches Rule 32 (`ARCHITECTURE.md:115, 179-185`) literally without adding new methods to `ConfigProvider`.
- **Cons:** Requires moving root-level flags into `"cognitive"` in `config.json`.

**[ Option B: Introduce a Dedicated `"core"` Top-Level Domain Separate from `"cognitive"` (7 Top-Level Domains) ]**
- **Action:** Place engine lifecycle flags (`audience`, `barge_in`, `plugin_init_timeout_secs`, `prompt`, etc.) under `"core"` and keep LLM model settings (`direct`, `side`, `max_tool_turns`) under `"cognitive"`.
- **Pros:** Cleanly separates engine runtime flags (`core`) from cognitive reasoning parameters (`cognitive`).
- **Cons:** Expands Rule 32 from 6 domains to 7 domains.

**My Recommendation:** I recommend **Option A** to match Rule 32 directly without adding a 7th domain.
Please select Option A or Option B for this RFC.
