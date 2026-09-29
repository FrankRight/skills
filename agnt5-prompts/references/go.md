# Go prompts and prompt caching

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3**.

## Managed Prompts: not in Go

The Go SDK has no `Prompt` type, no `prompts/<id>.mdx` resolution, no
`AGNT5_PROMPT_OVERRIDE`/`AGNT5_PROMPTS_MANIFEST`, no version pinning and no
`ctx.runtime.llm`/`ctx.runtime.prompts` overrides. Keep prompt text in the repo as Go constants
or `text/template` files (that gives you the "prompt version moves with the code" property),
render it yourself, and pass it as messages. Choose model/temperature per run in code (from
input or env) — `LanguageModel` values only hold config and are cheap to construct.

## Calling a model directly

```go
model := agnt5.NewOpenAIModel(agnt5.OpenAIConfig{Model: "gpt-4o-mini", APIKey: os.Getenv("OPENAI_API_KEY")})

func DraftReply(ctx *agnt5.Context, in DraftInput) (string, error) {
    temp, maxTokens := 0.2, 600
    resp, err := ctx.Generate(model, agnt5.GenerateRequest{
        Messages: []agnt5.Message{
            {Role: agnt5.MessageRoleSystem, Content: supportSystemPrompt},          // byte-identical every call
            {Role: agnt5.MessageRoleUser, Content: renderUser(in.Customer, in.Topic)}, // dynamic content here
        },
        Temperature: &temp,     // omit for reasoning models (gpt-6 family rejects it; AGNT5-1303 unmerged)
        MaxTokens:   &maxTokens,
    })
    if err != nil {
        return "", err
    }
    return resp.Content, nil
}

// inside a workflow: checkpoint the call so replay reuses it
reply, err := agnt5.Task(ctx, "draft_reply", DraftInput{...}, DraftReply)
```

`ctx.Generate(model, req) (agnt5.GenerateResponse, error)` records the call as an lm span /
MODEL activation. `GenerateRequest`: `Model` (per-call override of the config model),
`Messages`, `Tools`, `Temperature *float64`, `MaxTokens *int`, `Cache *PromptCache`,
`Metadata`, `RecoveryPolicy`. `GenerateResponse`: `Content`, `Usage` (`InputTokens`,
`OutputTokens`, `TotalTokens`, `CachedTokens`, `CacheCreationTokens`), `FinishReason`,
`ToolCalls`, `Model`, `ID`.

## Prompt caching

| Python (SKILL.md) | Go |
|---|---|
| `Agent(..., cache=True)` | `agnt5.WithAgentPromptCache(agnt5.EnablePromptCache())` |
| `cache=lm.PromptCache(ttl="1h", key="support-v3")` | `agnt5.WithAgentPromptCache(&agnt5.PromptCache{TTL: "1h", Key: "support-v3"})` — any non-empty field enables |
| `lm.generate(..., cache=True)` | `agnt5.GenerateRequest{Cache: agnt5.EnablePromptCache()}` or `agnt5.PromptCacheWithTTL("1h")` |
| `lm.create_cache("google/gemini-2.5-pro", [doc], ttl_seconds=3600)` | `name, err := gemini.CreateCachedContent(ctx, "gemini-2.5-pro", systemText, []string{doc}, 3600)`; then `Cache: agnt5.PromptCacheResource(name)`; `gemini.DeleteCachedContent(ctx, name)` |
| `response.usage.cached_tokens` / `cache_creation_tokens` | `resp.Usage.CachedTokens` / `resp.Usage.CacheCreationTokens` |
| `cache_control=` / `cache_ttl=` (deprecated) | `agnt5.WithAgentCacheControl(enabled, ttl)` (deprecated alias) |

What the built-in providers send (`lm.go`):

- Anthropic: `cache_control: {"type": "ephemeral"}` plus `ttl` when set (5-minute default on the
  provider side; use `"1h"` for longer gaps).
- OpenAI-compatible (`NewOpenAIModel` and friends): nothing request-side; automatic prefix
  caching is reported back in `Usage.CachedTokens` only.
- Google: `cachedContent` when `Resource` is set (`PromptCacheResource`). `Resource` on any
  other provider returns `agnt5: explicit context caches are only supported for Google Gemini`.
- `Key` and `Retention` are carried on the struct but not sent by these providers.

To get hits, the Python rules apply unchanged: identical `WithAgentInstructions` text, dynamic
data in the user message, stable tool list, same model string. Go-specific silent invalidators:
building prompt text by ranging over a `map` (iteration order is random per call, so the bytes
differ), `time.Now()` in instructions, a per-process random suffix. Tool schemas are
`map[string]any` but `encoding/json` marshals map keys sorted, so those stay stable.

## Not available in Go

Managed `Prompt`/`PromptRef`, `prompts/*.mdx`, version selection, `LLMRuntimeOptions`,
`ctx.runtime.*`, `lm.generate` (use `ctx.Generate` or `model.Generate(ctx, req)`).

## Go pitfalls for this skill

- `Temperature`/`MaxTokens` are pointers: a `0.0` value must be `&temp`, not omitted by
  accident; nil means "provider default".
- Agents cannot set `MaxTokens`; the Anthropic provider always sends `max_tokens: 1024` unless
  the request sets it, so long outputs need `ctx.Generate` with `MaxTokens` (or a
  `LanguageModel` wrapper — see `agnt5-agents-tools`).
- Sending `Temperature`/`MaxTokens` to a gpt-6 model is rejected (AGNT5-1303 unmerged) and the
  judge's fixed `temperature: 0` makes gpt-6 unusable as a judge (AGNT5-1374).
