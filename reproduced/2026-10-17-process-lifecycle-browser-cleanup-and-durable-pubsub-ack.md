# [MVP] [DRAFT] RFC: External Process Lifecycle, Browser Cleanup, and Durable Pub/Sub Acknowledgement

> **Classification:** `Flaw` (Orphaned Processes, Unrecovered Panics, Message Loss on Crash, & Disabled TLS Verification)
> **Draft Status:** Reproduced (`rfcs/reproduced/`) — Failing scenario test `rfc_2026_10_17_process_lifecycle_browser_cleanup_and_durable_pubsub_ack` verified in `synapto/crates/test/tests/scenario_tests.rs`; awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-process-lifecycle-browser-cleanup-and-durable-pubsub-ack.md`.

## 1. User-Centric Motivation & Goal

When an AI assistant spawns external MCP tool servers, launches headless Chromium browsers to join Google Meet calls, sends replies to Google Chat, or ingests chat attachments via Google Cloud Pub/Sub, it must clean up child processes deterministically, recover from transient network/DOM failures without crashing, and never acknowledge a Pub/Sub message before its attachments are safely ingested.

Currently, five operational flaws exist across external I/O plugins:
1. **Orphaned MCP Child Processes (`synapto/integrations/plugins/mcp/src/lib.rs:219-231`):** `tokio::process::Command` spawns stdio MCP servers without `.kill_on_drop(true)` and immediately drops the `Child` struct after taking `stdin`/`stdout`, leaving orphaned MCP server processes running after Synapto exits.
2. **Leaked Chromium Processes, `std::process::exit`, & Unfiltered Meet Joins (`org-assistant/integrations/google-meet/src/lib.rs:321-372, 384-739, 787`):** `GoogleMeetPlugin::start` calls `std::process::exit(0)` (`L787`), `meet_join` uses `.unwrap_or_else(|e| panic!(...))` 18 times on transient Playwright operations (skipping `browser.close().await` at `L736` and leaking Chromium processes), and `wait_and_join_meet_task` (`L321-372`) joins any `meet.google.com/` URL seen in any room even when the bot was not mentioned and injects hardcoded English strings directly into `cognitive_output_text_tx`.
3. **Panic in Google Chat Writer Task (`org-assistant/integrations/google-chat/src/write.rs:89-101`):** Uses `.unwrap_or_else(|e| panic!(...))` when reading the HTTP response JSON body, permanently killing `cognitive_output_text_task` on a single transient HTTP read error.
4. **Premature Pub/Sub ACK Before Attachment Ingestion (`org-assistant/integrations/google-chat/src/read.rs:68-71, 313-432`):** Acknowledges Pub/Sub messages at the end of the HTTP poll batch while attachment downloads and `peer_input_text_tx.send(chat_msg)` run asynchronously inside an unawaited `tokio::spawn`. If the download fails or the container restarts, the message is already acknowledged and permanently lost.
5. **Unconditional TLS Verification Bypass in Mumble (`synapto/integrations/plugins/mumble/src/lib.rs:236-238`):** Hardcodes `danger_accept_invalid_certs(true)` on all Mumble TLS connections without an explicit opt-in configuration flag.

This RFC fixes process and browser lifecycle cleanup, replaces transient I/O panics with structured error recovery, defers Pub/Sub acknowledgement until ingestion completes, and makes Mumble TLS certificate verification secure by default.

---

## 2. Executive Summary

- **Context:** `synapto/integrations/plugins/{mcp, mumble}`, `synapto-plugins/speaker-recognizer`, and `org-assistant/integrations/{google-chat, google-meet}`.
- **Motivation:** Prevent resource leaks (orphaned MCP and Chromium processes), prevent task death on transient network/DOM errors, prevent lost chat messages on restart, and enforce TLS verification by default.
- **Overarching Goals:** Deterministic child process cleanup, resilient I/O loops, at-least-once Pub/Sub ingestion durability, and secure-by-default TLS.
- **Purpose:** Add `kill_on_drop(true)` and child ownership in `mcp`, replace panics and `std::process::exit` in `google-meet`, `google-chat`, and `speaker-recognizer`, defer Pub/Sub ACK in `google-chat`, and gate `danger_accept_invalid_certs` behind an explicit config field in `mumble`.
- **Problem:**
  1. **`synapto/integrations/plugins/mcp/src/lib.rs:219-231`:** `Command::new(command)` omits `.kill_on_drop(true)` and drops `child` at the end of `spawn_stdio_mcp_server`.
  2. **`org-assistant/integrations/google-meet/src/lib.rs:321-372, 384-739, 787`:**
     - `L787`: `std::process::exit(0)`.
     - `L384, 390, 407, 426, 475, 495, 499, 504, 593, 604, 608, 646, 657, 674, 693, 730, 734, 739`: `.unwrap_or_else(|e| panic!("Error: {:?}", e))` skips `browser.close().await` (`L736`).
     - `L320-372`: Joins Meet links even when `is_mention_or_dm` is false (`//FIXME mention or dm` at `L320`).
  3. **`synapto-plugins/speaker-recognizer/src/lib.rs:1207`:** Calls `std::process::exit(1)` on audio channel lag.
  4. **`org-assistant/integrations/google-chat/src/write.rs:89-101`:** Panics on `response.json().await` failure inside `cognitive_output_text_task`.
  5. **`org-assistant/integrations/google-chat/src/read.rs:68-71, 313-432`:** Pushes `ack_id` to `ack_ids` (`L69-71`) and ACKs immediately (`L416-432`) before the spawned task (`L313-413`) downloads attachments and sends `PeerInputText`.
  6. **`synapto/integrations/plugins/mumble/src/lib.rs:236-238`:** Unconditionally sets `.danger_accept_invalid_certs(true)`.
- **Solution:**
  1. Set `cmd.kill_on_drop(true)` in `mcp/src/lib.rs` and move `child` into the spawned stdio actor task so it is terminated when the actor exits.
  2. In `google-meet/src/lib.rs`, replace `std::process::exit(0)` with `synapto_shutdown::shutdown()`, extract the inner body of `meet_join` into a helper returning `Result<(), String>` so `browser.close().await` is **always** executed before `meet_join` returns, and check `msg.is_mention_or_dm` before joining a Meet URL.
  3. In `google-chat/src/write.rs:89-101`, log HTTP response body read errors via `tracing::error!` and continue the writer loop instead of panicking.
  4. In `google-chat/src/read.rs`, acknowledge each Pub/Sub `ack_id` only after its spawned ingestion task finishes downloading attachments and sending `PeerInputText` to `peer_input_text_tx`.
  5. In `mumble/src/lib.rs`, add `pub accept_invalid_certs: bool` to `MumbleConfig` and pass `config.accept_invalid_certs` to `.danger_accept_invalid_certs(...)`.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/integrations/plugins/mcp/src/lib.rs:219-231`**:
   ```rust
   let mut cmd = tokio::process::Command::new(command);
   cmd.args(args)
       .envs(env)
       .stdin(std::process::Stdio::piped())
       .stdout(std::process::Stdio::piped())
       .stderr(std::process::Stdio::inherit());
   let mut child = cmd.spawn().map_err(|e| format!("Failed to spawn MCP server: {e}"))?;
   let stdin = child.stdin.take().ok_or("Failed to open stdin")?;
   let stdout = child.stdout.take().ok_or("Failed to open stdout")?;
   ```
2. **`org-assistant/integrations/google-meet/src/lib.rs:320-325, 384-739, 783-788`**:
   ```rust
   // L320-325:
   //FIXME mention or dm
   while let Ok(msg) = peer_input_text_rx.recv().await {
       let text = &msg.content;
       if let Some(idx) = text.find("meet.google.com/") { ... }
   }
   // L783-788:
   if self.config.setup {
       meet_setup(self.config.clone(), data_dir).await;
       std::process::exit(0);
   }
   ```
3. **`org-assistant/integrations/google-chat/src/write.rs:88-101`**:
   ```rust
   if response.status().is_success() {
       let created_msg: CreatedMessage = response
           .json()
           .await
           .unwrap_or_else(|e| panic!("Async operation failed: {:?}", e));
       ...
   } else {
       let error_body: serde_json::Value = response
           .json()
           .await
           .unwrap_or_else(|e| panic!("Async operation failed: {:?}", e));
       tracing::error!("Failed to send message: {:?}", error_body);
   }
   ```
4. **`org-assistant/integrations/google-chat/src/read.rs:68-71, 313-432`**:
   `ack_ids.push(received_message.ack_id)` at `L69-71` is acknowledged at `L416-432` before the spawned task at `L313-413` finishes downloading attachments and sending `PeerInputText`.
5. **`synapto/integrations/plugins/mumble/src/lib.rs:236-238`**:
   ```rust
   let connector = native_tls::TlsConnector::builder()
       .danger_accept_invalid_certs(true)
       .build()
   ```

---

## 4. Structural Hierarchy

- **Mission:** Deterministic External Process Cleanup, Crash-Free I/O Loops, and Durable Message Ingestion.
- **Goal:** Eliminate orphaned processes, transient network/DOM panics, premature Pub/Sub ACKs, and hardcoded TLS bypasses.
- **Atomic Responsibilities:**
  - **`McpStdioActor` (`plugins/mcp/src/lib.rs`):** Owns `child: tokio::process::Child` with `kill_on_drop(true)`.
  - **`meet_join` (`org-assistant/integrations/google-meet/src/lib.rs`):** Runs session logic inside an inner `async fn meet_join_session(page: &Page, ...) -> Result<(), String>` and unconditionally executes `browser.close().await` afterward.
  - **`cognitive_output_text_task` (`org-assistant/integrations/google-chat/src/write.rs`):** Logs HTTP/JSON errors via `tracing::error!` without panicking.
  - **`read_from_chat_task` (`org-assistant/integrations/google-chat/src/read.rs`):** Acknowledges each `ack_id` only after `peer_input_text_tx.send(chat_msg).await` succeeds.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Robust by Design Protocol:** Deterministic resource cleanup (`browser.close().await`, `kill_on_drop(true)`) and structured error handling on all external network/DOM operations.
- **Pragmatic Security by Default Protocol:** TLS certificate verification must be enabled by default unless explicitly configured otherwise.

### 5.2 `MumbleConfig` TLS Field
```rust
// synapto/integrations/plugins/mumble/src/lib.rs
pub struct MumbleConfig {
    pub host: String,
    pub port: u16,
    pub username: String,
    pub channel: Option<String>,
    pub accept_invalid_certs: bool,
}
```

### 5.3 JSON Contract Examples
_Not applicable (External process/transport lifecycle and Pub/Sub acknowledgement mechanics; no LLM schema changes)._

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For handling Google Meet URL joining in `org-assistant/integrations/google-meet/src/lib.rs:320-372`:

**[ Option A: Require `msg.is_mention_or_dm` Before Joining a Meet Link in `wait_and_join_meet_task` ]**
- **Action:** Check `if !msg.is_mention_or_dm { continue; }` at `google-meet/src/lib.rs:322` (resolving the `//FIXME mention or dm` comment) so the assistant only joins a Google Meet call when explicitly mentioned or messaged in a 1:1 DM.
- **Pros:** Prevents the assistant from barging into every `meet.google.com/` link shared between coworkers in public channels when the bot was not invited, with a 1-line fix.
- **Cons:** Keeps URL detection inside `google-meet` rather than migrating to an LLM `Command` / `Tool`.

**[ Option B: Expose a `join_call` LLM Tool/Command Instead of Substring Scanning `PeerInputText` ]**
- **Action:** Remove `peer_input_text_rx` substring scanning from `google-meet` and register a `join_meet_call` `Command` or `Tool` so the Cognitive Core decides when to join a call and what message to write back to chat.
- **Pros:** Aligns with Rule 7 (Separation of I/O and Processing: plugins should not contain domain routing heuristics or inject hardcoded English chat responses bypassing the Cognitive Core).
- **Cons:** Adds an LLM inference turn before joining a meeting.

**My Recommendation:** I recommend **Option A** for immediate MVP flaw remediation (or **Option B** if you want the LLM to control call joining and chat responses).
Please select Option A or Option B for this RFC.
