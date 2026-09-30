# Go model reference

Verified against github.com/agnt5dev/sdk-go v0.10.3 (`agnt5/lm.go`, `lm_http.go`,
`scorer_judge.go`).

## Constructors

There is no `"provider/model"` dispatch. Each provider is a constructor returning a
`LanguageModel` (`Generate(ctx context.Context, req GenerateRequest) (GenerateResponse, error)`).
Model names are **bare** (`"gpt-4o-mini"`, `"claude-sonnet-5"`) and `APIKey` is explicit -
nothing reads the environment for you.

| Constructor | Config | Defaults |
|---|---|---|
| `NewOpenAIModel(OpenAIConfig{...})` | `BaseURL, APIKey, APIKeyHeader, AuthScheme, Model, Organization, HTTPClient, Headers, Path` | `https://api.openai.com`, `/v1/chat/completions`, `Authorization: Bearer` |
| `NewAnthropicModel(AnthropicConfig{...})` | `BaseURL, APIKey, Model, HTTPClient, Version` | `https://api.anthropic.com`, version `2023-06-01`, **`max_tokens` 1024 unless `MaxTokens` set** |
| `NewGoogleModel` / `NewGeminiModel(GoogleConfig{...})` | `BaseURL, APIKey, Model, Version, HTTPClient` | `https://generativelanguage.googleapis.com`, `v1beta` |
| `NewAzureOpenAIModel(AzureOpenAIConfig{...})` | `Endpoint, APIKey, Deployment, APIVersion, HTTPClient` | `api-key` header, `/openai/deployments/<Deployment>/chat/completions?api-version=2024-02-15-preview` |
| `NewOpenRouterModel`, `NewGroqModel`, `NewDeepSeekModel`, `NewMistralModel`, `NewTogetherModel`, `NewXAIModel`, `NewMoonshotModel`, `NewOllamaModel` (all `OpenAIConfig`) | as OpenAI | base URLs `https://openrouter.ai/api`, `https://api.groq.com/openai`, `https://api.deepseek.com`, `https://api.mistral.ai`, `https://api.together.xyz`, `https://api.x.ai`, `https://api.moonshot.ai`, `http://localhost:11434` |

Any other OpenAI-compatible host (Fireworks, Baseten, Lepton, vLLM): `NewOpenAIModel(OpenAIConfig{BaseURL: ..., APIKey: ..., AuthScheme: "Api-Key"})`.
No Bedrock or Hugging Face constructor. HTTP retries for model calls read
`AGNT5_LM_MAX_RETRIES`, `AGNT5_LM_INITIAL_DELAY_MS`, `AGNT5_LM_MAX_DELAY_MS`.

## Request and response

```go
type GenerateRequest struct {
    Model          string          // overrides the config Model; bare name
    Messages       []Message       // {Role, Content, Name, ToolCallID, ToolCalls}
    Tools          []Tool          // agnt5.Tool{Name, Description, Schema map[string]any, Handler}
    Temperature    *float64
    MaxTokens      *int
    Cache          *PromptCache    // EnablePromptCache(), PromptCacheWithTTL("1h"), PromptCacheResource(name)
    Metadata       map[string]any
    RecoveryPolicy RecoveryPolicy  // default unknown_outcome
}
type GenerateResponse struct {
    ID, Model, Content, FinishReason string
    Usage     TokenUsage            // InputTokens, OutputTokens, TotalTokens, CachedTokens, CacheCreationTokens
    ToolCalls []ToolCall            // {ID, Name, Arguments map[string]any, Raw, CallID, ...}
    Metadata  map[string]any
}
```

Roles: `agnt5.MessageRoleSystem`, `MessageRoleUser`, `MessageRoleAssistant`, `MessageRoleTool`.
For Anthropic the first system message becomes the `system` field. Inside a component use
`ctx.Generate(model, request)` so the call is journaled as an `lm.*` activation and streamed
when the run is streaming; outside a component call `model.Generate(ctx, request)` directly.

```go
model := agnt5.NewAnthropicModel(agnt5.AnthropicConfig{APIKey: os.Getenv("ANTHROPIC_API_KEY"), Model: "claude-sonnet-5"})
maxTokens := 2048
resp, err := ctx.Generate(model, agnt5.GenerateRequest{
    Messages: []agnt5.Message{{Role: agnt5.MessageRoleUser, Content: "Summarize:\n" + doc}},
    MaxTokens: &maxTokens,
    Cache:     agnt5.EnablePromptCache(),
})
```

## Tool calling

```go
resp, err := model.Generate(ctx, agnt5.GenerateRequest{
    Messages: msgs,
    Tools: []agnt5.Tool{{Name: "lookup_order", Description: "Fetch an order",
        Schema: map[string]any{"type": "object", "properties": map[string]any{"order_id": map[string]any{"type": "string"}}}}},
})
for _, call := range resp.ToolCalls {
    result := lookup(call.Arguments["order_id"].(string))
    msgs = append(msgs,
        agnt5.Message{Role: agnt5.MessageRoleAssistant, ToolCalls: []agnt5.ToolCall{call}},
        agnt5.Message{Role: agnt5.MessageRoleTool, Content: result, ToolCallID: call.ID, Name: call.Name})
}
```

`Tool.Handler` is only used by `Agent`; for a direct call you execute the tool yourself.

## Structured output

No response-format field. Put the schema in the prompt, keep `Temperature` low on
non-reasoning models, and decode:

```go
var out struct{ Label string `json:"label"`; Confidence float64 `json:"confidence"` }
if err := json.Unmarshal([]byte(strings.TrimSpace(resp.Content)), &out); err != nil { /* retry or fail */ }
```

## Streaming

```go
type StreamingLanguageModel interface {
    LanguageModel
    Stream(ctx context.Context, request GenerateRequest, emit func(ModelStreamChunk) error) (GenerateResponse, error)
}
// ModelStreamChunk{Type, Content, Index, ToolCallID, ToolName, ArgumentsDelta, Arguments}
// Type: message_start | message_delta | message_stop | thinking_start | thinking_delta | thinking_stop |
//       tool_call_start | tool_call_delta | tool_call_stop
```

None of the built-in models implement `Stream`; `ctx.Generate` streams only when the run is
streaming (`ctx.IsStreaming()`) **and** the model implements the interface. Wrap a provider
model in your own type to add SSE parsing if you need token streaming.

## Test doubles and judge scorers

`agnt5.StaticModel{Model, Content, ToolCalls}` returns a fixed response (or echoes the last
message); `&agnt5.ScriptedModel{Responses: []agnt5.GenerateResponse{...}}` plays responses in
order. Both satisfy `LanguageModel` (`agnt5-testing`). Built-in judge scorers pick their
model from `config.model` and resolve keys from `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`,
`GOOGLE_API_KEY`/`GEMINI_API_KEY`, `MISTRAL_API_KEY`, `BASETEN_*`, `FIREWORKS_API_KEY`,
`GROQ_API_KEY`, `DEEPSEEK_API_KEY`, `OPENROUTER_API_KEY`, `LEPTON_*`; override with
`ctx = agnt5.WithLLMJudgeModel(ctx, agnt5.StaticModel{Content: `{"score":1,"passed":true}`})`.

## Quirks

- gpt-6 models are not usable from the Go SDK yet (temperature/reasoning handling is not
  in place yet); use non-reasoning models.
- Anthropic `max_tokens` defaults to 1024; set `MaxTokens` for long answers.
- `Temperature`/`MaxTokens` are pointers; nil means "not sent".
- Explicit context caches (`PromptCacheResource`) error on non-Google models;
  `GoogleModel.CreateCachedContent(ctx, model, system, contents, ttlSeconds)` and
  `DeleteCachedContent(ctx, name)` manage them.
