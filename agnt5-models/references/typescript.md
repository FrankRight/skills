# TypeScript model reference

Verified against @agnt5/sdk 0.10.5 (`dist/lm.d.ts`, `dist/lm.js`, `dist/agent.d.ts`).

## Factories

`LM` has a private constructor; create one per provider. Every factory takes an optional
config and falls back to the env var:

| Factory | Config | Model prefix it accepts |
|---|---|---|
| `LM.openai({ apiKey?, organizationId?, baseUrl? })` | `OPENAI_API_KEY` | `openai/` |
| `LM.anthropic({ apiKey?, baseUrl? })` | `ANTHROPIC_API_KEY` | `anthropic/` |
| `LM.azure({ apiKey?, endpoint, apiVersion? })` | `AZURE_OPENAI_API_KEY` | `azure/` |
| `LM.google({ apiKey?, baseUrl? })` | `GOOGLE_API_KEY` / `GEMINI_API_KEY` | `google/` or `gemini/` |
| `LM.bedrock({ region?, accessKeyId?, secretAccessKey?, sessionToken? })` | `AWS_*` | `bedrock/` |
| `LM.groq`, `LM.fireworks`, `LM.openrouter`, `LM.deepseek`, `LM.mistral`, `LM.lepton`, `LM.together`, `LM.xai`, `LM.moonshot`, `LM.huggingface` (`{ apiKey?, baseUrl? }`) | `<PROVIDER>_API_KEY` | matching prefix (`huggingface/` or `hf/`); `LM.openrouter()` accepts any prefix (gateway provider) |
| `LM.baseten({ apiKey?, baseUrl?, authScheme? })` | `BASETEN_API_KEY` | `baseten/` |
| `LM.ollama({ baseUrl?, apiKey? })` | `OLLAMA_BASE_URL` | `ollama/` |
| `LM.openaiChat({ apiKey?, baseUrl?, organization? })` | - | `openai_chat/` (custom OpenAI-compatible chat APIs) |

`lm.providerName` tells you which one you hold. `SUPPORTED_MODEL_PROVIDERS`,
`parseModelIdentifier(model)`, and `validateModelForProvider(model, provider)` are exported;
`generate`/`stream` call the validator, so a missing or mismatched prefix throws
`ConfigurationError` before any request is made.

## `generate()`

```typescript
import { LM, createTool, jsonSchemaFormat, parseToolArguments, systemMessage, userMessage } from '@agnt5/sdk';
import type { LMGenerateRequest, LMGenerateResponse, ReasoningEffort } from '@agnt5/sdk';

const lm = LM.openai();
const request: LMGenerateRequest = {
  model: 'openai/gpt-4o-mini',
  systemPrompt: 'Route support tickets.',                 // or a leading systemMessage(...)
  messages: [userMessage(ticket)],                        // Message { role, content, toolCalls?, toolCallId?, name? }
  tools: [createTool('lookup_order', 'Fetch an order', { type: 'object', properties: { order_id: { type: 'string' } }, required: ['order_id'] })],
  toolChoice: { choiceType: 'auto' },                     // 'auto' | 'none' | { choiceType: 'tool', toolName }
  config: {
    temperature: 1,                                       // gpt-6 accepts only 1; omit or set 1
    maxOutputTokens: 300,
    topP: undefined,
    responseFormat: undefined,                            // jsonSchemaFormat(name, schema, strict?)
    reasoningEffort: 'medium',                            // type: 'minimal' | 'medium' | 'high'
    builtInTools: undefined,                              // ['web_search', 'code_interpreter', 'file_search', 'web_fetch']
    cache: undefined,                                     // true | { ttl, key, retention, resource } | ContextCache | string
    modalities: undefined,                                // ['text' | 'audio' | 'image']
    recoveryPolicy: undefined,                            // interrupted-work policy; default unknown_outcome
  },
  userId: undefined,
  prompt: undefined,                                      // { id, version?, variables? } for a managed prompt
};
const response: LMGenerateResponse = await lm.generate(request);
// response: { id, model, created?, text, usage?, finishReason?, toolCalls?, structuredOutput?, raw? }
// structuredOutput stays undefined in 0.10.5; parse response.text
for (const call of response.toolCalls ?? []) {
  const args = parseToolArguments<{ order_id: string }>(call);
  // reply: messages.push({ role: 'assistant', content: '', toolCalls: response.toolCalls },
  //                      { role: 'user', content: JSON.stringify(result), toolCallId: call.id, name: call.name })
  // keep call.providerData unchanged when replaying a tool call
}
```

`usage` is `{ promptTokens, completionTokens, totalTokens, cachedTokens, cacheCreationTokens }`.
`ToolDefinition` is `{ name, description?, parameters?: string (JSON), strict? }`;
`createTool()` serializes the schema for you.

Type `LM` requests and responses as `LMGenerateRequest` / `LMGenerateResponse`. The root
`GenerateRequest` / `GenerateResponse` exports are the agent's model contract (see below); using
them for `lm.generate()` fails `tsc` (no `toolChoice`, `maxOutputTokens` or `prompt`).

## Structured output

```typescript
const format = jsonSchemaFormat('verdict', {
  type: 'object', additionalProperties: false,
  properties: { label: { type: 'string' }, confidence: { type: 'number' } },
  required: ['label', 'confidence'],
}, true);                                                  // strict = true -> OpenAI strict mode
const res = await lm.generate({ model: 'openai/gpt-4o-mini', messages: [userMessage(text)], config: { responseFormat: format } });
const verdict = JSON.parse(res.text) as { label: string; confidence: number };
```

The schema reaches the provider and `res.text` is the JSON document, but `res.structuredOutput`
is `undefined` in 0.10.5, so parse the text. `ResponseFormatOption` is
`{ formatType: 'text' | 'json' | 'json_schema', schemaName?, schema? (JSON string), strict? }`.

## Streaming

```typescript
await lm.stream(request, (chunk) => {
  if (chunk.chunkType === 'delta' && chunk.content) process.stdout.write(chunk.content);
  if (chunk.chunkType === 'completed' && chunk.response) console.log(chunk.response.usage);
});
```

`stream()` resolves when the stream ends; there is no async iterator and no cancellation
(`AbortSignal` is ignored).

## Caching

```typescript
await lm.generate({ model: 'anthropic/claude-sonnet-5', systemPrompt: bigContext, messages, config: { cache: { ttl: '1h' } } });
const cache = await LM.google().createCache({ model: 'google/gemini-2.5-flash', contents: [manual], systemPrompt: 'You are the manual.', ttlSeconds: 3600 });
await LM.google().generate({ model: cache.model, messages, config: { cache } });   // ContextCache { name, model, provider: 'google' }
await LM.google().deleteCache(cache);
```

`cacheControl`, `cacheTtl`, and `googleCachedContent` in `GenerationConfig` are deprecated
aliases of `cache`. `normalizePromptCache(cache)` returns the resolved `PromptCache`.

## Agents and fakes

`new Agent({ name, model: LM.openai(), modelName: 'openai/gpt-4o-mini', instructions, temperature?, builtInTools?, ... })`
takes either an `LM` or any object implementing the legacy `LanguageModel` interface
(`generate(request): Promise<GenerateResponse>`, optional `stream()`), which is how you inject
a canned model in tests (`agnt5-testing`). Here `GenerateRequest` / `GenerateResponse` are the
root exports: a reply is `{ text, usage?, finishReason?, toolCalls? }` with no `id` or `model`.

## Quirks

- Bare model names throw; prefix must match the factory (see table).
- `response.structuredOutput` is always `undefined`; `JSON.parse(response.text)`.
- `ReasoningEffort` excludes `'low'`/`'none'`; cast when the provider supports them.
  `'minimal'` is rejected (400) by `gpt-6-luna`.
- Set `temperature: 1` for gpt-6 models (`Agent` and `generate`).
- No cancellation of in-flight calls; bound with `maxOutputTokens` and provider timeouts.
