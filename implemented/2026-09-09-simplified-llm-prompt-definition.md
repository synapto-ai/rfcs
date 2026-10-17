# [MVP] RFC: Direct Typed LLM Client Construction

## 1. Motivation and Goal

Currently, creating an `LLMClient` requires developers to define a dummy unit struct and implement the `LLM` trait solely to associate `Content` and `Output` types:

```rust
struct SessionLLMPrompt;
impl LLM for SessionLLMPrompt {
    type Content = SessionLLMContent;
    type Output = SessionLLMOutput;
}
```

This pattern creates boilerplate unit structs across every plugin that uses the LLM.

The goal of this RFC is to provide a direct generic factory function `synapto_llm::create_client::<In, Out>(...)` and `LLMClient::builder()` that removes the need for throwaway unit structs.

## 2. Executive Summary

- **Context:** Plugins instantiate typed `LLMClient` instances using `PromptStruct::create_client(...)`.
- **Problem:** Every prompt requires 6 lines of boilerplate to declare a zero-sized struct and an `impl LLM` block.
- **Solution:**
  1. Add `synapto_llm::create_client::<Content, Output>(name, executor, config, instructions)` as a direct factory function.
  2. Keep the `LLM` trait as an optional marker for named domain prompts.
  3. Allow inline client creation directly where prompts are defined.

## 3. Side-by-Side API Comparison

### 1. Prompt Definition and Client Creation

```rust
// CURRENT (Requires dummy unit struct and trait impl)
#[derive(Serialize, schemars::JsonSchema, Clone, Debug, LLMSafe)]
pub struct DocumentSummaryContent {
    pub text: String,
}

#[derive(Deserialize, Serialize, schemars::JsonSchema, Clone, Debug, LLMSafe)]
pub struct DocumentSummaryOutput {
    pub summary: String,
}

struct DocumentSummaryLLMPrompt;

impl LLM for DocumentSummaryLLMPrompt {
    type Content = DocumentSummaryContent;
    type Output = DocumentSummaryOutput;
}

let summary_client = DocumentSummaryLLMPrompt::create_client(
    llm_executor,
    config,
    instructions,
);
```

```rust
// PROPOSED (Direct generic construction, zero unit struct)
#[derive(Serialize, schemars::JsonSchema, Clone, Debug, LLMSafe)]
pub struct DocumentSummaryContent {
    pub text: String,
}

#[derive(Deserialize, Serialize, schemars::JsonSchema, Clone, Debug, LLMSafe)]
pub struct DocumentSummaryOutput {
    pub summary: String,
}

let summary_client = synapto_llm::create_client::<DocumentSummaryContent, DocumentSummaryOutput>(
    "DocumentSummary",
    llm_executor,
    config,
    instructions,
);
```

### 2. Implementation in `synapto-llm`

```rust
// In synapto_llm/src/lib.rs
pub fn create_client<Content, Output>(
    name: impl Into<String>,
    executor: synapto_interface::llm::LlmExecutor,
    config: synapto_interface::llm::ModelConfig,
    system_prompt: Vec<Instruction>,
) -> LLMClient<Content, Output, WithoutTools>
where
    Content: schemars::JsonSchema + Serialize + LLMSafe,
    Output: schemars::JsonSchema + DeserializeOwned + Clone + LLMSafe,
{
    let output_schema = schemars::schema_for!(Output);
    let input_schema = schemars::schema_for!(Content);
    let input_schema = serde_yaml::to_string(&input_schema).unwrap();
    let system_prompt_rendered = Instruction::render(&system_prompt, 0);

    LLMClient {
        executor,
        name: name.into(),
        model: config.model,
        input_schema,
        system_prompt: system_prompt_rendered,
        tools: vec![],
        tools_state: WithoutTools,
        output_schema,
        _marker: PhantomData,
    }
}
```

## 4. Key Simplifications

1. **Eliminated Throwaway Types:** Removes unit structs (`SessionLLMPrompt`, `ProgressionLLMPrompt`, `DocumentSummaryLLMPrompt`, etc.) from codebase namespaces.
2. **Reduced Lines of Code:** Removes 6 lines of boilerplate per prompt definition across all plugins.
3. **Explicit Component Naming:** Prompts provide explicit telemetry names via parameter instead of relying on compiler-generated `type_name` parsing.

## 5. Migration Plan

1. Add `create_client` and `create_client_with_tools` functions to `synapto-llm`.
2. Refactor existing plugins to remove dummy unit structs.
3. Keep `trait LLM` available for backward compatibility.
