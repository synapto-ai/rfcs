# [MVP] [DRAFT] RFC: LocalStorage Concurrency Synchronization and EphemeralDir Leak Fix

> **Classification:** `Flaw` (Concurrent File Corruption & Disk Directory Leak)
> **Draft Status:** Reproduced (`rfcs/reproduced/`) — Failing scenario test `rfc_2026_10_17_local_storage_concurrency_locks_and_ephemeral_dir_leak` verified in `synapto/crates/test/tests/scenario_tests.rs`; awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-local-storage-concurrency-locks-and-ephemeral-dir-leak.md`.

## 1. User-Centric Motivation & Goal

When running local development, personal assistants, or automated scenario tests using `LocalStorage` and `EphemeralDir`, concurrent background tasks (such as interaction memory, task memory, semantic memory, and document storage) read and write records simultaneously.

Currently, two flaws break local storage reliability:
1. **Concurrent Read-Modify-Write Corruption in `LocalStorage`:** Every mutating method in `LocalStorage` (`upsert_record`, `delete_record`, `trim_records_before`, `set`, `delete`, `delete_vectors`) reads the entire JSON collection file from disk, modifies it in memory, and writes the file back with **zero synchronization lock**. When two tasks update records in the same collection or namespace concurrently, they overwrite each other's changes or read partially written JSON files.
2. **Per-Call Tempdir Leak and State Split in `EphemeralDir`:** `EphemeralDir::get_data_dir()` (`synapto/crates/core/src/storage/local_storage.rs:570-574` and `synapto/crates/test/src/ephemeral_datadir.rs:7-11`) calls `tempfile::tempdir().unwrap().keep()` on **every single invocation**. Every caller that asks `EphemeralDir::get_data_dir()` receives a completely different temporary directory, and none of those directories are ever cleaned up from `/tmp`.

This RFC adds per-file async concurrency locks and atomic write-rename semantics to `LocalStorage`, replaces blocking `path.exists()` checks with `tokio::fs`, and fixes `EphemeralDir` so it returns a consistent shared directory per test/process and cleans up properly.

---

## 2. Executive Summary

- **Context:** `synapto/crates/core/src/storage/local_storage.rs` and `synapto/crates/test/src/ephemeral_datadir.rs`.
- **Motivation:** Prevent race-condition data loss and JSON corruption in `LocalStorage` and stop leaking `/tmp` directories on every `EphemeralDir::get_data_dir()` call.
- **Overarching Goals:** Thread-safe, atomic local file persistence and deterministic temporary directory lifecycle.
- **Purpose:** Synchronize concurrent `LocalStorage` operations per file path, use async filesystem checks, and memoize `EphemeralDir`.
- **Problem:**
  1. **Unsynchronized Read-Modify-Write Cycles (`synapto/crates/core/src/storage/local_storage.rs:58-82, 126-179, 184-213, 241-265, 514-557`):** `LocalStorage` has no `RwLock` or `Mutex` guarding file access. Concurrent `tokio::fs::read_to_string` + `tokio::fs::write` calls race and truncate files.
  2. **Blocking `path.exists()` on Async Executor (`local_storage.rs:59, 95, 128, 154, 189, 220, 243, 272, 328, 344, 521`):** Calls synchronous `std::path::Path::exists()` on Tokio worker threads.
  3. **`EphemeralDir::get_data_dir` Per-Call Tempdir Leak (`local_storage.rs:570-574` & `crates/test/src/ephemeral_datadir.rs:7-11`):**
     ```rust
     fn get_data_dir() -> PathBuf {
         tempfile::tempdir().unwrap().keep()
     }
     ```
     Creates and leaks a new `/tmp/.tmpXXXXXX` directory on every call.
- **Solution:**
  1. Add a shared lock registry `locks: Arc<std::sync::Mutex<HashMap<PathBuf, Arc<tokio::sync::RwLock<()>>>>>` to `LocalStoragePool` / `LocalStorage` so all read operations acquire a read guard and all read-modify-write operations acquire an exclusive write guard for the target `PathBuf`, writing via a temporary sibling file and `tokio::fs::rename` for crash/read atomicity.
  2. Replace `path.exists()` with `tokio::fs::try_exists(&path).await` (or handling `ErrorKind::NotFound` directly on `tokio::fs::read_to_string`).
  3. Memoize the temporary directory in `EphemeralDir` or scope it cleanly so all stores within a test instance share the same directory.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/crates/core/src/storage/local_storage.rs:58-82`** (`upsert_record`):
   ```rust
   let mut records: BTreeMap<String, serde_json::Value> = if path.exists() {
       let content = fs::read_to_string(&path).await.map_err(|e| e.to_string())?;
       if content.trim().is_empty() {
           BTreeMap::new()
       } else {
           serde_json::from_str(&content).map_err(|e| e.to_string())?
       }
   } else {
       BTreeMap::new()
   };
   let val = serde_json::to_value(data).map_err(|e| e.to_string())?;
   records.insert(id.to_string(), val);
   let serialized = serde_json::to_string_pretty(&records).map_err(|e| e.to_string())?;
   fs::write(&path, serialized).await.map_err(|e| e.to_string())?;
   ```
   Identical unsynchronized read-modify-write patterns exist in `delete_record` (`L126-150`), `trim_records_before` (`L152-179`), `set` (`L184-213`), `delete` (`L241-265`), and `delete_vectors` (`L514-557`).
2. **`synapto/crates/core/src/storage/local_storage.rs:570-574`** & **`synapto/crates/test/src/ephemeral_datadir.rs:7-11`**:
   ```rust
   pub struct EphemeralDir;
   impl DataDirProvider for EphemeralDir {
       fn get_data_dir() -> PathBuf {
           tempfile::tempdir().unwrap().keep()
       }
   }
   ```

---

## 4. Structural Hierarchy

- **Mission:** Concurrency-Safe Local Persistence and Deterministic Test Data Directories.
- **Goal:** Guarantee serializability of concurrent `LocalStorage` file mutations and stop leaking temporary directories.
- **Atomic Responsibilities:**
  - **`LocalStoragePool` (`crates/core/src/storage/local_storage.rs`):** Holds `file_locks: Arc<Mutex<HashMap<PathBuf, Arc<tokio::sync::RwLock<()>>>>>` shared across all `LocalStorage` namespace instances created from the pool.
  - **`EphemeralDir` (`crates/test/src/ephemeral_datadir.rs`):** Provides a consistent temporary directory path without leaking a new directory on each `get_data_dir()` call.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 24 (Strict Domain Registries / Shared Storage Pool):** All namespaces sharing a `LocalStoragePool` share the underlying `file_locks` table.
- **Robust by Design Protocol:** File updates must be synchronized and atomic.

### 5.2 Per-Path Lock Table
```rust
// synapto/crates/core/src/storage/local_storage.rs
#[derive(Clone, Default)]
pub struct LocalStorageLockTable {
    locks: Arc<std::sync::Mutex<HashMap<PathBuf, Arc<tokio::sync::RwLock<()>>>>>,
}

impl LocalStorageLockTable {
    pub fn lock_for(&self, path: &Path) -> Arc<tokio::sync::RwLock<()>> {
        let mut map = self.locks.lock().unwrap_or_else(|e| e.into_inner());
        map.entry(path.to_path_buf())
            .or_insert_with(|| Arc::new(tokio::sync::RwLock::new(())))
            .clone()
    }
}
```

### 5.3 JSON Contract Examples

#### Persisted Collection File (`records_sessions.json`)
```json
{
  "2026-10-17T08:00:00Z": {
    "started_at": "2026-10-17T08:00:00Z",
    "ended_at": "2026-10-17T08:15:00Z",
    "summary": "Atomic local storage write completed."
  }
}
```

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For `EphemeralDir::get_data_dir()` in `synapto/crates/test/src/ephemeral_datadir.rs:7-11`:

**[ Option A: Thread-Local / Process-Scoped `TempDir` with Automatic Drop Cleanup ]**
- **Action:** Store a `TempDir` inside a `OnceLock<TempDir>` (or thread-local `RefCell<Option<TempDir>>` with a reset helper for multi-test runners) and return `tempdir.path().to_path_buf()`, so repeated calls to `EphemeralDir::get_data_dir()` within a run return the **same** directory and the `TempDir` is cleaned up when dropped.
- **Pros:** Fixes the split-directory bug where `ConfigProvider`, `StorageConnection`, and plugins calling `EphemeralDir::get_data_dir()` each got a different directory, and avoids leaking directories via `.keep()`.
- **Cons:** Parallel tests in the same binary using `EphemeralDir` need thread-local isolation or unique subdirectories per `LocalStoragePool` connection.

**[ Option B: Allocate `TempDir` Once Inside `LocalStoragePool::connect` When Using `EphemeralDir` ]**
- **Action:** In `LocalStorage::connect`, call `D::get_data_dir()` exactly once to initialize `LocalStoragePool { base_dir: PathBuf, ... }` (which already happens once per bundle run at `local_storage.rs:28`), and store the `TempDir` guard in a `OnceLock` in `EphemeralDir` so multiple callers within the same process share one directory.
- **Pros:** Both `LocalStoragePool` and any other component calling `EphemeralDir::get_data_dir()` resolve to the same path.
- **Cons:** None.

**My Recommendation:** I recommend **Option A** (thread-local `TempDir` so parallel `#[tokio::test]` functions in the same test binary are isolated from each other while all calls within the same test share the same directory and clean up automatically).
Please select Option A or Option B for this RFC.
