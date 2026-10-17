# [MVP] [DRAFT] RFC: Non-Blocking Plugin Task Spawning and Async I/O Offloading

> **Classification:** `Guideline` (Rule 11 Asynchronous Edge Ingestion & Rule 20 Encapsulated Task Orchestration)
> **Draft Status:** Phase 2 — Awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-non-blocking-plugin-start-and-async-io-offloading.md`.

## 1. User-Centric Motivation & Goal

Rule 20 (`ARCHITECTURE.md:74-78`, Encapsulated Task Orchestration) states that every plugin's `start` method must use `tokio::spawn` to launch its internal background tasks and return `Ok(())` immediately to the orchestrator. Rule 11 (`ARCHITECTURE.md:29`, Asynchronous Edge Ingestion) states that plugins must never block async ingestion loops or Tokio worker threads with heavy synchronous I/O or CPU-bound encoding.

Currently, all five cloud STT and TTS plugins (`stt-elevenlabs`, `stt-google`, `stt-speechmatics`, `tts-elevenlabs`, `tts-google`) run their infinite event loops directly inside `start().await` without spawning a task, hanging any caller that awaits `plugin.start(...)`. Furthermore, `call-recorder` runs synchronous `std::fs` file writes and CPU-bound `FlacSampleWriter` encoding inside its async loop, `skills` executes synchronous `std::fs::read_dir` and `Path::canonicalize` inside `ContextProvider::context` (called on every cognitive turn) and `Tool::execute`, `mumble` runs synchronous Opus audio encoding/decoding inside its single `tokio_select!` network heartbeat loop, and `google-chat` performs an inline HTTP `GET` for space details inside its primary Pub/Sub message loop.

This RFC makes all STT and TTS `start()` methods non-blocking via internal `tokio::spawn` and offloads synchronous filesystem and audio codec operations from Tokio async worker threads.

---

## 2. Executive Summary

- **Context:** `synapto/integrations/plugins/{stt-elevenlabs, stt-google, stt-speechmatics, tts-elevenlabs, tts-google, skills, mumble}`, `org-assistant/integrations/{call-recorder, google-chat, google-mail}`, and `synapto-plugins/documents`.
- **Motivation:** Comply with Rule 20 (self-spawning `start()` methods) and Rule 11 (non-blocking async worker threads).
- **Overarching Goals:** Ensure every `Plugin::start` implementation returns `Ok(())` immediately after spawning its worker tasks, and prevent synchronous filesystem/codec calls from starving the Tokio executor.
- **Purpose:** Wrap STT/TTS infinite loops in `tokio::spawn`, move `call-recorder`, `skills`, and `mumble` synchronous work off async worker threads, reuse a shared `reqwest::Client` in `google-mail`, and move `handle_add_document` binary saving off the synchronous fast path.
- **Problem:**
  1. **Blocking `start()` in Cloud STT/TTS Plugins (Rule 20 Violation):**
     - `synapto/integrations/plugins/stt-elevenlabs/src/lib.rs:53-72`
     - `synapto/integrations/plugins/stt-google/src/lib.rs:83-112`
     - `synapto/integrations/plugins/stt-speechmatics/src/lib.rs:83-102`
     - `synapto/integrations/plugins/tts-elevenlabs/src/lib.rs:51-68`
     - `synapto/integrations/plugins/tts-google/src/lib.rs:56-68`
     All five `await` their infinite loops directly inside `start()` instead of spawning via `tokio::spawn` and returning `Ok(())`.
  2. **Synchronous File I/O and FLAC Encoding in `call-recorder` (`org-assistant/integrations/call-recorder/src/lib.rs:38, 91-145`):** Calls synchronous `std::fs::create_dir_all`, `std::fs::File::create`, and `FlacSampleWriter::write` directly inside an async `tokio::spawn` task.
  3. **Synchronous Directory Scanning in `skills` (`synapto/integrations/plugins/skills/src/{context.rs:141-144, tool.rs:107-109, 123-149}`):** Calls `std::fs::read_dir` and `canonicalize()` on the Tokio worker thread during `ContextProvider::context` and `Tool::execute`.
  4. **Synchronous Opus Codec in `mumble` Network Loop (`synapto/integrations/plugins/mumble/src/lib.rs:345, 386, 398`):** Encodes and decodes full TTS/voice Opus buffers synchronously inside the `tokio_select!` loop that also services Mumble ping heartbeats.
  5. **Per-Call `reqwest::Client::new()` in `google-mail` (`org-assistant/integrations/google-mail/src/tools.rs:47`) & Inline Space HTTP GET in `google-chat` (`org-assistant/integrations/google-chat/src/read.rs:227-241`):** Re-allocates connection pools per tool call in `google-mail` and blocks the batch loop on space metadata lookup in `google-chat`.
- **Solution:**
  1. In all five STT/TTS plugins, spawn the worker task with `tokio::spawn` inside `start()` (checking `synapto_shutdown::is_shutting_down()` in reconnect loops) and return `Ok(())` immediately; remove the redundant outer `tokio::spawn` wrapper in `crates/core/src/lib.rs:1168, 1189` (or keep it for uniform error logging).
  2. Offload `call-recorder` FLAC encoding and `skills` recursive directory reads to `tokio::task::spawn_blocking` (or `tokio::fs`).
  3. Offload `mumble` TTS Opus encoding to `spawn_blocking` before streaming packets in the network loop.
  4. Store a reusable `reqwest::Client` (with explicit connect/request timeouts) on `GoogleMailPlugin` / `CreateGmailDraftTool` and move the space lookup in `google-chat` into the spawned per-message handler task (with an in-memory space type cache).

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/integrations/plugins/stt-google/src/lib.rs:83-112`** (representative of all 5 STT/TTS plugins):
   ```rust
   async fn start(
       &self,
       voice_audio_rx: mpsc::Receiver<InputVoiceAudio>,
       speech_transcript_tx: mpsc::Sender<SpeechTranscript>,
       speech_detected_tx: mpsc::Sender<()>,
   ) -> Result<(), String> {
       match self.config.version {
           GoogleSttVersion::V1 => self.run_v1(voice_audio_rx, speech_transcript_tx, speech_detected_tx).await,
           GoogleSttVersion::V2 => self.run_v2(voice_audio_rx, speech_transcript_tx, speech_detected_tx).await,
       }
   }
   ```
2. **`org-assistant/integrations/call-recorder/src/lib.rs:91-115`**:
   ```rust
   let file = std::fs::File::create(&file_path)...;
   let mut writer = FlacSampleWriter::new(file, ...);
   writer.write(&samples)...;
   writer.finalize()...;
   ```
3. **`synapto/integrations/plugins/skills/src/tool.rs:123-149`**:
   ```rust
   pub(crate) fn collect_files_recursive(root: &Path, current: &Path, out: &mut Vec<String>) {
       let Ok(entries) = std::fs::read_dir(current) else { return; };
       ...
   }
   ```

---

## 4. Structural Hierarchy

- **Mission:** Non-Blocking Async Task Execution and Self-Starting Plugin Encapsulation.
- **Goal:** Ensure 100% compliance with Rules 11 and 20 across all plugins.
- **Atomic Responsibilities:**
  - **`STTPlugin::start` / `TTSPlugin::start`:** Spawns the internal streaming loop via `tokio::spawn` and returns `Ok(())`.
  - **`CallRecorderPlugin::start` (`org-assistant/integrations/call-recorder`):** Executes FLAC file encoding inside `tokio::task::spawn_blocking`.
  - **`SkillsContextProvider` / `LoadSkillTool` (`synapto/integrations/plugins/skills`):** Caches skill file listings at discovery time or runs filesystem traversal via `tokio::fs` / `spawn_blocking`.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 11 (Asynchronous Edge Ingestion):** Heavy I/O and CPU-bound encoding must never block primary async ingestion loops or Tokio worker threads.
- **Rule 20 (Encapsulated Task Orchestration):** `start` functions use `tokio::spawn` to launch internal tasks and return immediately to the orchestrator.

### 5.2 Self-Spawning `STTPlugin::start` Pattern
```rust
async fn start(
    &self,
    voice_audio_rx: mpsc::Receiver<InputVoiceAudio>,
    speech_transcript_tx: mpsc::Sender<SpeechTranscript>,
    speech_detected_tx: mpsc::Sender<()>,
) -> Result<(), String> {
    let plugin = self.clone();
    tokio::spawn(async move {
        if let Err(e) = plugin.run_loop(voice_audio_rx, speech_transcript_tx, speech_detected_tx).await {
            tracing::error!("{}", e);
        }
    });
    Ok(())
}
```

### 5.3 JSON Contract Examples
_Not applicable (Task spawning and async I/O offloading; no LLM prompt or command schema changes)._

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For `collect_files_recursive` in `synapto/integrations/plugins/skills/src/{context.rs:141, tool.rs:123}`:

**[ Option A: Cache Bundled File Paths During Initial Skill Discovery (`discover_skills`) ]**
- **Action:** Populate `bundled_files: Vec<String>` once when `discover_skills` scans the skill directories at plugin initialization, storing it on `SkillMetadata` so `SkillsContextProvider::context` (called on every cognitive turn) and `LoadSkillTool::execute` read the precomputed `Vec<String>` from memory with zero per-turn filesystem I/O.
- **Pros:** Eliminates redundant recursive `read_dir` syscalls on every cognitive turn, reducing turn latency and avoiding `spawn_blocking` overhead during prompt compilation.
- **Cons:** Files added to a skill directory after process startup require a restart to appear in `bundled_files` (matching how `discover_skills` already works for `SKILL.md`).

**[ Option B: Wrap Per-Turn `collect_files_recursive` in `tokio::task::spawn_blocking` ]**
- **Action:** Keep scanning the directory on every `context()` and `execute()` call, wrapped in `tokio::task::spawn_blocking`.
- **Pros:** Picks up newly created files inside an existing skill folder without restart.
- **Cons:** Performs unnecessary disk I/O on every cognitive turn.

**My Recommendation:** I recommend **Option A** because `discover_skills` already loads skills once at startup (`plugin.rs:32`), so caching `bundled_files` on `SkillMetadata` eliminates per-turn disk I/O completely.
Please select Option A or Option B for this RFC.
