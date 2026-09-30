# Python model reference

Verified against agnt5 0.13.6 (`src/agnt5/lm/__init__.py`, `lm/client.py`, `lm/types.py`,
`lm/events.py`, `_schema_utils.py`).

## `lm.generate()`

```python
from agnt5 import lm
from agnt5.lm import BuiltInTool, Message, PromptCache, ReasoningEffort

response = await lm.generate(
    model="openai/gpt-4o-mini",             # required, "provider/model"
    prompt=None,                            # str for a single user turn, or Prompt / dict for a managed prompt
    messages=[{"role": "user", "content": "..."}],   # list of dicts; roles system/user/assistant
    system_prompt=None,
    temperature=None, max_tokens=None, top_p=None,   # None = not sent
    cache=None,                             # True | PromptCache(...) | ContextCache | str
    response_format=None,                   # Pydantic model class, dataclass, or JSON-schema dict
    built_in_tools=None,                    # [BuiltInTool.WEB_SEARCH, .CODE_INTERPRETER, .FILE_SEARCH, .WEB_FETCH]
    reasoning_effort=None,                  # ReasoningEffort.MINIMAL | MEDIUM | HIGH - accepted, never sent
    modalities=None, store=None, previous_response_id=None,   # OpenAI Responses API
    variables=None, project_id=None, environment=None, environment_id=None, prompt_version=None,  # managed prompts
)
```

`GenerateResponse` fields: `text`, `usage` (`prompt_tokens`, `completion_tokens`,
`total_tokens`, `cached_tokens`, `cache_creation_tokens`), `finish_reason`, `tool_calls`
(list of dicts), `response_id`. `structured_output` / `parsed` / `object` exist but return
`None` in 0.13.6.

`prompt` and `messages` are mutually exclusive with a managed prompt (`prompt=Prompt(id=...)`
or the deprecated `prompt_ref=`); runtime model overrides from Studio are applied before the
call (`agnt5-prompts`). Supported prefixes: `lm.SUPPORTED_MODEL_PROVIDERS` (anthropic, azure,
baseten, bedrock, deepseek, fireworks, gemini, google, groq, hf, huggingface, lepton, mistral,
moonshot, ollama, openai, openrouter, together, xai).

## Structured output

```python
from pydantic import BaseModel, ConfigDict

class Verdict(BaseModel):
    model_config = ConfigDict(extra="forbid")   # emits additionalProperties: false (OpenAI strict mode)
    label: str
    confidence: float

response = await lm.generate(model="openai/gpt-4o-mini", prompt=text, response_format=Verdict)
verdict = Verdict.model_validate_json(response.text)      # do not rely on response.parsed
```

`detect_format_type()` converts a Pydantic class with `model_json_schema()`, a dataclass to a
schema with `additionalProperties: false`, and passes dicts through unchanged.

## Tool calling (`GenerateRequest` + `LMClient`)

`lm.generate()` exposes no `tools`. Build the request yourself:

```python
from agnt5.lm import GenerateRequest, GenerationConfig, LMClient, Message, ToolChoice, ToolDefinition

client = LMClient(provider="openai")
request = GenerateRequest(
    model="openai/gpt-4o-mini",
    messages=[Message.system("You route support tickets."), Message.user(ticket)],
    tools=[ToolDefinition(name="lookup_order", description="Fetch an order",
                          parameters={"type": "object", "properties": {"order_id": {"type": "string"}},
                                      "required": ["order_id"]})],
    tool_choice=ToolChoice.AUTO,           # AUTO | NONE | REQUIRED
    config=GenerationConfig(max_tokens=300),
)
response = await client.generate(request)
for call in response.tool_calls or []:      # provider-shaped dicts: id, name/function, arguments
    ...
# continue the conversation:
request.messages += [Message.assistant("", tool_calls=response.tool_calls),
                     Message.tool_result(tool_call_id=call["id"], content=json.dumps(result))]
```

`Message` helpers: `Message.system()`, `.user()`, `.assistant(content, tool_calls=)`,
`.tool_result(tool_call_id, content)`. `LMClient` implements the `agnt5.lm.LanguageModel`
ABC (`generate`, `stream`), the same interface `Agent(model=...)` accepts, so a subclass
with canned responses is the test double (`agnt5-testing`).

## Streaming

```python
from agnt5.lm import LMCompleted, LMContentBlockDelta

async for event in lm.stream(model="anthropic/claude-sonnet-5", prompt="Tell me a story", max_tokens=400):
    if isinstance(event, LMContentBlockDelta):      # event.block_type: "text" | "thinking"
        print(event.content, end="")
    elif isinstance(event, LMCompleted):
        print(event.input_tokens, event.output_tokens, event.finish_reason)
```

Event classes and `event_type` strings: `LMStarted` (`lm.started`), `LMContentBlockStarted`
(`lm.content_block.started`), `LMContentBlockDelta` (`lm.content_block.delta`, payload in
`.content`), `LMContentBlockCompleted` (`lm.content_block.completed`), `LMCompleted`
(`lm.completed`: `model`, `provider`, `input_tokens`, `output_tokens`, `total_tokens`,
`cached_tokens`, `finish_reason`), `LMFailed` (`lm.failed`). `lm.stream()` takes the same
keyword arguments as `generate()` except `response_format` and the managed-prompt fields.

## Prompt caching

```python
await lm.generate(model="anthropic/claude-sonnet-5", system_prompt=big_context, prompt=q, cache=True)
await lm.generate(model="openai/gpt-4o-mini", prompt=q, cache=PromptCache(key="support-v3", retention="24h"))

cache = await lm.create_cache("google/gemini-2.5-flash", contents=[manual_text],
                              system_prompt="You are the manual.", ttl_seconds=3600)   # Gemini explicit cache
await lm.generate(model=cache.model, prompt=q, cache=cache)
await lm.delete_cache(cache)
```

`PromptCache(enabled=True, ttl=None, key=None, retention=None, resource=None)`: Anthropic maps
`ttl` to its ephemeral cache TTL, OpenAI Responses maps `key`/`retention` to
`prompt_cache_key`/`prompt_cache_retention`, Gemini maps `resource` to `cachedContent`. Cache
hits show up in `usage.cached_tokens`. Details and Agent-level `cache=` in `agnt5-prompts`.

## Quirks that bite in Python

- Never pass `temperature=0` to a gpt-6 model; `Agent(...)` defaults to `0.7`, so pass
  `temperature=None` there.
- `reasoning_effort` is dropped in `LMClient._build_kwargs()` -> Rust binding (no parameter).
- `structured_output`/`parsed` are `None`; parse `text`.
- `lm.generate()` inside a workflow is memoized per step position like any model call; give
  surrounding work stable `ctx.step` keys when it runs in loops (`agnt5-workflows`).
