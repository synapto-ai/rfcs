# Formal Request for Comments (RFC) Protocol

## Core Directive: RFC Process Lifecycle

Activation Trigger: Activates when introducing a new major feature, subsystem, or architectural shift requiring structured design prior to code generation.

Overarching Goal: Systematically define, review, and lock in architectural decisions. Guarantee alignment with existing project patterns (e.g., event-driven flows, LLM safety boundaries), enforce robust trade-off analysis, and establish strict execution criteria.

Micro-Delegation Rule: During any phase, if a specific design debate arises (e.g., naming conventions, schema structures, native vs. bespoke implementations), spawn a subagent to generate a focused, side-by-side analysis of the options. Present the subagent's findings to the Lead Architect for a decision.

## Directory State Structure & Naming Convention

Every RFC and retrospective file MUST be prefixed with its creation date in `YYYY-MM-DD-` format (e.g., `2026-10-17-my-feature.md`) and placed in the directory corresponding to its current lifecycle state:

- `draft/YYYY-MM-DD-<slug>.md`: RFCs currently being drafted, reviewed, or awaiting failure scenario reproduction. Must include `[DRAFT]` in the H1 title.
- `reproduced/YYYY-MM-DD-<slug>.md`: `Flaw` RFCs whose primary runtime failure has been deterministically reproduced via an `xfail` (`#[should_panic(expected = "Scenario Failed")]`) scenario test in `synapto/crates/test`. Must include `[DRAFT]` in the H1 title until Phase 6 approval.
- `approved/YYYY-MM-DD-<slug>.md`: RFCs formally approved by the Lead Architect (`[DRAFT]` removed from H1 title) that are queued for or undergoing implementation.
- `implemented/YYYY-MM-DD-<slug>.md`: Completed RFCs where all Definition of Done (DoD) items, passing scenario tests (`#[should_panic]` removed and test/scenario renamed to canonical `<slug>` without the `rfc_<date>_` / `YYYY-MM-DD-` prefix), and Solution Verification checks have passed.
- `retrospectives/YYYY-MM-DD-<slug>-retro.md`: Post-implementation retrospectives documenting architectural evolution, rejected options, and mid-execution pivots.

When an RFC transitions between states:
- For `Flaw` RFCs: `draft/` -> `reproduced/` -> `approved/` -> `implemented/`
- For `Guideline` and `Feature` RFCs (where no existing runtime flaw exists to reproduce): `draft/` -> `approved/` -> `implemented/`
Preserve the original `YYYY-MM-DD-` creation date prefix and update the index table in `README.md`.

## Zero-Context Resumability Mandate (Draft Checkpoints & Approved Handover)

Every RFC must support starting a brand-new agent session (with zero prior conversation history) using only a prompt such as `"Continue with RFC <filename>"` at two boundaries:

1. **During Draft Phases (`draft/`, Phases 1-6):**
   - **No Context Only in Chat History:** Whenever an agent gathers audit findings, file paths, line numbers, defective code snippets, or design constraints prior to pausing (such as at the Phase 2 Architectural Fork Checkpoint), 100% of those concrete details MUST be written directly into Section 3 (*Current State Analysis*), Section 4 (*Structural Hierarchy*), and Section 5 (*Data Models*) of the draft file before yielding to the user. Never summarize findings in Section 2 while leaving the file paths and line numbers only in the chat transcript.
   - **Draft Status Header Block:** Every `[DRAFT]` file in `draft/` MUST include a top-level status block immediately below the H1 title indicating the exact current phase, blocking checkpoint, and next action upon resume:
     ```markdown
     > **Draft Status:** Phase 2 — Awaiting Lead Architect decision on Architectural Fork Checkpoint (Section 5.4).
     > **Next Step on Resume:** Apply selected fork option, complete Sections 6–10 (Phases 3–5), run zero-context subagent review, and present for Phase 6 approval.
     ```
2. **At Approved Handover (`approved/`, Phase 6 -> Phase 7 Boundary):**
   - When an RFC is finalized and moved from `draft/` to `approved/`, remove `[DRAFT]` from the H1 title and remove the `> **Draft Status:**` block.
   - The resulting `approved/` document MUST be 100% self-standing (verified by the Phase 3 zero-context subagent review) so a brand-new session can execute the entire Phase 7 implementation from start to finish without needing prior chat context.
   - Tracking partial mid-implementation progress inside the RFC during Phase 7 is not required.

## Standard RFC Template

All RFCs MUST adhere to the following 10-section Markdown structure:

1. **User-Centric Motivation & Goal** (A high-level, easily understandable explanation of _why_ we are doing this and _what_ it achieves from the user's perspective, avoiding deep technical jargon)
2. **Executive Summary** (Context, Technical Motivation, Overarching Goals, Purpose, Problem, Solution)
3. **Current State Analysis** (Process and flow of the existing architecture, explicitly identifying flaws, tight coupling, and inverted control)
4. **Structural Hierarchy** (Constraints and atomic responsibilities)
5. **Data Models** (Semantic schemas, enums, `!LLMSafe` boundaries, and concrete JSON examples for prompt contexts and output commands)
6. **Component Architecture** (Execution flow, event-driven boundaries)
7. **Architectural Diagrams** (Mermaid.js flowcharts and state diagrams)
8. **Robustness & Pragmatic Security Trade-offs** (Hazards, accepted risks, mitigations)
9. **Systemic Consequences** (Impact on other modules, breaking changes, performance implications)
10. **Definition of Done (DoD)** (Binary execution checklist, MUST include documentation checks/updates)

## Hierarchy of Truth & Scope Containment

- **Atomic Single-Problem RFC Mandate (1 Problem = 1 RFC):** Every RFC MUST address **exactly one specific problem, defect, or architectural invariant**. Never bundle multiple independent problems, unrelated subsystems, or mix runtime functional flaws with architectural guideline/style cleanups in a single RFC.
  - **Flaw vs. Guideline Classification:** Every RFC must declare its classification in its top-level status block and in `README.md`:
    - `Flaw`: Fixes a runtime bug, concurrency race, deadlock, data loss, resource leak, or security vulnerability.
    - `Guideline`: Enforces an architectural boundary, interface purity rule, configuration explicitness rule, strong typing rule, or documentation invariant.
    - `Feature`: Introduces a net-new capability or subsystem.
- **Prose Authority:** The descriptive text, intent, and motivation defined in the **User-Centric Motivation & Goal**, **Executive Summary**, **Current State Analysis**, **Structural Hierarchy**, and **Data Models** (Sections 1, 2, 3, 4, and 5) MUST always take precedence over code blocks, diagrams, or the Definition of Done (DoD). If a code example or a DoD item introduces a complexity or dependency (e.g., unnecessary wrappers, additional fields) not explicitly justified in the prose, it MUST be discarded.
- **Literal Interpretation:** Do not "bundle" architectural patterns from other subsystems unless they are explicitly required by the current task's problem definition.
- **No Complete Implementations (License Tainting Avoidance):** The RFC MUST NOT contain a full, complete implementation of the proposed change to avoid tainting by other licenses. See README.md. Keep code examples brief and focused strictly on API boundaries and data contracts.
- **Agile Design Pruning & Dependency Grounding (The Pragmatic Path):** Before proposing speculative, multi-subscriber broadcast networks, generic message buses, or intermediate translation loops, perform an explicit dependency audit of the target subsystem. Identify exactly what channels and data types it relies on (e.g., `tasks` only requires `State` and `InteractionMemory`). If the target subsystem only relies on a few existing channels, do **not** invent generalized broadcast systems or brokers. Simply move the required shared types/receivers down to the boundary interface (`synapto-interface`) and pass the existing receivers directly to the external plugin during bundle boot-time. Prioritize surgical, direct, type-safe wiring over speculative engineering.
  - **No Subsystem-Specific Interface Traits:** Under no circumstances should you define subsystem-specific traits (e.g., `TaskMemoryPluginTrait`) on the interface boundary crate (`synapto-interface`). The interface crate MUST define only generic, highly reusable capability traits (e.g., `ContextProviderPlugin`). The concrete plugin implementation implements this generic trait directly. Any subsystem-specific channels or receivers (such as `state_rx` and `interaction_memory_rx`) must be passed directly into the concrete plugin's constructor or setup method in the bundle boot sequence, and the resulting instance registered as a generic trait object. This prevents interface boundary pollution and keeps the core and interface completely decoupled.
- **Verification Rule:** During Phase 3 (Review), subagents MUST be specifically tasked to check for "Shadow Scope"—functionality or complexity that crept in through examples or DoD items without a corresponding requirement in the structural sections.

## Operational Phases

### Phase 1: Problem Definition & Structural Hierarchy (Initial Draft)

- **File Creation:** Immediately create the markdown file in the `draft` directory with the current date prefix in `YYYY-MM-DD-` format (e.g., `draft/2026-10-17-my-feature.md`) and add an entry to `README.md`. Mark the document clearly as a `[DRAFT]` in the H1 header. For initial versions focusing on functional core, explicitly mark them as `[MVP]`.
- **User-Centric Motivation:** Start by explicitly defining the human-level goal in Section 1. Explain _why_ the user will care about this change in plain language.
- **Executive Summary:** Isolate, ground, and articulate the following six pillars in Section 2:
  - **Context:** The current structural state and operational baseline of the subsystem.
  - **Motivation:** The direct friction, technical debt, or systemic inefficiencies driving the change.
  - **Overarching Goals:** The long-term architectural vision, scalability target, or decoupling objective.
  - **Purpose:** The immediate functional scope.
  - **Problem:** The specific issue/bug/bottleneck targeted by this RFC.
  - **Solution:** The proposed technical resolution.
- **Scope Lock (MVP):** If the RFC is marked as `[MVP]`, the scope MUST be strictly limited to replacing or refactoring existing functionality. Do not introduce new features, broader state machines, or additional component integrations (e.g., if refactoring a chat-specific component, do not expand it to voice unless explicitly requested).
- **Concrete Implementation Target:** Every RFC (abstract or structural) MUST include at least one concrete implementation target (e.g., refactoring a specific subsystem, creating a specific plugin) to prove the practical viability of the design. No purely abstract or "framework-only" RFCs without a concrete consumer.
- **Current State Analysis:** Explicitly outline how the relevant functionality works today. Identify specific points of friction, tightly coupled boundaries, and flawed execution flows before proposing the new architecture.
- **Structural Constraints:** Define the conceptual hierarchy (e.g., Mission -> Goal -> Task). Enforce atomic responsibility.
- **Output:** The saved draft file containing Sections 1, 2, and 3.

### Phase 2: Data Modeling & Architectural Alignment

- **Semantic Schema:** Draft data structures using core primitives. Identify state variations and enums.
- **JSON Contract Examples:** You MUST include concrete, readable JSON examples of both the compiled prompt context inputs (under `context_plugins`) and the LLM structured output command payloads to guarantee absolute alignment on runtime data formats.
- **Principle of Locality:** Tool-specific behavioral constraints (e.g., "Do not call this tool if X is true") must reside within the tool's description/metadata, not the global system prompt. Reserve the global prompt for identity and high-level architectural rules.
- **Pattern Alignment:** Read existing architecture documentation of the Synapto project ( `../synapto/docs/ARCHITECTURE.md`). Enforce their boundaries. Under no circumstances must the proposed design or API signatures violate the architectural guidelines (such as the SSoT, Exclusivity, and Channel Handover/Inversion of Control protocols defined in `docs/ARCHITECTURE.md`). Apply negative traits (e.g., `!LLMSafe`) to isolate internal state from LLM view models. Apply the **Opaque Resource Identifier (ORI)** protocol (as defined in `../synapto/docs/ARCHITECTURE.md`) to all entity IDs. **Explicit Citation Requirement:** You MUST explicitly cite the relevant rules from `../synapto/docs/ARCHITECTURE.md` that govern your proposed data models and boundaries within the RFC text.
- **Component Architecture:** Define the execution flow. Mandate event-driven, eventually consistent boundaries over scheduled polling or cron jobs.
- **Architectural Fork Checkpoint:** At the conclusion of Phase 2, formulate the chosen approach against at least one contrasting architectural paradigm (e.g., Approach A vs Approach B). **HALT EXECUTION HERE.** You MUST present this fork to the user (Lead Architect) and explicitly wait for their decision. DO NOT proceed to Phase 3, do not run subagent reviews, and do not draft subsequent sections until the Lead Architect has replied with their choice. **CRITICAL:** The user's selection of an option ONLY resolves the design question. It does **NOT** authorize execution. You must proceed to draft the remaining RFC sections and formally seek approval in Phase 6.
- **Output:** Sections 4 and 5 of the RFC document (Wait for user input before continuing to Phase 3).

### Phase 3: Visual Mapping & Architectural Review

- **System Topography**: Generate Mermaid.js flowcharts to map state mutations, channel communication, and component responsibilities.
- **Lifecycle Modeling**: Generate Mermaid.js state diagrams for complex data transitions.
- **Resource Ownership Validation (Mandatory Audit)**: Before finalizing an RFC review, perform a "Drift Audit" on all function signatures:
  - **Exclusivity Check**: Identify resources requiring a Single Source of Truth (SSoT). If a resource was previously an `Option` (for `take()` logic) and is now a bare type, flag it as a violation of the Exclusivity Protocol.
  - **Channel Suitability**: High-bandwidth data (e.g., `CognitiveOutputAudio`, `InputVoiceAudio`) MUST NOT use `broadcast` channels unless multi-subscriber support is explicitly required and the latency/lag impact is analyzed. Prefer `mpsc` for backpressured audio streams.
- **Verification**: Ensure the visual model adheres to the Single Source of Truth (SSoT) and Separation of Concerns protocols.
- **Zero-Context Subagent Review:** Use the `spawn_agent` tool to independently review the drafted RFC. This resets the context window and ensures the RFC document is completely self-contained, unambiguous, and structurally sound for a fresh executor. **If the RFC is marked as `[MVP]`, you MUST append the following sentence to the prompt:** "Note: This is an MVP version focusing exclusively on the functional core. Any suggested complex robustness measures (e.g., drop guards, retries, advanced timeouts) should be identified as missing but categorized as 'Accepted Risks' for future work rather than required for this initial implementation." Use the following exact prompt for the subagent:
  > "Please review the following drafted RFC as a zero-context architectural reviewer. Ensure it is completely self-contained, unambiguous, and structurally sound for a fresh executor. You MUST explicitly cross-reference the proposed design against standard project architecture patterns (e.g., plugin boundaries, channel-only wiring, interface segregation). Point out any flaws, ambiguities, or missing elements:
  >
  > ````markdown
  > [INSERT RFC CONTENT HERE]
  >
  > ```"
  >
  > ```
  > ````
- **Iterative Convergence Rule:** The RFC is a living document. If the subagent identifies flaws, the RFC must be updated directly by mutating its text. **Crucially, the main agent must filter subagent feedback against the MVP Simplification Rule.** If the subagent suggests complex protections (e.g., Drop Guards, multi-step recovery) for an `[MVP]` RFC, these MUST be documented as "Accepted Risks" in Section 8 rather than implemented in the core logic. Do not append logs of corrections. Whenever changes are made based on the subagent's feedback, the review subagent MUST be spawned again to verify the fixes. This process must converge. If the subagent diverges (continuously requests cyclical changes without resolution), STOP immediately and escalate to the human Lead Architect.
- **Output:** Section 7 (Diagrams).

### Phase 4: Robustness & Security Trade-off Analysis

- **Hazard Interrogation:** Execute the `Robust by Design` and `Pragmatic Security` protocols (as defined in your system instructions). Anticipate infinite loops, state corruption, and resource exhaustion.
- **MVP Simplification Rule:** If the RFC is marked as `[MVP]`, the first implementation MUST NOT include complex protections like retries, advanced timeouts, pagination, or multi-step recovery. These should be documented as "Accepted Risks / Future Work" in Section 7. Focus exclusively on the functional core.
- **Trade-off Documentation:** Explicitly define the accepted risks. State the balance between implementation simplicity and operational risk (e.g., LLM inference cost vs. architectural complexity).
- **Mitigation Strategy:** Document safeguards (e.g., TTLs, atomic writes, truncation limits).
- **Output:** Section 8 of the RFC document.

### Phase 5: Systemic Impact & Implementation Planning

- **Mandatory Codebase Audit:** Before writing the Systemic Consequences section, you MUST execute raw search commands (e.g., `rg`, `grep`) for every core type, channel, or function being moved or decoupled.
- **Audit Logging:** The RFC MUST explicitly list the file paths of all external modules that currently import or depend on the targeted subsystem, proving the dependency web was mechanically checked.
- **Systemic Consequences:** Explicitly document how the proposed change affects other parts of the system. Identify any breaking changes to internal APIs, shared data structures, or plugin interfaces. Assess the impact on system-wide performance (latency, memory) and state consistency.
- **Documentation:** Explicitly state what high-level (`docs/`) or low-level documentation must be updated, created, or verified. Ensure proper separation of concerns per `DOCUMENTATION.md` (e.g., Architectural Anti-patterns and structural invariants belong in `ARCHITECTURE.md`, while code-level Best Practice recipes and practical examples belong in `CREATE_PLUGIN.md`). If no documentation changes are needed, explicitly state why.
- **Execution Checklist:** Construct a binary, non-negotiable Definition of Done (DoD). This checklist MUST include the concrete implementation target (e.g., migrating a specific plugin) that proves the design works in practice. Specify _what_ files are moved _where_ and what docs are updated. For `Flaw` RFCs, the DoD MUST include:
  1. Removing `#[should_panic(expected = "Scenario Failed")]` from `rfc_<YYYY_MM_DD_rfc_slug_with_underscores>` in `synapto/crates/test/tests/scenario_tests.rs`.
  2. Verifying that the scenario test passes cleanly.
  3. Renaming the scenario directory from `tests/scenarios/<YYYY-MM-DD-rfc-slug>` to `tests/scenarios/<rfc-slug>`, updating `name` in `scenario.yaml`, and renaming the test function from `rfc_<YYYY_MM_DD_rfc_slug_with_underscores>` to its canonical name `<rfc_slug_with_underscores>`.
- **Scope:** Include dependencies, data models, persistence mechanisms, event integrations, command mutations, and specific test simulations required to validate the feature.
- **Output:** Sections 9 and 10 of the RFC document.

### Phase 5.5: Failure Scenario Reproduction (`Flaw` RFCs Only)

- **Activation Condition:** Mandatory for all RFCs classified as `Flaw` where the defect manifests at runtime. Skipped for `Guideline` and `Feature` RFCs (which transition directly from `draft/` to `approved/` upon Phase 6 approval).
- **Objective:** Create a scenario test in `synapto/crates/test` that deterministically reproduces the primary failure(s) described in the RFC on the current unmodified codebase before seeking Lead Architect approval to modify production code:
  1. Create `synapto/crates/test/tests/scenarios/<YYYY-MM-DD-rfc-slug>/scenario.yaml` with `name: "<YYYY-MM-DD-rfc-slug>"` and steps (`user_writes`, `user_says`, `wait`, `await_response`) that trigger the defect and fail the final assertion.
  2. Register the test in `synapto/crates/test/tests/scenario_tests.rs` named `rfc_<YYYY_MM_DD_rfc_slug_with_underscores>` and mark it as `xfail` using `#[should_panic(expected = "Scenario Failed")]`:
     ```rust
     #[ignore]
     #[should_panic(expected = "Scenario Failed")]
     #[tokio::test(flavor = "multi_thread", worker_threads = 2)]
     async fn rfc_<YYYY_MM_DD_rfc_slug_with_underscores>() {
         run_scenario(
             "tests/scenarios/<YYYY-MM-DD-rfc-slug>/scenario.yaml",
             test_bundle,
         )
         .await;
     }
     ```
  3. Do not add custom `xfail` fields to `ScenarioManifest` or modify core code; use the direct implementation.
  4. Run `just test-scenarios -p synapto-test rfc_<YYYY_MM_DD_rfc_slug_with_underscores>` in `synapto` to verify that the scenario fails with `Scenario Failed` and passes the `#[should_panic]` assertion.
  5. **State Transition:** Move the RFC file from `draft/YYYY-MM-DD-<slug>.md` to `reproduced/YYYY-MM-DD-<slug>.md` and update `README.md`.
- **Exception Handling:** If a `Flaw` RFC targets a low-level component that cannot be triggered through `ScenarioManifest` steps, explain why to the Lead Architect and either provide a deterministic `#[should_panic]` unit/integration test in the target crate or request an explicit Phase 5.5 exemption.

### Phase 6: Final Review & Approval Gate

- **Presentation:** Present the completed RFC (and for `Flaw` RFCs, the verified `reproduced/` scenario test) to the user (Lead Architect).
- **Authorization:** Explicitly ask for approval to proceed. DO NOT advance to execution without clear human authorization. Approval must be conscious and specific; "OK" or generic affirmations are insufficient. The user must explicitly state they have reviewed and approved the design for implementation.

### Phase 7: Execution & Amendment Fallback

- **Plan Lock-In:** Remove the `[DRAFT]` tag and `> **Draft Status:**` block from the RFC document, move the file from `reproduced/YYYY-MM-DD-<slug>.md` (or `draft/YYYY-MM-DD-<slug>.md` for `Guideline`/`Feature` RFCs) to `approved/YYYY-MM-DD-<slug>.md`, update `README.md`, and freeze the design.
- **Pre-Implementation Recheck:** Before starting implementation, perform a final audit of the current codebase to ensure all assumptions, data models, and architectural boundaries defined in the RFC still apply and have not been invalidated by recent changes.
- **Iterative Implementation:** Execute the DoD sequentially. Apply `Proactive Change Implementation` to code changes. For `Flaw` RFCs:
  1. Remove `#[should_panic(expected = "Scenario Failed")]` from `rfc_<YYYY_MM_DD_rfc_slug_with_underscores>` in `synapto/crates/test/tests/scenario_tests.rs`.
  2. Run `just test-scenarios -p synapto-test rfc_<YYYY_MM_DD_rfc_slug_with_underscores>` to confirm the scenario now passes cleanly.
  3. Rename `synapto/crates/test/tests/scenarios/<YYYY-MM-DD-rfc-slug>` to `synapto/crates/test/tests/scenarios/<rfc-slug>`, update `name` in `scenario.yaml` to `"<rfc-slug>"`, rename the test function to `<rfc_slug_with_underscores>`, and re-run `just test-scenarios -p synapto-test <rfc_slug_with_underscores>`.
- **Drift Check:** If implementation reveals a superior approach (e.g., native protocol vs. custom schema), immediately halt execution to amend the RFC. The codebase and the RFC MUST remain synchronized.
- **Validation:** Execute the `Solution Verification Protocol` upon completion to ensure the codebase mirrors the accepted RFC constraints.
- **Amendment/Rollback Procedure:** If execution reveals fatal flaws, immediately halt. Unfreeze the RFC (re-add `[DRAFT]` and move the file from `approved/` back to `draft/`), document the blocking issue, propose an architectural amendment, and cycle back to Phase 2 or 3 (to actively redesign the data models and diagrams) before seeking Phase 6 re-approval.

### Phase 8: Retrospective Generation & Process Improvement

- **State Transition & Documentation:** Once implementation is complete and the `Solution Verification Protocol` has passed, move the RFC from `approved/YYYY-MM-DD-<slug>.md` to `implemented/YYYY-MM-DD-<slug>.md`, create a `retrospectives/YYYY-MM-DD-<slug>-retro.md` document (using the date the retrospective is created), and update `README.md`.
- **Content:** Document the iterative evolution of the RFC. Explicitly record rejected approaches and the reasons for their rejection. The retrospective must accurately reconcile the initial RFC proposal with the final implemented code, documenting the 'Why' behind any mid-execution pivots.
- **Attribution:** Clearly attribute the source of pivotal ideas, corrections, and paradigm shifts (e.g., Main Agent, Subagent, Lead Architect).
- **Process Evolution:** After writing the retrospective, analyze the interaction meta-process itself. If workflow bottlenecks, miscommunications, or repeated mistakes occurred, explicitly propose concrete improvements to `AGENTS.md` (as was done during the stateless document retrieval RFC) to iteratively refine the RFC protocol for future designs.
- **Output:** The saved retrospective markdown file and any proposed process adjustments.

## Third-Party Dependency Authorization & RFC Drift Protocol

**Activation Trigger:** Any task requiring the addition of a new third-party crate not already present in the workspace, OR any point during implementation where the code must deviate from the approved RFC to function.

1. **Dependency Authorization (Zero-Trust Additions):**
   - Before modifying `Cargo.toml` to add a new 3rd-party dependency, the agent MUST present the proposed crate to the Lead Architect.
   - The proposal must include the crate name, its primary purpose, and a brief trade-off analysis (e.g., compile time, standard adoption, underlying dependencies like `tonic`/`tokio`).
   - The agent MUST halt and wait for explicit human authorization before proceeding.
2. **RFC Drift Fallback (Implementation Stop):**
   - If execution reveals a fatal flaw, missing API, or architectural blocker that requires violating the constraints, data models, or boundaries established in the approved RFC.
   - The agent MUST immediately STOP implementation.
   - The agent MUST present the blocker to the Lead Architect, explain why the RFC is currently unviable, and offer architectural alternatives (including an amendment to the RFC).

# Implementation & Testing Protocols

## 1. Test-Driven Tool Development

Any new tool implementation must be accompanied by a corresponding test scenario in `scenarios/<tool-name>-test/scenario.yaml`. The test must be executed using `just test-run` to verify the tool's behavior in the actual `test-mode` runtime before concluding the task.

## 2. Scenario Structure

Test scenarios must strictly follow the `steps` and `action` schema (e.g., `user_writes`, `await_response`) as defined in `src/test_mode.rs`. High-level field shortcuts (like `text:` or `assert_contains:` at the root) are deprecated and should not be used for new tests.

## Absolute Prohibitions

1. **No "SAFETY" Comments**: Never use the `SAFETY:` prefix or the word `SAFETY` in code comments or explanations unless strictly writing `unsafe` Rust blocks that require it by compiler convention.
2. **No Alarmist Language**: Do not use the word "critical", "crucial", "urgent", or similar alarmist/dramatic terms in comments or text. Calmly describe the mechanism, why it exists, and what it prevents.
