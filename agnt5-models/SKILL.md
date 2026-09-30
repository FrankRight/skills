---
name: agnt5-models
description: Call a model directly from AGNT5 code without an agent loop - Python agnt5.lm.generate/stream (messages, system_prompt, response_format, cache, store/previous_response_id) and GenerateRequest with tools, TypeScript LM.<provider>().generate/stream (config.maxOutputTokens, reasoningEffort, responseFormat via jsonSchemaFormat, builtInTools, cache, createCache), Go NewOpenAIModel/NewAnthropicModel/NewGoogleModel/NewAzureOpenAIModel plus the OpenAI-compatible constructors with ctx.Generate and GenerateRequest; the provider list and API-key env vars, the provider/model naming rule (bare names in Go), streaming events, and the current model quirks (gpt-6 temperature, Python reasoning_effort and structured-output gaps, TypeScript structuredOutput, ReasoningEffort and AbortSignal, Go max_tokens). Use when generating text or JSON from an LLM inside a function, workflow, or tool, choosing a model string, adding a provider, or debugging a 400/invalid-model error from a model call.
---

# AGNT5 Models

Use the model API for one call: classify, extract, summarize, draft. Use `Agent` when the
model needs a tool loop or memory (`agnt5-agents-tools`); use a managed Prompt when the text
should be versioned outside code (`agnt5-prompts`). Every call is traced as `lm.*` events
(`agnt5-observe`), and inside a workflow it is memoized like any step.

## Model identity: `provider/model` in Python and TypeScript, bare in Go

| SDK | Form | What happens otherwise |
|---|---|---|
| Python | `"openai/gpt-4o-mini"` | `ValueError: Model must include provider prefix` / unknown provider |
| TypeScript | `"openai/gpt-4o-mini"` **even with** `LM.openai()`; the prefix must match the factory (`LM.google()` takes `google/` or `gemini/`, `LM.huggingface()` takes `huggingface/` or `hf/`, `LM.openrouter()` accepts any prefix) | `ConfigurationError: Model must include provider prefix` or `Provider 'x' does not match model prefix` - the bare `'gpt-4'` examples in the `.d.ts` JSDoc are stale |
| Go | bare `Model: "gpt-4o-mini"` on the provider config or request | `"openai/gpt-4o-mini"` is sent verbatim and the provider answers 400 invalid model ID |

## Providers and credentials

Python and TypeScript share the Rust core, so they accept the same prefixes and read the same
env vars (`<PROVIDER>_BASE_URL` overrides exist for each). Go constructors take `APIKey`
explicitly and never read the environment (only Go's built-in judge scorers do).

| Prefix (Py/TS) | Env vars (Py/TS) | Go constructor |
|---|---|---|
| `openai` | `OPENAI_API_KEY` | `NewOpenAIModel(OpenAIConfig{APIKey, Model})` |
| `anthropic` | `ANTHROPIC_API_KEY` | `NewAnthropicModel(AnthropicConfig{APIKey, Model})` |
| `google`, `gemini` | `GOOGLE_API_KEY` or `GEMINI_API_KEY` | `NewGoogleModel` / `NewGeminiModel(GoogleConfig{APIKey, Model})` |
| `azure` | `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_VERSION` | `NewAzureOpenAIModel(AzureOpenAIConfig{Endpoint, APIKey, Deployment, APIVersion})` |
| `bedrock` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION` | none |
| `groq`, `openrouter`, `deepseek`, `mistral`, `together`, `xai`, `moonshot` | `GROQ_API_KEY`, `OPENROUTER_API_KEY`, `DEEPSEEK_API_KEY`, `MISTRAL_API_KEY`, `TOGETHER_API_KEY`, `XAI_API_KEY`, `MOONSHOT_API_KEY` | `NewGroqModel`, `NewOpenRouterModel`, `NewDeepSeekModel`, `NewMistralModel`, `NewTogetherModel`, `NewXAIModel`, `NewMoonshotModel` - all take `OpenAIConfig` |
| `ollama` | `OLLAMA_BASE_URL` (`OLLAMA_API_KEY`) | `NewOllamaModel(OpenAIConfig{})` (default `http://localhost:11434`) |
| `fireworks`, `huggingface`/`hf`, `baseten`, `lepton` | `FIREWORKS_API_KEY`, `HUGGINGFACE_API_KEY`, `BASETEN_API_KEY` (+`BASETEN_BASE_URL`, `BASETEN_AUTH_SCHEME`), `LEPTON_API_KEY` or `LEPTON_API_TOKEN` (+`LEPTON_BASE_URL`) | none; any OpenAI-compatible endpoint works with `NewOpenAIModel(OpenAIConfig{BaseURL, APIKey, AuthScheme})` |
| TS only: `openai_chat` | `LM.openaiChat({ baseUrl, apiKey, organization })` for custom OpenAI-compatible chat APIs | - |

Local dev: export the key or put it in `.env` (`agnt5-project-init`). Deployed workers:
Studio integrations or `agnt5 secrets set` (`agnt5-deploy`); code never sees the raw key.
Studio's first-class providers are openai, anthropic, google/gemini, groq, openrouter,
mistral, deepseek, xai.

## Generate

```python
from agnt5 import lm

response = await lm.generate(
    model="openai/gpt-4o-mini",
    system_prompt="Answer in one sentence.",
    messages=[{"role": "user", "content": question}],   # or prompt="..." for one turn
    max_tokens=200,                                      # temperature defaults to None (not sent)
)
print(response.text, response.usage.total_tokens, response.finish_reason)
```

```typescript
import { LM } from '@agnt5/sdk';
const lm = LM.openai();                                  // apiKey defaults from OPENAI_API_KEY
const response = await lm.generate({
  model: 'openai/gpt-4o-mini',
  systemPrompt: 'Answer in one sentence.',
  messages: [{ role: 'user', content: question }],
  config: { maxOutputTokens: 200 },
});
console.log(response.text, response.usage?.totalTokens, response.finishReason);
```

```go
model := agnt5.NewOpenAIModel(agnt5.OpenAIConfig{APIKey: os.Getenv("OPENAI_API_KEY"), Model: "gpt-4o-mini"})
maxTokens := 200
resp, err := ctx.Generate(model, agnt5.GenerateRequest{   // outside a handler: model.Generate(context.Background(), req)
    Messages: []agnt5.Message{
        {Role: agnt5.MessageRoleSystem, Content: "Answer in one sentence."},
        {Role: agnt5.MessageRoleUser, Content: question},
    },
    MaxTokens: &maxTokens,
})
fmt.Println(resp.Content, resp.Usage.TotalTokens, resp.FinishReason)
```

Per-language signatures, tool calling, structured output, streaming, and caching are in
`references/python.md`, `references/typescript.md`, `references/go.md`. Summary:

| Feature | Python | TypeScript | Go |
|---|---|---|---|
| Structured output | `response_format=PydanticModel \| dataclass \| dict` (schema sent) - **parse `response.text` yourself**, see quirks | `config.responseFormat = jsonSchemaFormat(name, schema, strict)` (schema sent) - **`JSON.parse(response.text)`**; `response.structuredOutput` is `undefined` | none; ask for JSON, `json.Unmarshal(resp.Content)` |
| Tools | `GenerateRequest(tools=[ToolDefinition], tool_choice=ToolChoice.AUTO)` via `LMClient(provider).generate()` (`lm.generate()` has no tools) | `tools: [createTool(name, desc, schema)]`, `toolChoice: { choiceType: 'auto' \| 'none' \| 'tool', toolName }` | `Tools: []agnt5.Tool{...}`, read `resp.ToolCalls` |
| Provider-hosted tools | `built_in_tools=[BuiltInTool.WEB_SEARCH]` | `config.builtInTools: ['web_search', 'code_interpreter', 'file_search', 'web_fetch']` | none |
| Reasoning | `reasoning_effort=` accepted but **not sent** | `config.reasoningEffort: 'minimal' \| 'medium' \| 'high'` | none |
| Responses API state | `store=`, `previous_response_id=`, `modalities=` | - | - |
| Prompt cache | `cache=True \| PromptCache(ttl=, key=, retention=)`; Gemini `lm.create_cache()` | `config.cache: true \| { ttl, key, retention, resource }`; `lm.createCache()` | `Cache: agnt5.EnablePromptCache() \| PromptCacheWithTTL("1h") \| PromptCacheResource(name)`; `GoogleModel.CreateCachedContent` |
| Streaming | `async for event in lm.stream(...)` -> `lm.content_block.delta` events in process | `lm.stream(request, chunk => ...)` with `chunkType: 'delta' \| 'completed'` | `StreamingLanguageModel` interface; built-in models only implement `Generate` |
| Managed prompt | `prompt=Prompt(id=...)`, `variables=` | `prompt: { id, variables }` | - |

## Model quirks (verified 29-30 Sep 2026)

- **gpt-6 family accepts only `temperature` 1.** Python: `lm.generate` sends nothing unless
  you pass it, but `Agent(...)` defaults to `temperature=0.7` - pass `temperature=None`.
  TypeScript: set `temperature: 1` (or omit it in `config`). Go: use non-reasoning models.
  Built-in judge scorers default to `temperature` 0.0, so a gpt-6 judge scores
  everything 0 - keep judges on another model (`agnt5-scorers`).
- **Python `reasoning_effort` is never sent** - the Rust binding has no such parameter, so the
  value is dropped silently. Use the provider default or TypeScript.
- **Python `response.structured_output` / `.parsed` / `.object` are always `None`**. The
  schema still goes to the provider; parse the text:
  `Model.model_validate_json(response.text)` or `json.loads(response.text)`.
- **Python Pydantic `response_format` under OpenAI strict mode** needs
  `additionalProperties: false` on every object: add `model_config = ConfigDict(extra="forbid")`
  to the model and its nested models. Dataclasses get it automatically; raw dict schemas need
  it added by hand.
- **TypeScript `response.structuredOutput` is always `undefined`** (0.10.5). The
  `responseFormat` schema is still sent and `response.text` holds the JSON:
  `JSON.parse(response.text)`.
- **TypeScript `ReasoningEffort`** is `'minimal' | 'medium' | 'high'`; `'low'` and `'none'`
  work at runtime with a cast (`'low' as ReasoningEffort`); `'minimal'` returns 400 on
  `gpt-6-luna`.
- **TypeScript `generate`/`stream` ignore `AbortSignal`**: `ctx.signal` does not
  cancel an in-flight model call; bound work with `maxOutputTokens` and your own timeouts.
- **Go Anthropic sends `max_tokens: 1024` unless `MaxTokens` is set** - set it for long
  outputs. Go has no structured-output, reasoning, or built-in-tool fields.
- **Go bare names.** `OpenAIConfig{Model: "openai/gpt-4o-mini"}` fails with 400; the Go
  constructors also do not read `OPENAI_API_KEY` - pass `APIKey` or requests go out
  unauthenticated.
- **Google explicit context caches only**: `cache.resource` / `PromptCacheResource` raise on
  any other provider in all three SDKs.

## Source

https://agnt5.com/docs/integrations/ai-providers · https://agnt5.com/docs/build/prompt-caching · https://agnt5.com/docs/build/prompts
