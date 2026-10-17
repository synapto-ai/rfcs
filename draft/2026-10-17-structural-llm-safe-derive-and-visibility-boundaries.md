# [MVP] [DRAFT] RFC: Structural `#[derive(LLMSafe)]` Field Verification and `pub(crate)` Visibility Boundaries

> **Classification:** `Guideline` (Rule 6 & Section 6 `LLMSafe` Boundary Enforcement)
> **Draft Status:** Phase 2 — Awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply the selected fork option (Option A or Option B), draft Sections 6–10 (Phases 3–5), run the zero-context subagent review, and present to the Lead Architect for Phase 6 approval to move to `approved/2026-10-17-structural-llm-safe-derive-and-visibility-boundaries.md`.

## 1. User-Centric Motivation & Goal

To prevent internal system state, database metadata, or sensitive fields from accidentally being serialized into LLM prompts, Synapto uses the `LLMSafe` marker trait and `#[derive(LLMSafe)]` procedural macro.

Currently, `#[derive(LLMSafe)]` (`synapto/crates/derive/src/lib.rs:14-20`) generates `impl LLMSafe for Type {}` unconditionally without checking whether the struct's fields implement `LLMSafe`. A developer can embed a non-`LLMSafe` internal struct inside an `#[derive(LLMSafe)]` prompt struct and it compiles without error. Additionally, `LLMSafe` is not implemented for standard Rust primitive types (`String`, `bool`, integers, floats), internal storage models `Progression` and `Continuum` implement `LLMSafe` directly instead of using view DTOs, and `LLMSafe` view structs across `crates/core` and memory plugins are `pub` instead of `pub(crate)` (violating `ARCHITECTURE.md:158`).

This RFC updates `#[derive(LLMSafe)]` to enforce compile-time `FieldType: LLMSafe` bounds on all nested fields, implements `LLMSafe` for standard primitives, introduces dedicated `LLMSafe` view structs for `Progression` and `Continuum`, and restricts `LLMSafe` view models to `pub(crate)`.

---

## 2. Executive Summary

- **Context:** `synapto/crates/derive/src/lib.rs`, `synapto/crates/interface/src/llm.rs`, `synapto/crates/core/src/cognitive/types.rs`, and memory/context plugins (`episodic-memory`, `behavioral-memory`, `semantic-memory`, `task-memory`, `skills`, `clock`).
- **Motivation:** Enforce Rule 6 and Section 6 (`LLMSafe` Boundaries) at compile time across the entire type hierarchy.
- **Overarching Goals:** Guarantee that no type can derive `LLMSafe` unless every transitive field in its structure is also `LLMSafe`, and keep all `LLMSafe` view models internal (`pub(crate)`) to their crate.
- **Purpose:** Add field-level trait bounds to `#[derive(LLMSafe)]`, implement `LLMSafe` for primitives, separate storage structs from `LLMSafe` DTOs in `episodic-memory`, and enforce `pub(crate)` visibility on `LLMSafe` structs.
- **Problem:**
  1. **Unchecked `#[derive(LLMSafe)]` (`synapto/crates/derive/src/lib.rs:14-20`):** Does not generate `where FieldType: LLMSafe` bounds for struct or enum fields.
  2. **Missing Primitive `LLMSafe` Impls (`synapto/crates/interface/src/llm.rs:3-13`):** Only implements `LLMSafe` for `Vec`, `Option`, `Box`, `&T`, `[T]`, `[T; N]`, and `HashMap`, omitting `String`, `str`, `bool`, `i8..i128`, `u8..u128`, `isize`, `usize`, `f32`, `f64`, `BTreeMap`.
  3. **Storage Models Directly Implementing `LLMSafe` (`synapto/integrations/plugins/episodic-memory/src/{progression.rs:41, continuum.rs:41}`):** `Progression` and `Continuum` implement `LLMSafe` directly instead of mapping to dedicated `LLMProgression` / `LLMContinuum` view structs (unlike `Session` -> `LLMSession` in `session.rs:38`).
  4. **`pub` Visibility on Internal `LLMSafe` Structs (`ARCHITECTURE.md:158` Violation):**
     - `synapto/crates/core/src/cognitive/types.rs:203, 231, 337, 352` (`LLMUser`, `LLMUserMessage`, `CognitiveLLMInteractionMemory`, `CognitiveLLMContent`)
     - `synapto/integrations/plugins/behavioral-memory/src/lib.rs:60, 126, 133`
     - `synapto/integrations/plugins/episodic-memory/src/{session.rs:38, progression.rs:45, continuum.rs:45}`
     - `synapto/integrations/plugins/semantic-memory/src/lib.rs:84, 152, 166, 226, 240, 261, 267`
     - `synapto/integrations/plugins/task-memory/src/types.rs:36, 48, 59, 68`
     - `synapto/integrations/plugins/skills/src/{context.rs:12, 19, tool.rs:15}`
     - `synapto/integrations/plugins/clock/src/lib.rs:16`
- **Solution:**
  1. Implement `LLMSafe` in `synapto/crates/interface/src/llm.rs` for `String`, `str`, `bool`, `i8`, `i16`, `i32`, `i64`, `i128`, `isize`, `u8`, `u16`, `u32`, `u64`, `u128`, `usize`, `f32`, `f64`, `BTreeMap<K, V>`, and `synapto_interface::interaction::Timestamp`.
  2. Update `impl_llm_safe_macro` in `synapto/crates/derive/src/lib.rs` to emit `where FieldTy: synapto_interface::llm::LLMSafe` bounds for every field in the struct or enum.
  3. Define `pub(crate) struct LLMProgression` and `pub(crate) struct LLMContinuum` in `episodic-memory` and remove `impl LLMSafe for Progression` and `impl LLMSafe for Continuum`.
  4. Restrict `LLMSafe` view structs across `crates/core` and plugins to `pub(crate)`.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/crates/derive/src/lib.rs:14-20`**:
   ```rust
   fn impl_llm_safe_macro(ast: &syn::DeriveInput) -> TokenStream {
       let name = &ast.ident;
       let (impl_generics, ty_generics, where_clause) = ast.generics.split_for_impl();
       let generate = quote! {
           impl #impl_generics synapto_interface::llm::LLMSafe for #name #ty_generics #where_clause {}
       };
       generate.into()
   }
   ```
2. **`synapto/crates/interface/src/llm.rs:3-13`**:
   ```rust
   pub trait LLMSafe {}
   impl<T: LLMSafe> LLMSafe for Vec<T> {}
   impl<T: LLMSafe> LLMSafe for Option<T> {}
   impl<T: LLMSafe> LLMSafe for Box<T> {}
   impl<T: LLMSafe + ?Sized> LLMSafe for &T {}
   impl<T: LLMSafe> LLMSafe for [T] {}
   impl<T: LLMSafe, const N: usize> LLMSafe for [T; N] {}
   impl<K, V: LLMSafe, S> LLMSafe for std::collections::HashMap<K, V, S> {}
   ```
3. **`synapto/integrations/plugins/episodic-memory/src/{progression.rs:41, continuum.rs:41}`**:
   ```rust
   impl synapto_interface::llm::LLMSafe for Progression {}
   impl synapto_interface::llm::LLMSafe for Continuum {}
   ```

---

## 4. Structural Hierarchy

- **Mission:** Compile-Time Verified `LLMSafe` Type Trees and Crate-Private View Models.
- **Goal:** Prevent accidental LLM serialization of internal types and enforce `pub(crate)` encapsulation on all `LLMSafe` structs.
- **Atomic Responsibilities:**
  - **`synapto-derive` (`crates/derive/src/lib.rs`):** Extracts all field types from `syn::Data::Struct` and `syn::Data::Enum` and appends `#field_ty: synapto_interface::llm::LLMSafe` predicates to the generated `where` clause.
  - **`synapto_interface::llm::LLMSafe` (`crates/interface/src/llm.rs`):** Implements `LLMSafe` for standard scalar primitives and collections where keys/values are `LLMSafe`.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 6 & Section 6 (`LLMSafe` Boundaries, `ARCHITECTURE.md:24, 158`):** Internal state types must not implement `LLMSafe`; only dedicated `pub(crate)` view models whose fields are all `LLMSafe` may implement `LLMSafe`.

### 5.2 Generated `#[derive(LLMSafe)]` Expansion
```rust
// For:
#[derive(LLMSafe)]
pub(crate) struct ClockContext {
    pub current_time_utc: String,
}
// Expands to:
impl synapto_interface::llm::LLMSafe for ClockContext
where
    String: synapto_interface::llm::LLMSafe,
{}
```

### 5.3 JSON Contract Examples
_Not applicable (Compile-time trait bound verification and Rust visibility modifiers; serialized JSON payloads are unchanged)._

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

For enforcing `LLMSafe` on map keys in `HashMap<K, V>` and `BTreeMap<K, V>` (`synapto/crates/interface/src/llm.rs:13`):

**[ Option A: Require Both `K: LLMSafe` and `V: LLMSafe` on `HashMap` and `BTreeMap` ]**
- **Action:** Change `impl<K, V: LLMSafe, S> LLMSafe for HashMap<K, V, S>` to `impl<K: LLMSafe, V: LLMSafe, S> LLMSafe for HashMap<K, V, S>` (and `BTreeMap<K: LLMSafe, V: LLMSafe>`).
- **Pros:** Prevents an unverified internal key type `K` from leaking into the serialized JSON object keys of an LLM prompt.
- **Cons:** Custom newtype keys used in `LLMSafe` maps must derive `LLMSafe`.

**[ Option B: Only Require `V: LLMSafe` on Map Values ]**
- **Action:** Keep `K` unconstrained on `HashMap<K, V>`.
- **Pros:** Slightly fewer `#[derive(LLMSafe)]` annotations on key newtypes.
- **Cons:** Allows non-`LLMSafe` key types to be serialized into prompts.

**My Recommendation:** I recommend **Option A** (`K: LLMSafe, V: LLMSafe`) for complete structural safety across both map keys and values.
Please select Option A or Option B for this RFC.
