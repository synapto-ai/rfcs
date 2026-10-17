# [MVP] [DRAFT] RFC: Telemetry Reloadable Filter Fix and Async GUI Error Channel

> **Classification:** `Flaw` (Broken Plugin Logging Filter, Stderr Channel Spam, & Recursive Subscriber Event)
> **Draft Status:** Reproduced (`rfcs/reproduced/`) — Failing scenario test `rfc_2026_10_17_telemetry_reloadable_filter_and_async_gui_error_channel` verified in `synapto/crates/test/tests/scenario_tests.rs`; awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-telemetry-reloadable-filter-and-async-gui-error-channel.md`.

## 1. User-Centric Motivation & Goal

Developers and operators rely on `synapto-telemetry` to automatically enable `TRACE`/`DEBUG`/`INFO` logging for every registered plugin crate via `Tracing::add_plugin_to_log`, and to route error logs cleanly without polluting `stderr` or blocking Tokio worker threads.

Currently, three bugs in `synapto/crates/telemetry/src/tracing.rs` break telemetry behavior:
1. **Plugin Logs Are Blocked Despite `Tracing::add_plugin_to_log`:** `Tracing::init` creates `log_filter` (`"warn,synapto=trace,..."`), wraps a clone in `reload::Layer::new(log_filter.clone())` at the outer subscriber level (`L314`), **and also attaches non-reloadable clones of the initial `log_filter` to `stdout_error_layer` (`L192`), `stdout_panic_layer` (`L219`), `stdout_non_error_layer` (`L245`), and `RerunLoggingLayer` (`L306`)**. When `Tracing::add_plugin_to_log` (`L332-338`) updates `reloadable_filter` to add `my_plugin=trace`, the per-layer static `log_filter` on all stdout and Rerun layers still has `warn` as the default directive, silently dropping all `INFO`, `DEBUG`, and `TRACE` logs from every plugin!
2. **`GuiErrorLayer` Closed-Channel `stderr` Spam & Sync `std::sync::mpsc` Receiver:** `GuiErrorLayer::new()` (`L350-358`) creates an unbounded synchronous `std::sync::mpsc::channel()`. When no `GuiPlugin` is registered in a bundle (`crates/core/src/lib.rs:976`), `self.error_rx` is dropped when `run_internal` starts. Every subsequent `tracing::error!` event hits `self.sender.send(...)` at `tracing.rs:374`, which fails and executes `eprintln!("{}", e)` (`sending on a closed channel`) at `tracing.rs:375`. Furthermore, `GuiPlugin::start` (`crates/interface/src/gui.rs:9`) receives a blocking `std::sync::mpsc::Receiver<String>` in an async trait.
3. **Re-Entrant `tracing::trace!` Inside `ShutdownDowngradeLayer::on_event` (`tracing.rs:9-24`):** When `is_shutting_down()` is true in release builds, `ShutdownDowngradeLayer::on_event` emits a new `tracing::trace!(...)` event from inside `Layer::on_event`, re-entering the tracing subscriber dispatch loop.

This RFC fixes the per-layer filter bug so `Tracing::add_plugin_to_log` actually enables plugin logs, replaces `GuiErrorLayer`'s synchronous unbounded channel with a bounded `tokio::sync::mpsc` channel that does not spam `stderr` when unused, and removes re-entrant event emission in `ShutdownDowngradeLayer`.

---

## 2. Executive Summary

- **Context:** `synapto/crates/telemetry/src/{lib.rs, tracing.rs}`, `synapto/crates/interface/src/gui.rs`, and `synapto/crates/core/src/lib.rs`.
- **Motivation:** Restore plugin log visibility on stdout and Rerun, eliminate `sending on a closed channel` stderr noise in headless/server bundles, and eliminate blocking `std::sync::mpsc` from `GuiPlugin::start`.
- **Overarching Goals:** Clean, non-blocking, dynamically reloadable telemetry filtering across all layers.
- **Purpose:** Remove redundant static `EnvFilter` clones from inner layers in `tracing.rs`, switch `GuiErrorLayer` and `GuiPlugin::start` to a bounded `tokio::sync::mpsc` channel using `try_send`, and fix `ShutdownDowngradeLayer`.
- **Problem:**
  1. **Static Per-Layer `EnvFilter` Clones Override `reloadable_filter` (`synapto/crates/telemetry/src/tracing.rs:108-110, 192, 219, 245, 306, 314, 332-338`):** Inner layers filter out plugin events before or after `reloadable_filter` because their static `EnvFilter` clones are never updated by `GLOBAL_RELOAD_HANDLE`.
  2. **Unbounded Sync Channel & `eprintln!` on Closed Receiver (`tracing.rs:350-377`, `crates/interface/src/gui.rs:9`, `crates/core/src/lib.rs:593, 976`):** `std::sync::mpsc::channel()` is unbounded, blocks async callers in `GuiPlugin::start`, and prints `sending on a closed channel` via `eprintln!` whenever no `GuiPlugin` is registered.
  3. **Re-Entrant Subscriber Dispatch (`tracing.rs:9-24`) & Hardcoded Unencrypted Docker Fallback (`tracing.rs:293-296`):** `ShutdownDowngradeLayer::on_event` calls `tracing::trace!` inside `on_event`, and `RerunLoggingLayer` falls back to `rerun+http://host.docker.internal:9876/proxy` (`// TODO consider security concerns`).
- **Solution:**
  1. Use `reloadable_filter` as the single `EnvFilter` for target/crate filtering, and use simple `filter_fn` predicates on `stdout_error_layer` (`metadata.level() == &Level::ERROR && metadata.target() != "panic"`), `stdout_panic_layer` (`metadata.target() == "panic"`), and `stdout_non_error_layer` (`metadata.level() > &Level::ERROR`).
  2. Change `GuiErrorLayer` and `GuiPlugin::start` to use `tokio::sync::mpsc::channel::<String>(256)` with `try_send`, silently ignoring `TrySendError::Closed` when no `GuiPlugin` is active.
  3. Convert `ShutdownDowngradeLayer` into a `Filter` (or `filter_fn`) on the error layer rather than re-emitting a new event inside `on_event`.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/crates/telemetry/src/tracing.rs:108-110, 192, 219, 245, 306, 314`**:
   ```rust
   let log_filter = EnvFilter::try_from_default_env().unwrap_or_else(|_| {
       EnvFilter::new("warn,synapto=trace,synapto_llm=trace,synapto_shutdown=trace,synapto_interface=trace,telemetry=trace")
   });
   ...
   // L192:
   .with_filter(log_filter.clone())
   // L219:
   .with_filter(log_filter.clone())
   // L245:
   .with_filter(log_filter.clone())
   // L306:
   let rerun_layer = RerunLoggingLayer::new(rec).with_filter(log_filter);
   // L314:
   let _ = subscriber.with(reloadable_filter).try_init();
   ```
2. **`synapto/crates/telemetry/src/tracing.rs:350-377`**:
   ```rust
   pub fn new() -> (Self, std::sync::mpsc::Receiver<String>) {
       let (sender, receiver) = std::sync::mpsc::channel();
       (Self { sender }, receiver)
   }
   ...
   if let Err(e) = self.sender.send(visitor.message) {
       eprintln!("{}", e);
   }
   ```
3. **`synapto/crates/telemetry/src/tracing.rs:9-24`**:
   ```rust
   impl<S: Subscriber> Layer<S> for ShutdownDowngradeLayer {
       fn on_event(&self, event: &tracing::Event<'_>, _ctx: Context<'_, S>) {
           if !cfg!(debug_assertions) && *event.metadata().level() == Level::ERROR && is_shutting_down() {
               ...
               tracing::trace!(is_downgraded = true, "{}", m);
           }
       }
   }
   ```

---

## 4. Structural Hierarchy

- **Mission:** Accurate Dynamic Log Filtering and Non-Blocking Error Telemetry.
- **Goal:** Ensure `Tracing::add_plugin_to_log` works across all layers and `GuiErrorLayer` is async-safe and silent when unused.
- **Atomic Responsibilities:**
  - **`Tracing::init` (`crates/telemetry/src/tracing.rs`):** Applies `reloadable_filter` once globally and uses level/target predicates on stdout layers.
  - **`GuiErrorLayer` (`crates/telemetry/src/tracing.rs`) & `GuiPlugin` (`crates/interface/src/gui.rs`):** Uses bounded `tokio::sync::mpsc::Sender<String>` (`try_send`) and `tokio::sync::mpsc::Receiver<String>`.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 20 & Async Channel Purity:** All plugin `start` methods in `synapto-interface` must use async `synapto_interface::sync::mpsc::Receiver` rather than blocking `std::sync::mpsc::Receiver`.

### 5.2 Updated `GuiPlugin` Signature
```rust
// synapto/crates/interface/src/gui.rs
#[async_trait]
pub trait GuiPlugin: Plugin {
    async fn start(
        &self,
        error_rx: crate::sync::mpsc::Receiver<String>,
    ) -> Result<(), String>;
}
```

### 5.3 JSON Contract Examples
_Not applicable (Telemetry subscriber internal infrastructure; no LLM prompt context or command schema impact)._

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For handling `ERROR` events during shutdown in release builds (`ShutdownDowngradeLayer` at `synapto/crates/telemetry/src/tracing.rs:9-24`):

**[ Option A: Filter Out Non-Panic `ERROR` Events on `stdout_error_layer` During Shutdown in Release Builds ]**
- **Action:** Remove `ShutdownDowngradeLayer` (and its re-entrant `tracing::trace!` call inside `on_event`) and update `stdout_error_layer`'s `filter_fn` to suppress non-panic `ERROR` events when `!cfg!(debug_assertions) && is_shutting_down()`.
- **Pros:** Eliminates re-entrant subscriber dispatch completely while preserving the exact release-mode stdout behavior (suppressing noisy channel-closed errors during shutdown).
- **Cons:** Downgraded shutdown errors are suppressed rather than re-emitted as `TRACE`.

**[ Option B: Route `stdout_non_error_layer` to Accept `ERROR` Events When `is_shutting_down()` in Release Builds ]**
- **Action:** Remove `ShutdownDowngradeLayer` and adjust the complementary `filter_fn` predicates on `stdout_error_layer` and `stdout_non_error_layer` so that when `!cfg!(debug_assertions) && is_shutting_down()`, `ERROR` events are formatted by `stdout_non_error_layer` instead of `stdout_error_layer` without re-entering `tracing::trace!`.
- **Pros:** Preserves the log message on stdout/Rerun during shutdown without re-emitting a synthetic event inside `on_event`.
- **Cons:** Still prints the event on `stdout_non_error_layer` if `ERROR` passes the filter.

**My Recommendation:** I recommend **Option A** because the sole purpose of `ShutdownDowngradeLayer` was to prevent benign teardown errors from printing to `stdout_error_layer` in release builds when `TRACE` is filtered out.
Please select Option A or Option B for this RFC.
