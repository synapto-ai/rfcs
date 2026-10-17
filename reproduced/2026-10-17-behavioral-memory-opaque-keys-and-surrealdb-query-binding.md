# [MVP] [DRAFT] RFC: Behavioral Memory Opaque Record Keys and SurrealDB Parameterized Vector Queries

> **Classification:** `Flaw` (Storage Path Corruption & Database Query Injection)
> **Draft Status:** Reproduced (`rfcs/reproduced/`) — Failing scenario test `rfc_2026_10_17_behavioral_memory_opaque_keys_and_surrealdb_query_binding` verified in `synapto/crates/test/tests/scenario_tests.rs`; awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-behavioral-memory-opaque-keys-and-surrealdb-query-binding.md`.

## 1. User-Centric Motivation & Goal

When the assistant stores behavioral insights or vector embeddings in its configured database (Firestore or SurrealDB), special characters in LLM-generated text or filter metadata must never corrupt database document paths or enable query injection.

Currently, `synapto-plugin-behavioral-memory` uses the full LLM-generated insight sentence (`&new_insight.content`) as the primary key in `store.upsert_record("insights", &new_insight.content, ...)`. In Firestore (`synapto-storage-firestore`), document IDs cannot contain forward slashes (`/`) or exceed key byte limits; an insight such as `"Use CI/CD pipelines"` breaks Firestore path resolution. Meanwhile, in `synapto-storage-surrealdb`, `VectorStore` interpolates `collection`, `dimension`, and `filter_field` directly into SurrealQL strings using `format!`, creating a query injection vulnerability.

This RFC replaces raw sentence keys in `behavioral-memory` with safe deterministic keys and enforces strict identifier validation and parameter binding in `synapto-storage-surrealdb`.

---

## 2. Executive Summary

- **Context:** `synapto/integrations/plugins/behavioral-memory/src/lib.rs` and `synapto/integrations/storage-providers/surrealdb/src/lib.rs`.
- **Motivation:** Prevent Firestore path errors caused by natural-language document keys and eliminate SurrealQL string interpolation hazards in `VectorStore`.
- **Overarching Goals:** Guarantee storage key safety across all `RecordStore` consumers and query parameterization across all `VectorStore` implementations.
- **Purpose:** Generate safe opaque document keys in `behavioral-memory` and parameterize/validate identifiers in `synapto-storage-surrealdb`.
- **Problem:**
  1. **Raw Sentence Used as Primary Key (`synapto/integrations/plugins/behavioral-memory/src/lib.rs:296`):** `store.upsert_record("insights", &new_insight.content, new_insight.clone()).await` passes arbitrary LLM output text as the `id` argument to `RecordStore::upsert_record`.
  2. **SurrealQL String Interpolation in `VectorStore` (`synapto/integrations/storage-providers/surrealdb/src/lib.rs:265-336`):**
     - `setup_collection` (`L266-270`): Interpolates `{collection}` and `{dimension}` via `format!`.
     - `insert_vectors` (`L284`): Interpolates `UPSERT type::thing('{collection}', $id)` instead of `type::thing($table, $id)`.
     - `search_vectors` (`L307-309`): Interpolates `FROM {collection} WHERE {filter_clause}` where `filter_clause` interpolates `metadata.{filter_field} = $filter_val`.
     - `delete_vectors` (`L327-329`): Interpolates `DELETE {collection} WHERE metadata.{filter_field} = $filter_val;`.
- **Solution:**
  1. In `behavioral-memory/src/lib.rs:296`, derive a deterministic fixed-length hex key (`insight_<16-hex>`) from `new_insight.content` before calling `store.upsert_record`.
  2. In `surrealdb/src/lib.rs:265-336`, bind `$table` via `type::table($table)` and `type::thing($table, $id)` (matching `RecordStore` at `L127, 169`) and validate that `collection` and `filter_field` contain strictly ASCII alphanumeric or underscore characters (`[a-zA-Z0-9_]`).

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/integrations/plugins/behavioral-memory/src/lib.rs:293-302`**:
   ```rust
   for new_insight in new_insights {
       tracing::info!("Adding new behavioral insight: {:?}", new_insight);
       if let Err(e) = store
           .upsert_record("insights", &new_insight.content, new_insight.clone())
           .await
       {
           tracing::error!("Failed to write behavioral insight: {:?} at line {}", e, line!());
       }
       insight_memory.push(new_insight);
   }
   ```
2. **`synapto/integrations/storage-providers/surrealdb/src/lib.rs:265-336`**:
   ```rust
   // L284:
   let query = format!("UPSERT type::thing('{collection}', $id) CONTENT $data;");
   // L299-309:
   let filter_clause = if let Some((filter_field, _)) = &filter {
       format!("metadata.{filter_field} = $filter_val AND ")
   } else {
       String::new()
   };
   let sql = format!(
       "SELECT meta::id(id) AS id, vector::similarity::cosine(embedding, $query) AS score, metadata FROM {collection} WHERE {filter_clause} embedding <|10|> $query ORDER BY score DESC LIMIT $top_k;"
   );
   // L327-329:
   let sql = format!("DELETE {collection} WHERE metadata.{filter_field} = $filter_val;");
   ```

---

## 4. Structural Hierarchy

- **Mission:** Safe Storage Primary Keys and Parameterized Database Queries.
- **Goal:** Eliminate illegal document key characters in `behavioral-memory` and SQL injection in `surrealdb`.
- **Atomic Responsibilities:**
  - **`behavioral_insight_key(content: &str) -> String` (`behavioral-memory/src/lib.rs`):** Produces a deterministic ASCII hex identifier for `RecordStore::upsert_record`.
  - **`validate_identifier(ident: &str) -> Result<(), String>` (`storage-providers/surrealdb/src/lib.rs`):** Rejects any collection or metadata field name containing characters outside `[a-zA-Z0-9_]` and uses `type::table($table)` / `type::thing($table, $id)` bindings.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 17 (Opaque Resource Identifier Protocol):** Database primary keys must be opaque, URL/path-safe identifiers rather than raw user or LLM sentences.
- **Pragmatic Security by Default Protocol:** Never interpolate unvalidated identifiers into database queries.

### 5.2 Deterministic Key & Identifier Validation
```rust
// synapto/integrations/storage-providers/surrealdb/src/lib.rs
fn validate_sql_identifier(name: &str) -> Result<(), String> {
    if !name.is_empty() && name.chars().all(|c| c.is_ascii_alphanumeric() || c == '_') {
        Ok(())
    } else {
        Err(format!("Invalid SQL identifier: {name}"))
    }
}
```

### 5.3 JSON Contract Examples

#### Compiled Prompt Context Input (`context_plugins.behavioral_memory`)
```json
{
  "context_plugins": {
    "behavioral_memory": {
      "insights": [
        "Always include exact file paths and line numbers in code reviews."
      ]
    }
  }
}
```

#### Behavioral Insight Extraction LLM Output (`BehavioralInsightLLMOutput`)
```json
{
  "reasoning": "User stated a clear communication rule.",
  "new_insights": [
    "Always include exact file paths and line numbers in code reviews."
  ]
}
```

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For generating the record key in `synapto/integrations/plugins/behavioral-memory/src/lib.rs:296`:

**[ Option A: Deterministic FNV-1a / SHA-256 Hex Hash of `new_insight.content` (`insight_<16-hex>`) ]**
- **Action:** Compute a deterministic 64-bit FNV-1a (or SHA-256) hex digest of `new_insight.content` (`format!("insight_{hash:016x}")`) as the `RecordStore` key while keeping `BehavioralInsight` schema unchanged.
- **Pros:** Zero external crate dependencies (64-bit FNV-1a is 8 lines of pure Rust), idempotent across identical insights, safe for Firestore and SurrealDB, and requires no schema change to `BehavioralInsight`.
- **Cons:** Key order in `get_ordered_records` is hash-ordered unless sorted by `updated_at` in memory after loading.

**[ Option B: Timestamp-Prefixed Deterministic Hash Key (`{updated_at}_{hash:016x}`) ]**
- **Action:** Format the key as `format!("{}_{:016x}", new_insight.updated_at, fnv1a_64(&new_insight.content))`.
- **Pros:** Preserves chronological ordering in `get_ordered_records("insights", ...)` and `trim_records_before` while remaining 100% path-safe in Firestore and SurrealDB.
- **Cons:** Slightly longer key string.

**My Recommendation:** I recommend **Option B** (`{updated_at}_{hash:016x}`) because `get_ordered_records` and `trim_records_before` rely on chronological key ordering across all memory plugins.
Please select Option A or Option B for this RFC.
