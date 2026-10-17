# [MVP] [DRAFT] RFC: Broadcast Lag Resilience and Direct Audio Stream Wiring

> **Classification:** `Flaw` (Runtime Liveness & Stream Survival)
> **Draft Status:** Reproduced (`rfcs/reproduced/`) — Failing scenario test `rfc_2026_10_17_broadcast_lag_resilience_and_direct_audio_wiring` implemented in `synapto/crates/test/tests/scenario_tests.rs`; awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-broadcast-lag-resilience-and-direct-audio-wiring.md`.

## 1. User-Centric Motivation & Goal

When users converse with an AI assistant over text chat or live voice, a brief burst of incoming messages or a momentary network/CPU spike must never permanently break the assistant's ability to hear or respond.

Currently, if more than 10 chat messages arrive while the assistant is waiting for an LLM response, or if audio streaming lags by more than 1.6 seconds (20 chunks), `tokio::sync::broadcast::Receiver::recv()` returns `Err(RecvError::Lagged(n))`. The core loops in `cognitive_side_task`, the Speech-to-Text (STT) audio bridge, and the speaker output bridge treat any `Err` from `recv()` as channel closure and break out of their main `loop`, permanently killing text chat or audio processing for the remainder of the process lifetime.

This RFC fixes broadcast lag handling across the core switchboard and eliminates unnecessary intermediate `broadcast` bridges on high-bandwidth audio paths.

---

## 2. Executive Summary

- **Context:** `synapto` (`crates/core`) routes incoming text (`PeerInputText`), voice audio (`InputVoiceAudio`), and synthesized speech audio (`CognitiveOutputAudio`) between plugins and cognitive tasks.
- **Motivation:** Treating `broadcast::error::RecvError::Lagged` as a terminal stream EOF causes silent, unrecoverable death of chat and voice subsystems under burst load.
- **Overarching Goals:** Guarantee that transient consumer lag never terminates any long-running cognitive or audio pipeline task, and adhere to Rule 23 (Direct Wiring) and Phase 3 Channel Suitability (`mpsc` over `broadcast` for high-bandwidth audio).
- **Purpose:** Fix all broadcast receiver loops in `crates/core` and `synapto-plugins/speaker-recognizer` to survive `RecvError::Lagged`, and replace intermediate audio broadcast bridge loops with direct `mpsc` wiring.
- **Problem:**
  1. **`cognitive_side_task` Dies on >10 Queued Text Messages (`synapto/crates/core/src/cognitive/side.rs:195-198` & `lib.rs:771`):** `peer_input_text_broadcast_tx` has capacity `10`. In `side.rs:195-198`, `let Ok(msg) = res else { break; };` exits `cognitive_side_task` permanently when `res` is `Err(RecvError::Lagged(_))`.
  2. **STT Audio Bridge Dies on >1.6s Lag (`synapto/crates/core/src/speech_to_text.rs:23, 34-43`):** `input_voice_audio_tx` is `broadcast::channel(20)`. The intermediate bridge task at `speech_to_text.rs:36` runs `while let Ok(msg) = input_voice_audio_rx_bridge.recv().await`, exiting permanently on `Err(RecvError::Lagged(_))`.
  3. **Speaker Audio Output Bridge Dies on >10 TTS Chunks (`synapto/crates/core/src/lib.rs:784, 792-800` & `speaking_coordinator.rs:34-40`):** `cognitive_output_audio_tx` is `broadcast::channel(10)`. The bridge task at `lib.rs:793` runs `while let Ok(msg) = cognitive_output_audio_rx_broadcast.recv().await`, exiting permanently on `Err(RecvError::Lagged(_))`.
  4. **CallPlugin Text Broadcast Loop (`org-assistant/integrations/google-meet/src/lib.rs:321`):** Uses `while let Ok(msg) = peer_input_text_rx.recv().await`, which also terminates `wait_and_join_meet_task` on `RecvError::Lagged`.
- **Solution:**
  1. Replace `while let Ok(msg) = rx.recv().await` and `let Ok(msg) = res else { break; }` on all `broadcast::Receiver` loops with explicit `match` blocks that log `RecvError::Lagged(skipped)` via `tracing::warn!` and continue the loop, breaking only on `RecvError::Closed`.
  2. Increase `peer_input_text_broadcast_tx` capacity from `10` to `128`.
  3. Wire `STTPlugin` and `AudioOutputPlugin` via direct `mpsc` channels (Rule 23) with optional `mpsc` fan-out taps for `DiarizationPlugin` and `AudioRecorderPlugin`.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/crates/core/src/lib.rs:770-781`**:
   ```rust
   let (peer_input_text_broadcast_tx, _) = broadcast::channel::<PeerInputText>(10);
   ...
   tokio::spawn(async move {
       while let Some(msg) = peer_input_text_rx.recv().await {
           if let Err(e) = broadcast_tx_clone.send(msg) {
               tracing::error!("{}", e);
           }
       }
   });
   ```
2. **`synapto/crates/core/src/cognitive/side.rs:195-198`**:
   ```rust
   res = text_rx.recv() => {
       let Ok(msg) = res else {
           break;
       };
       current_messages.push(msg);
   }
   ```
3. **`synapto/crates/core/src/speech_to_text.rs:23, 34-43`**:
   ```rust
   let (input_voice_audio_tx, input_voice_audio_rx) = broadcast::channel::<InputVoiceAudio>(20);
   let (plugin_voice_audio_tx, plugin_voice_audio_rx) = mpsc::channel::<InputVoiceAudio>(20);
   let mut input_voice_audio_rx_bridge = input_voice_audio_tx.subscribe();
   tokio::spawn(async move {
       while let Ok(msg) = input_voice_audio_rx_bridge.recv().await {
           if plugin_voice_audio_tx.send(msg).await.is_err() {
               break;
           }
       }
   });
   ```
4. **`synapto/crates/core/src/lib.rs:783-800` & `synapto/crates/core/src/speaking_coordinator.rs:34-40`**:
   ```rust
   let (cognitive_output_audio_tx, _) = broadcast::channel::<CognitiveOutputAudio>(10);
   let (cognitive_output_audio_tx_plugin, cognitive_output_audio_rx_plugin) = mpsc::channel::<CognitiveOutputAudio>(10);
   let (cognitive_output_audio_tx_speaker, cognitive_output_audio_rx_speaker) = mpsc::channel::<CognitiveOutputAudio>(10);
   let mut cognitive_output_audio_rx_broadcast = cognitive_output_audio_tx.subscribe();
   tokio::spawn(async move {
       while let Ok(msg) = cognitive_output_audio_rx_broadcast.recv().await {
           if cognitive_output_audio_tx_speaker.send(msg).await.is_err() {
               break;
           }
       }
   });
   ```
5. **`org-assistant/integrations/google-meet/src/lib.rs:321`**:
   ```rust
   while let Ok(msg) = peer_input_text_rx.recv().await {
   ```

---

## 4. Structural Hierarchy

- **Mission:** Stream Liveness and Zero-Intermediary Audio Coupling.
- **Goal:** Prevent task termination on broadcast lag and eliminate intermediate audio bridge tasks.
- **Atomic Responsibilities:**
  - **`BroadcastRecvLoop` (`crates/core/src/cognitive/side.rs`, `org-assistant/integrations/google-meet/src/lib.rs`):** Matches `Ok(msg)`, `Err(RecvError::Lagged(skipped))` (logs warning and continues), and `Err(RecvError::Closed)` (breaks).
  - **`detect_voice_task` (`crates/core/src/speech_to_text/detect_voice.rs`):** Sends `InputVoiceAudio` directly to `stt_voice_audio_tx: mpsc::Sender<InputVoiceAudio>` and optional `diarization_audio_tx` / `recorder_audio_tx`.
  - **`speaking_coordinator_task` (`crates/core/src/speaking_coordinator.rs`):** Forwards `CognitiveOutputAudio` directly to `speaker_audio_tx: mpsc::Sender<CognitiveOutputAudio>` and optional `recorder_audio_tx`.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 8 & 23 (Direct Channel Coupling & Direct Wiring Principle):** Connect producer and consumer tasks directly via `mpsc` without intermediate translation/bridge tasks.
- **Phase 3 Channel Suitability (`rfcs/AGENTS.md`):** High-bandwidth audio streams (`CognitiveOutputAudio`, `InputVoiceAudio`) must prefer backpressured `mpsc` channels over lossy `broadcast` channels.

### 5.2 Lag-Resilient Broadcast Pattern
```rust
res = text_rx.recv() => {
    match res {
        Ok(msg) => current_messages.push(msg),
        Err(tokio::sync::broadcast::error::RecvError::Lagged(skipped)) => {
            tracing::warn!("Text receiver lagged by {} messages", skipped);
            continue;
        }
        Err(tokio::sync::broadcast::error::RecvError::Closed) => break,
    }
}
```

### 5.3 JSON Contract Examples

#### Compiled Prompt Context Input (`current_messages`)
```json
{
  "current_messages": [
    {
      "user": { "Recognized": "Alice" },
      "content": "Can you summarize the latest sprint updates?",
      "channel": { "plugin": "google_chat", "context": "{}" },
      "attached_documents": []
    }
  ]
}
```

#### LLM Structured Output Command Payload (`CognitiveSideCommands`)
```json
{
  "reasoning": "Responding to the buffered user messages in order.",
  "commands": {
    "write": {
      "content": "Here is the summary of the sprint updates.",
      "target": { "plugin": "google_chat", "context": "{}" }
    }
  }
}
```

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For wiring `InputVoiceAudio` (`speech_to_text.rs:23-43`) and `CognitiveOutputAudio` (`lib.rs:783-800`):

**[ Option A: Direct `mpsc` Wiring + Optional Tap Senders in Producer ]**
- **Action:** Remove the `broadcast` channels and spawned bridge tasks for `InputVoiceAudio` and `CognitiveOutputAudio`. Pass `mpsc::Sender` directly into `detect_voice_task` and `speaking_coordinator_task` (with optional secondary `mpsc::Sender`s when `DiarizationPlugin` or `AudioRecorderPlugin` is registered), and change `DiarizationPlugin::start` / `AudioRecorderPlugin::start` to accept `mpsc::Receiver`.
- **Pros:** Eliminates two intermediate bridge tasks, complies with Rule 23 and Phase 3 Channel Suitability, and prevents silent audio frame dropping.
- **Cons:** Updates `DiarizationPlugin::start` and `AudioRecorderPlugin::start` channel receiver type from `broadcast::Receiver` to `mpsc::Receiver`.

**[ Option B: Retain `broadcast` for Multi-Consumer Tap, Fix `RecvError::Lagged` in All Bridges, and Increase Capacity ]**
- **Action:** Keep the existing `broadcast::Receiver` signature on `DiarizationPlugin` and `AudioRecorderPlugin`, increase buffer capacities from `10`/`20` to `256`, and replace `while let Ok(msg) = rx.recv().await` in `speech_to_text.rs:36` and `lib.rs:793` with explicit `RecvError::Lagged` handling.
- **Pros:** Zero trait signature changes on `DiarizationPlugin` or `AudioRecorderPlugin`.
- **Cons:** Keeps intermediate bridge tasks in `speech_to_text.rs` and `lib.rs`.

**My Recommendation:** I recommend **Option A** for strict Rule 23 compliance, or **Option B** if preserving `DiarizationPlugin`/`AudioRecorderPlugin` trait signatures is preferred.
Please select Option A or Option B for this RFC.
