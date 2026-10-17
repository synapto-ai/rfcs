# Retrospective: Optional Decision Subsystem and First-Class Decision Plugin Architecture

## Executive Summary
This retrospective documents the formal RFC lifecycle and implementation of the optional Decision Subsystem in Synapto. The work introduces domain-level typed judgments (`Choice`, `Noul`, `Score`), first-class decision provider plugins (`synapto-decision-typesafe`), and complete LLM replacement in `task-memory`.

## Iterative Evolution & Paradigm Shifts

### 1. Decision Engine Optionality: Setup-Time vs. Runtime Branching
- **Initial Options:** 
  - Option A: Setup-time strategy resolution via `DecisionHandle::is_available()`.
  - Option B: Universal null object with runtime branching at every call site.
- **Architectural Shift:** Lead Architect approved Option A. Consuming plugins inspect availability upfront and instantiate their execution strategy once. This eliminates runtime branching overhead in hot loops.

### 2. Bundle Ordering Race Condition Mitigation
- **Flaw Identified (Subagent Review 1):** In `synapto-core`, plugins in `PluginTuple` initialize sequentially. If a consumer plugin precedes a provider plugin in the tuple, the decision provider is not yet registered when `Plugin::create` executes.
- **Second Flaw Identified (Subagent Review 2):** Attempting two-phase generic compile-time sorting in `PluginTuple` failed because Rust does not support runtime type reflection or trait specialization over generic type parameters.
- **Resolution:** `DecisionHandle` uses an internal shared cell (`Arc<RwLock<Option<Arc<dyn RawDecisionExecutor>>>>`). `synapto-core` instantiates `DecisionHandle::empty()`. When a provider registers, it updates this handle. All plugins finish registration before `InteractionObserver::start` executes, ensuring 100% deterministic strategy selection independent of bundle tuple order.

### 3. Delimited Task Keys and Strong Typing (Rule 15)
- **Initial Idea:** Use simple string keys (`"{task_id}_trigger"`, `"{task_id}_cancel"`).
- **Subagent Finding:** Naive splitting on colons or underscores fails if a `TaskId` contains delimiters (e.g. URNs).
- **Resolution:** Implemented typed `TaskQuestionKey` and `TaskEvaluationKind`. Used `rsplit_once(':')` to split keys deterministically from the right.

### 4. Deterministic Conflict Precedence
- **Edge Case Identified:** High ambient noise or ambiguous user messages could cause both `trigger` and `cancel` probabilities to exceed the `0.75` threshold.
- **Resolution:** Explicit conflict precedence was codified: cancellation overrides activation.

## Attribution
- **RFC Protocol & Core Abstraction Design:** Main Coding Agent
- **Setup-Time Optionality Directive (Option A Selection):** Lead Architect
- **Bundle Ordering & Two-Phase Refinement:** Zero-Context Subagent
- **Compound Key & Missing Key Validation Rules:** Zero-Context Subagent

## Verification
- Unit test `test_serialization_request` passed in `synapto-decision-typesafe`.
- Unit tests `test_task_question_key_roundtrip`, `test_task_question_key_with_colons`, `test_generate_task_questions_with_trigger`, `test_generate_task_questions_without_trigger`, and `test_conflict_precedence_cancellation_overrides_activation` passed in `synapto-plugin-task-memory`.
- Workspace-wide compilation (`cargo check --workspace`) succeeded with zero errors.
