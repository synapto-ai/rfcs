# [MVP] [DRAFT] RFC: Unified Function Calling for Speech, Chat, Commands, and Tools

> **Classification:** `Feature` (Cognitive Loop I/O Unification & Simultaneous Speech/Text + Tool Execution)
> **Draft Status:** Phase 2 — Awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
> **Next Step on Resume:** Apply selected fork option, complete Sections 6–10 (Phases 3–5), run zero-context subagent review, and present for Phase 6 approval.

## 1. User-Centric Motivation & Goal

When a user speaks to or messages the AI with a request that requires calling an external tool (such as reading a document or looking up information), a natural assistant should be able to **acknowledge the user immediately** (`say` or `write`, e.g., *"Let me check that document for you..."*) **and trigger the tool in the exact same turn**. Conversely, when a user asks a simple question that needs no external tool, the AI should simply reply (`say` or `write`) without being pressured by conflicting schema constraints into calling unrelated tools.

Today, `say`, `write`, and `Command` plugins are embedded inside a mandatory JSON response schema (`CognitiveLLMOutput` via `generationConfig.responseJsonSchema`), while `Tool` plugins are passed as native LLM `functionDeclarations`. Because LLM providers such as Google Gemini treat `responseJsonSchema` (structured text generation) and `functionCall` parts as mutually exclusive within a single turn, the AI **cannot** speak/write and call a tool in the same turn, and the competing decoder modes cause spurious tool calls on trivial turns (such as `multi_assert_test`).

This RFC unifies `say`, `write`, `Command` plugins, and `Tool` plugins under native LLM Function Calling in the cognitive loops, distinguishing immediate **Effect Commands** (`say`, `write`, `Command` — which execute immediately without re-triggering the cognitive loop) from asynchronous **Query Tools** (`Tool` — which execute in the background and wake the cognitive loop via `ResolvedToolBatch`).

---

## 2. Executive Summary

- **Context:** `synapto` (`crates/core/src/cognitive/{direct.rs, side.rs, processor.rs, types.rs}` and `crates/core/src/utils/schema.rs`) and `synapto-llm` (`crates/llm/src/lib.rs`) currently split LLM outputs into two competing mechanisms on the same request: `output_schema` (`CognitiveLLMOutput<CognitiveDirectCommands>` / `CognitiveLLMOutput<CognitiveSideCommands>`) and `tools` (`Vec<genai::chat::Tool>`).
- **Motivation:** Combining `generationConfig.responseJsonSchema` (`responseMimeType: "application/json"`) with `tools: [{"functionDeclarations": [...]}]` in `GoogleLlmExecutor::execute_raw` (`synapto/integrations/llm-providers/google/src/lib.rs:136-147`) prevents simultaneous `say`/`write` + `Tool` execution in a single turn and causes decoder contention where the model invokes unrelated zero-argument tools (`mock_lookup_key`) before emitting a structured text response.
- **Overarching Goals:** Enable single-turn simultaneous communication (`say`/`write`), fire-and-forget commands (`Command`), and asynchronous data retrieval (`Tool`) in `cognitive_direct_task` and `cognitive_side_task` without `responseJsonSchema` vs. `functionDeclarations` contention.
- **Purpose:** Replace the `CognitiveLLMOutput` JSON response schema on the tool-enabled cognitive loops with native function declarations for `say`, `write`, and registered `Command` plugins alongside `Tool` plugins, and partition returned `ToolCall`s into immediate Effect Commands vs. asynchronous Query Tools.
- **Problem:**
  1. **Mutually Exclusive `responseJsonSchema` and `functionCall` Modes (`synapto/integrations/llm-providers/google/src/lib.rs:136-147` & `synapto/crates/llm/src/lib.rs:250, 274-291`):** When `options.output_schema` and `options.tools` are both sent to Gemini, a `functionCall` turn produces zero text parts (`response_obj.texts()` is empty), so `LLMResult::Interrupted` always has `None` for `CognitiveLLMOutput`. The model can never `say` or `write` in the same turn that it calls a `Tool`.
  2. **Decoder Contention & Spurious Tool Invocation (`synapto/crates/core/src/cognitive/side.rs:26-38, 154-174` & `direct.rs:28-41, 225-245`):** Presenting `responseJsonSchema` (`CognitiveLLMOutput`) simultaneously with `functionDeclarations` causes Gemini to occasionally invoke zero-argument tools (`mock_lookup_key`) on simple conversational prompts (`multi_assert_test`).
  3. **Missing `cognitive_written` in `CognitiveLLMInteraction` (`synapto/crates/core/src/cognitive/types.rs:267-313`):** `Interaction` (`crates/core/src/interactions/recent.rs:24-35`) stores `cognitive_written: Option<CognitiveWritten>`, but `CognitiveLLMInteraction` omits it, hiding all prior chat replies from `interaction_memory`.
- **Solution:**
  1. Expose `say` (in `direct`), `write` (in `direct` and `side` when chat is wired), and registered `ErasedCommand` plugins (`registries.commands`) as native `genai::chat::Tool` function declarations alongside `ErasedTool` plugins (`registries.tools`).
  2. Omit `output_schema` (`responseJsonSchema`) on cognitive calls that operate via unified function calling (`WithTools`), while preserving `output_schema` on `WithoutTools` clients (used by background memory distillation, semantic, episodic, and task-memory evaluators).
  3. Partition LLM `ToolCall`s in the cognitive loop into:
     - **Effect Commands (`say`, `write`, `registries.commands`):** Executed immediately during turn output processing; never spawn background tasks on `tool_resolved_tx` and never increment `consecutive_tool_turns`.
     - **Query Tools (`registries.tools`):** Recorded in `in_flight_tools`, increment `consecutive_tool_turns`, and dispatched via `llm_client.execute_tools(request, query_tool_calls).await` to resolve as a `ResolvedToolBatch` on `tool_resolved_tx`.
  4. Add `cognitive_written: Option<CognitiveWritten>` to `CognitiveLLMInteraction` so the model has complete visibility of its past written responses in `interaction_memory`.

---

## 3. Current State Analysis (Exact File & Line Inventory)

1. **`synapto/crates/core/src/cognitive/direct.rs:28-41, 223-245`**:
   `CognitiveDirectCommands` defines `say: Option<CognitiveOutputSpeech>`, `write: Option<CognitiveOutputText>`, and `commands_map: BTreeMap<String, Value>`, which are compiled into `custom_output_schema` via `customize_cognitive_commands_schema` (`crates/core/src/utils/schema.rs:56-162`) and attached to `llm_client` alongside dynamic tools.
   ```rust
   struct CognitiveDirectCommands {
       #[serde(skip_serializing_if = "Option::is_none")]
       say: Option<CognitiveOutputSpeech>,
       #[serde(skip_serializing_if = "Option::is_none")]
       write: Option<CognitiveOutputText>,
       #[serde(flatten)]
       commands_map: std::collections::BTreeMap<String, serde_json::Value>,
   }
   ```
2. **`synapto/crates/core/src/cognitive/side.rs:26-38, 152-174`**:
   `CognitiveSideCommands` defines `write: Option<CognitiveOutputText>` and `commands_map: BTreeMap<String, Value>`, also compiled into `custom_output_schema` on `llm_client`.
3. **`synapto/crates/core/src/cognitive/types.rs:265-314, 437-457`**:
   - Lines 265–314 define `CognitiveLLMInteraction` with `cognitive_spoken: Option<CognitiveSpoken>` and `cognitive_reasoning: Option<CognitiveReasoning>`, but omit `cognitive_written: Option<CognitiveWritten>`.
   - Lines 437–446 define `CognitiveLLMOutput<CognitiveCommands>`:
     ```rust
     pub(super) struct CognitiveLLMOutput<CognitiveCommands> {
         pub commands: CognitiveCommands,
         pub reasoning: CognitiveReasoning,
         pub users_messages_evaluation: UsersMessagesEvaluation,
     }
     ```
4. **`synapto/crates/llm/src/lib.rs:59-130, 240-298`**:
   `LLMClient::call_inner` always sets `output_schema: Some(self.output_schema.clone())` in `RawLlmOptions` (line 250), even when `WithTools` is used. When `response_obj.tool_calls()` is non-empty (line 261), it attempts to parse `response_obj.texts().join("")` as `Output` (lines 274–289) and returns `Tools::interrupted(parsed_output, tool_calls_cloned)`.
5. **`synapto/integrations/llm-providers/google/src/lib.rs:136-147`**:
   ```rust
   if let Some(ref output_schema) = options.output_schema {
       chat_options =
           chat_options.with_response_format(genai::chat::ChatResponseFormat::JsonSpec(
               genai::chat::JsonSpec::new("schema", output_schema.clone()),
           ));
   }

   if let Some(tools) = options.tools
       && !tools.is_empty()
   {
       chat_req = chat_req.with_tools(tools);
   }
   ```
   When both `JsonSpec` and `with_tools` are sent to Gemini, Gemini never emits structured JSON `text` and `functionCall` parts in the same turn.

---

## 4. Structural Hierarchy

- **Mission:** Unified Native Function Calling for Cognitive Loops with Simultaneous Effect + Query Execution.
- **Goal:** Allow the cognitive loops (`direct` and `side`) to invoke `say`, `write`, `Command` plugins, and `Tool` plugins in a single LLM turn without `responseJsonSchema` contention, while keeping `WithoutTools` structured-output clients unchanged.
- **Atomic Responsibilities:**
  - **Effect Function Declarations (`say`, `write`, `ErasedCommand`):** Represent outbound side effects that execute immediately in the current turn (`DirectOutputProcessor` / `SideOutputProcessor`) and do not emit a `ToolResponse` or wake `tool_resolved_rx`.
  - **Query Tool Declarations (`ErasedTool`):** Represent asynchronous data-fetching or external operations that return a `ToolOutput`, append to `WorkingMemoryStore`, and wake the cognitive loop via `ResolvedToolBatch` on `tool_resolved_tx`.
  - **Cognitive Turn Dispatcher (`direct.rs`, `side.rs`, `processor.rs`):** Partitions the turn's `Vec<ToolCall>` into Effect Calls and Query Tool Calls; executes Effect Calls immediately, records the `Interaction` (including `cognitive_spoken`, `cognitive_written`, and `in_flight_tools` for Query Tool Calls), and dispatches only Query Tool Calls to `llm_client.execute_tools(request, query_tool_calls).await`.

---

## 5. Data Models

### 5.1 Governing Architectural Rules
- **Rule 3 & Rule 4 (`synapto/docs/src/ARCHITECTURE.md` — Strongly-Typed LLM Interfaces & Native Tool Calling):** All effect commands (`say`, `write`, `Command`) and query tools (`Tool`) expose strongly-typed `schemars::JsonSchema` argument structs sanitized via `clean_json_schema_for_llm`.
- **Rule 5 (`synapto/docs/src/ARCHITECTURE.md` — Generic Tool Architecture):** `LLMClient` remains domain-agnostic; `cognitive` owns the partitioning between Effect Commands and Query Tools.
- **Rule 6 (`synapto/docs/src/ARCHITECTURE.md` — Strict LLM I/O Boundaries `!LLMSafe`):** `CognitiveLLMInteraction` and all tool argument structs enforce `LLMSafe` boundaries.
- **Rule 25 (`synapto/docs/src/ARCHITECTURE.md` — Core Boundary Resilience & Schema Sanitization):** Built-in `say`/`write` schemas, `Command` schemas, and `Tool` schemas pass through `clean_json_schema_for_llm` before being attached to `genai::chat::Tool`.

### 5.2 Built-In Effect Tool Schemas & Updated `CognitiveLLMInteraction`

```rust
// synapto/crates/core/src/cognitive/types.rs
#[derive(Serialize, Deserialize, JsonSchema, PartialEq, Eq, Debug, Clone)]
#[serde(rename = "Interaction")]
struct CognitiveLLMInteraction {
    user_messages: Vec<LLMUserMessage>,

    #[schemars(
        description = "The communication channel context for this interaction. Use this for the 'write' tool target_channel when responding to resolved_tools from this interaction."
    )]
    #[serde(skip_serializing_if = "Option::is_none")]
    channel: Option<MessageChannel>,

    #[schemars(description = "What AI said aloud to the user in this interaction.")]
    #[serde(skip_serializing_if = "Option::is_none")]
    cognitive_spoken: Option<CognitiveSpoken>,

    #[schemars(description = "What AI wrote to the user in chat in this interaction.")]
    #[serde(skip_serializing_if = "Option::is_none")]
    cognitive_written: Option<CognitiveWritten>,

    #[serde(skip_serializing_if = "Option::is_none")]
    cognitive_reasoning: Option<CognitiveReasoning>,

    #[serde(skip_serializing_if = "Vec::is_empty")]
    in_flight_tools: Vec<LlmSafeInFlightTool>,

    #[serde(skip_serializing_if = "Vec::is_empty")]
    resolved_tools: Vec<LlmSafeInFlightTool>,
}
```

```rust
// Built-in effect tool definitions:
// - "say" (registered in cognitive_direct_task): schema = CognitiveOutputSpeech
// - "write" (registered in cognitive_direct_task and cognitive_side_task when chat is enabled): schema = CognitiveOutputText
// - Each ErasedCommand in registries.commands: name = cmd.name(), description = cmd.description(), schema = cmd.schema()
```

### 5.3 JSON Contract Examples

#### Compiled Prompt Context Input (`CognitiveLLMContent`)
```json
{
  "available_documents": [],
  "working_memory": [],
  "interaction_memory": [
    {
      "user_messages": [
        {
          "Text": {
            "channel": { "context": {} },
            "sender": "test_user",
            "text": "Please answer: Hello, World",
            "attached_documents": [],
            "explicitly_addressed": true
          }
        }
      ],
      "channel": { "context": {} },
      "cognitive_written": {
        "target_channel": { "context": {} },
        "text": "Hello\nWorld"
      }
    }
  ],
  "user_messages": [
    {
      "Text": {
        "channel": { "context": {} },
        "sender": "test_user",
        "text": "Call mock_slow_read to find the architect and let me know you are checking.",
        "attached_documents": [],
        "explicitly_addressed": true
      }
    }
  ]
}
```

#### Simultaneous Effect + Query Tool Calls Emitted by LLM in a Single Turn (`Vec<ToolCall>`)
```json
[
  {
    "call_id": "call#write#0",
    "fn_name": "write",
    "fn_arguments": {
      "target_channel": { "context": {} },
      "text": "I am checking the document for the architect's name now."
    }
  },
  {
    "call_id": "call#mock_slow_read#1",
    "fn_name": "mock_slow_read",
    "fn_arguments": {}
  }
]
```

---

### 5.4 Architectural Fork Checkpoint (Phase 2)

When removing `CognitiveLLMOutput` (`responseJsonSchema`) from the cognitive loops (`direct.rs` and `side.rs`) so that `say`, `write`, `Command` plugins, and `Tool` plugins are all invoked via native `functionDeclarations`, how should `users_messages_evaluation` (turn gating fallback when `DecisionHandle` is unavailable) and `reasoning` be handled?

**[ Option A: Pure Function-Calling Turn (`output_schema: None`; Implicit Silence & Native Reasoning) ]**
- **Action:**
  - Set `output_schema: None` on `WithTools` cognitive calls (`direct.rs` and `side.rs`), and pass `say`, `write`, `Command` plugins, and `Tool` plugins as `functionDeclarations` (`toolChoice: AUTO`).
  - If the LLM calls one or more tools (`say`, `write`, `Command`, or `Tool`), the turn is treated as `Actionable`: Effect Calls (`say`, `write`, `Command`) execute immediately, and any Query Tool Calls (`Tool`) are dispatched to `RegistryToolExecutor` to wake the loop on completion.
  - If the LLM emits zero tool calls (or only plain text without calling `say`/`write`), the turn produces no side effects (equivalent to `WaitingForMoreInput` / `NonActionable` silence).
  - Populate `Interaction::cognitive_reasoning` from `ChatResponse::reasoning_content` (native model thinking tokens) when present.
- **Pros:** Simplest tool surface; zero extra parameters on `say`/`write` and zero extra meta-tools; completely eliminates `responseJsonSchema` from cognitive turns.
- **Cons:** When `DecisionHandle` preflight is disabled/unavailable and the LLM chooses not to call any tool, the cognitive loop does not distinguish between `WaitingForMoreInput` (keep `pending_user_messages`) and `Unintelligible` (clear `pending_user_messages` and call `on_unintelligible_input()`).

**[ Option B: Function-Calling Turn with Explicit `evaluate_turn` Fallback Tool (or `evaluation` Parameter) ]**
- **Action:**
  - Same as Option A (`output_schema: None`; `say`, `write`, `Command`, and `Tool` as native `functionDeclarations`), plus register a lightweight built-in effect tool `ignore_or_wait(evaluation: NonActionable | WaitingForMoreInput | Unintelligible, reasoning: String)` (or include `reasoning: Option<String>` on `say`/`write`) so that when `DecisionHandle` preflight is unavailable and the user input is incomplete or unintelligible noise, the LLM can explicitly signal `WaitingForMoreInput` (retain `pending_user_messages`), `NonActionable` (record silent interaction & clear `pending_user_messages`), or `Unintelligible` (discard noise & trigger `on_unintelligible_input()`).
- **Pros:** Preserves 100% behavioral parity for `WaitingForMoreInput`, `NonActionable`, and `Unintelligible` turn gating in `direct.rs` and `side.rs` (including scenarios like `edge_cases_test`) even when no `DecisionProvider` is configured.
- **Cons:** Adds one built-in control tool (`ignore_or_wait` / `evaluate_turn`) to the cognitive tool list when `DecisionHandle` is not used.

**My Recommendation:** I recommend **Option B** because `test_bundle` (and deployments without `DecisionProvider`) rely on the generative LLM to distinguish `WaitingForMoreInput` (buffering partial sentences in `pending_user_messages`) from `NonActionable` and `Unintelligible` (clearing `pending_user_messages` in scenarios like `edge_cases_test`), while still giving us full single-turn `say` + `write` + `Command` + `Tool` function calling with `output_schema: None`.
Please select Option A or Option B for this RFC.
