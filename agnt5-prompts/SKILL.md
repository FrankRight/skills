---
name: agnt5-prompts
description: Manage AGNT5 Prompt artifacts as versioned, code-bundled text - author prompts/<id>.mdx files, pin or select a prompt version, override runtime model/temperature per run (lm.generate only), and enable prompt caching (cache=True / lm.PromptCache, Gemini context caches) to get cache hits. Use when extracting inline prompt text into a managed Prompt, pinning a prompt version, changing how a prompt resolves in dev vs production, or debugging low cache-hit rates.
---

# AGNT5 Prompts

> **TypeScript or Go?** This file shows the Python API. Read [references/typescript.md](references/typescript.md) or [references/go.md](references/go.md) first: same sections, the exact signatures for that SDK, and what it does not support. Known SDK bugs with workarounds are in the `agnt5-sdk-pitfalls` skill.

A **Prompt** is a managed LLM prompt you can draft/test in AGNT5, then commit alongside your
application code for production — so the prompt version and code version move together.

## Run a Prompt

```python
from agnt5 import lm
from agnt5.lm import Prompt   # NOT `from agnt5 import Prompt` — that is the MCP Prompt type

response = await lm.generate(
    model="openai/gpt-4o-mini",
    prompt=Prompt(id="support_reply", variables={"customer": {"name": "Ada"}, "topic": "shipping"}),
)
print(response.text)
```

Don't pass raw `messages`/prompt text alongside `prompt=`; `prompt` replaces them for that
call.

## Commit a production Prompt

Create `prompts/<id>.mdx` in the application repo:

```mdx
---
id: support_reply
version: 3
version_id: 018f0000-0000-7000-8000-000000000003
model: openai/gpt-4o-mini
temperature: 0.2
max_tokens: 600
variables:
  - customer.name
  - topic
response_format: text
---

<System>
You are a concise support agent.
</System>

<User>
Reply to {{customer.name}} about {{topic}}.
</User>
```

Front matter drives routing/generation settings; the body renders into ordered chat messages
via `<System>`, `<User>`, `<Assistant>` blocks (no block present → treated as one user
message). Files must be Markdown or MDX — not `.json` or `prompts.lock`.

**Production resolution order:**
1. `AGNT5_PROMPT_OVERRIDE` (a `.md`/`.mdx` file, or a directory searched like the cwd)
2. `AGNT5_PROMPTS_MANIFEST` (same)
3. `<cwd>/prompts/<id>.mdx`, then `prompts/<id>.md`, then `<cwd>/<id>.mdx`, `<cwd>/<id>.md`
4. AGNT5 prompt-run API fallback (non-production draft/test only)

In production, the Prompt **must** be bundled with the deployed artifact — a missing Prompt
fails closed, it does not fall back to control-plane state.

## Select a version

```python
response = await lm.generate(
    model="openai/gpt-4o-mini",
    prompt=Prompt(id="support_reply", version="3"),
)
```

`version` is an exact string match against the file's `version` (`"3"`) or `version_id`
(the UUID) — `"version-3"` or `"v3"` will not match.

## Override runtime settings without changing the Prompt

Useful for Playground / experiments / one-off comparisons — workflow code stays unchanged:

```python
from agnt5 import LLMRuntimeOptions

ctx.runtime.llm.model = "openai/gpt-4o"
ctx.runtime.llm.temperature = 0.6
ctx.runtime.llm.max_tokens = 800
ctx.runtime.llm.top_p = 0.9

# per-prompt overrides for workflows with multiple prompts
ctx.runtime.prompts["draft"] = LLMRuntimeOptions(model="anthropic/claude-haiku-4-5", temperature=0.7)
ctx.runtime.prompts["review"] = LLMRuntimeOptions(model="openai/gpt-4o", temperature=0.3)
```

The Prompt file remains the source of truth for prompt *text*; runtime overrides only change
model execution settings for that run. A prompt-specific override **replaces** the global one
for that prompt (fields are not merged — set every field you need).

Scope (0.13.6): only `lm.generate()` reads `ctx.runtime.llm` / `ctx.runtime.prompts`.
`lm.stream()` and `Agent` ignore them, so a model call that must be overridable from the
Playground or an experiment has to go through `lm.generate`. Direct model-call options
(`response_format`, streaming, provider quirks such as gpt-6) are covered in `agnt5-models`.

## Use inside workflows

Keep the LLM call inside a `@function`/checkpointed step so replay reuses the completed
result instead of re-calling the model:

```python
@function
async def draft_reply(ctx: FunctionContext, customer_name: str, topic: str) -> str:
    response = await lm.generate(
        model="openai/gpt-4o-mini",
        prompt=Prompt(id="support_reply", variables={"customer": {"name": customer_name}, "topic": topic}),
    )
    return response.text
```

## Compatibility note

Older code may use `prompt_ref="support_reply"` / `PromptRef`. Still supported, but new code
should use `prompt=Prompt(id=...)`.

## Prompt caching

Reuses an identical leading prefix (tool defs → system prompt → messages) without reprocessing
— large latency and token-cost savings for the cached portion. Turn it on per call or per
agent:

```python
response = await lm.generate(model="anthropic/claude-sonnet-5", prompt=..., cache=True)
agent = Agent(name="support", model="anthropic/claude-sonnet-5", instructions=..., cache=True)

# explicit policy: TTL, cache key, retention
agent = Agent(..., cache=lm.PromptCache(ttl="1h", key="support-v3"))

# Gemini explicit context cache for a large reusable document
cache = await lm.create_cache("google/gemini-2.5-pro", [big_doc], ttl_seconds=3600)
response = await lm.generate(model="google/gemini-2.5-pro", prompt=..., cache=cache)
```

`cache_control=` / `cache_ttl=` on `Agent` are deprecated aliases for `cache=`. AGNT5
surfaces the numbers on `response.usage`:

| Field | Meaning |
|---|---|
| `cached_tokens` | Tokens read from cache (cache hit) |
| `cache_creation_tokens` | Tokens written to cache this call (cache write) |
| `prompt_tokens` | All input tokens, including reads and writes |

On Anthropic the cache TTL defaults to 5 minutes (resets on hit); use `ttl="1h"` for longer
gaps. Minimum cacheable prefix is ~1024-4096 tokens (model-dependent) — short system prompts won't
cache.

**To get cache hits**: keep `instructions=`/`system_prompt` text byte-identical across calls
for the same agent role, put dynamic content (names, dates, request context) in the *user*
message not the system prompt, keep the tool list stable, use the same `model` string every
call.

**Silent invalidators** (every call becomes a cache miss):
- `datetime.now()` / `date.today()` embedded in system prompt text
- A UUID/random seed generated at startup and appended to instructions
- JSON-serializing a dict with unstable key order (use `sort_keys=True`)
- Conditional instruction blocks that vary by request data

## Source

https://agnt5.com/docs/build/prompts · https://agnt5.com/docs/build/prompt-caching
