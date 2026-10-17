# [MVP] [DRAFT] RFC: LLM JSON Schema `$ref` Inlining and Boundary Sanitization

> **Classification:** `Flaw` (Corrupted Nested Tool Schemas & Misplaced Sanitization Boundary)
> **Draft Status:** Reproduced (`rfcs/reproduced/`) — Failing scenario test `rfc_2026_10_17_llm_schema_ref_inlining_and_boundary_sanitization` verified in `synapto/crates/test/tests/scenario_tests.rs`; awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-llm-schema-ref-inlining-and-boundary-sanitization.md`.

## 1. User-Centric Motivation & Goal

When a plugin defines a tool or structured output whose Rust struct contains nested structs or enums, `schemars` generates a JSON Schema using `"$defs"` (or `"definitions"`) and `"$ref": "#/$defs/MySubType"`.

Currently, `clean_json_schema_for_llm` (`synapto/crates/core/src/cognitive/types.rs:503-591`) simply deletes the `"$defs"` and `"$ref"` keys from the JSON object without inlining the referenced sub-schema first! As a result, any nested struct or enum field in a tool argument schema is reduced to an empty `{}` object when sent to the LLM, causing the LLM to hallucinate invalid arguments. Additionally, `clean_json_schema_for_llm` is private to `crates/core/src/cognitive/types.rs` and is only called on dynamic tools, leaving static tools (`LLMClient::create_client_with_tools`) and structured output schemas (`synapto-llm/src/lib.rs:388-393`) unsanitized.

This RFC fixes schema sanitization to recursively inline `#/$defs/*` and `#/definitions/*` references before stripping unsupported keywords, and moves sanitization to `synapto-llm` so every schema crossing the LLM boundary is sanitized uniformly (Rule 25).

---

## 2. Executive Summary

- **Context:** `synapto/crates/core/src/cognitive/types.rs:492, 503-591`, `crates/core/src/utils/schema.rs:56-162`, and `synapto/crates/llm/src/lib.rs:388-393`.
- **Motivation:** Deleting `"$defs"` and `"$ref"` without inlining corrupts nested tool schemas, and placing sanitization in `crates/core` violates Rule 25 (`synapto-llm` boundary responsibility).
- **Overarching Goals:** Loss-free JSON schema normalization at the `synapto-llm` compilation boundary for all static tools, dynamic tools, and structured output schemas.
- **Purpose:** Implement `$defs`/`$ref` inlining prior to keyword stripping and relocate `clean_json_schema_for_llm` into `synapto-llm`.
- **Problem:**
  1. **Destructive `$defs`/`$ref` Deletion (`synapto/crates/core/src/cognitive/types.rs:507-508`):**
     ```rust
     map.remove("$defs");
     map.remove("$ref");
     ```
     Deleting `"$ref"` without replacing the node with its target definition from `"$defs"` leaves `{}` in place of the nested type's schema.
  2. **Incomplete Boundary Coverage (`synapto/crates/llm/src/lib.rs:388-393` vs `crates/core/src/cognitive/types.rs:492`):** Static tools registered via `create_client_with_tools` and structured `output_schema`s are not passed through `clean_json_schema_for_llm`.
  3. **`serde_saphyr::to_string` `unreachable!` Misuse (`synapto/crates/llm/src/lib.rs:422-424`):** Uses `unreachable!` on YAML serialization of input schemas.
- **Solution:**
  1. Move `clean_json_schema_for_llm` to `synapto_llm::schema::sanitize_json_schema`.
  2. Before removing `"$defs"` / `"definitions"`, extract the definitions map and recursively resolve any `{"$ref": "#/$defs/<Name>"}` or `{"$ref": "#/definitions/<Name>"}` by cloning the target definition in-place (with a visited-set cycle guard for recursive types).
  3. Call `sanitize_json_schema` inside `synapto-llm` on all static tools, dynamic tools, and output schemas.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/crates/core/src/cognitive/types.rs:503-591`**:
   ```rust
   fn clean_json_schema_for_llm(value: &mut serde_json::Value) {
       if let serde_json::Value::Object(map) = value {
           map.remove("$schema");
           map.remove("additionalProperties");
           map.remove("$defs");
           map.remove("$ref");
           ...
   ```
2. **`synapto/crates/core/src/cognitive/types.rs:491-493`**:
   ```rust
   let mut schema_val = serde_json::to_value(tool.parameters_schema()).unwrap_or_default();
   clean_json_schema_for_llm(&mut schema_val);
   ```
3. **`synapto/crates/llm/src/lib.rs:388-394`**:
   `create_client_with_tools` converts `tools` and `output_schema` without calling `clean_json_schema_for_llm`.

---

## 4. Structural Hierarchy

- **Mission:** Loss-Free Schema Normalization at the Core LLM Boundary (Rule 25).
- **Goal:** Ensure all nested Rust types and third-party MCP schemas are valid, self-contained JSON Schemas before reaching any `LlmProvider`.
- **Atomic Responsibilities:**
  - **`synapto_llm::schema::sanitize_json_schema` (`crates/llm/src/lib.rs`):** Extracts `$defs`/`definitions`, inlines `#/$defs/*` and `#/definitions/*` with cycle protection, normalizes `"const"` and nullable type arrays, and strips unsupported meta-keywords (`$schema`, `additionalProperties`, `$defs`, `definitions`).

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 25 (Core Boundary Resilience & Schema Sanitization):** All tool definitions and schemas must be sanitized at the core LLM boundary before being forwarded to underlying LLM providers.

### 5.2 Reference Inlining Algorithm Signature
```rust
// synapto/crates/llm/src/schema.rs
pub fn sanitize_json_schema(root: &mut serde_json::Value);
```

### 5.3 JSON Contract Examples

#### Before Sanitization (`schemars` Output with `$defs` and `$ref`)
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "filter": { "$ref": "#/$defs/SearchFilter" }
  },
  "required": ["filter"],
  "$defs": {
    "SearchFilter": {
      "type": "object",
      "properties": {
        "tag": { "type": "string" }
      },
      "required": ["tag"],
      "additionalProperties": false
    }
  }
}
```

#### After Sanitization (Inlined & Cleaned for LLM Provider)
```json
{
  "type": "object",
  "properties": {
    "filter": {
      "type": "object",
      "properties": {
        "tag": { "type": "string" }
      },
      "required": ["tag"]
    }
  },
  "required": ["filter"]
}
```

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For handling self-referential (recursive) `$ref` cycles during inlining:

**[ Option A: Visited-Ref Ancestor Set with `{ "type": "object" }` Cycle Fallback ]**
- **Action:** Track active `$ref` names along the current recursion stack (`HashSet<String>`). Non-recursive shared types (referenced in multiple sibling fields) are inlined at every site; if a true cycle (`A -> B -> A`) is encountered, replace the cyclic back-edge with `{"type": "object"}`.
- **Pros:** Handles both shared sub-structs (inlining them completely) and recursive types (preventing infinite recursion) deterministically in ~30 lines of code.
- **Cons:** Recursive types terminate at depth 1 of the cycle.

**[ Option B: Reject Recursive Tool Schemas with an Explicit Error ]**
- **Action:** Return an `Err` if a cyclic `$ref` is detected while inlining.
- **Pros:** Fails fast on recursive schemas.
- **Cons:** Could reject an external MCP server tool that uses a recursive JSON schema definition.

**My Recommendation:** I recommend **Option A** because external MCP servers can define recursive JSON schemas, and Rule 25 mandates that the core boundary sanitize external tool schemas resiliently without crashing.
Please select Option A or Option B for this RFC.
