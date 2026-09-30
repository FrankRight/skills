# Prompts and prompt caching in TypeScript

Verified against `@agnt5/sdk` **0.10.5**. Same section order as the Python SKILL.md. For the
rest of the direct model-call API (`LM.generate`, streaming, structured output) see
`agnt5-models`.

## Run a Prompt

```typescript
import { LM } from '@agnt5/sdk';

const lm = LM.openai();   // provider client; there is no module-level lm.generate()

const response = await lm.generate({
  model: 'openai/gpt-4o-mini',
  prompt: { id: 'support_reply', variables: { customer: { name: 'Ada' }, topic: 'shipping' } },
});
console.log(response.text);
```

`prompt` is a plain object (type `LMPrompt`): `{ id, version?, variables?, model?,
temperature?, maxOutputTokens?, topP?, projectId?, environmentId?, environmentRef? }`.
**Do not `import { Prompt } from '@agnt5/sdk'` for this** — that `Prompt` is the MCP server
prompt class; the LM prompt has no class. Do not pass `messages`/`systemPrompt` together with
`prompt`; the Prompt replaces them for that call. Variables may be nested objects
(`customer.name` resolves dotted paths).

## Commit a production Prompt

Identical file format (`prompts/<id>.mdx`, front matter + `<System>`/`<User>`/`<Assistant>`
blocks, `.md` or `.mdx` only). Resolution order in the TypeScript SDK:

1. `AGNT5_PROMPT_OVERRIDE` (a `.md`/`.mdx` file, or a directory)
2. `AGNT5_PROMPTS_MANIFEST` (same)
3. `<cwd>/prompts/<id>.mdx`, then `<cwd>/<id>.mdx`
4. AGNT5 prompt-run API (non-production draft/test only)

`cwd` is the worker's working directory (`npx tsx app.ts` from the project root), so keep
`prompts/` at the project root. Missing prompts fail closed in production.

## Select a version

```typescript
await lm.generate({ model: 'openai/gpt-4o-mini', prompt: { id: 'support_reply', version: '3' } });
```

Exact string match against the file's `version` (`'3'`) or `version_id` (UUID).

## Override runtime settings without changing the Prompt

```typescript
import type { LLMRuntimeOptions } from '@agnt5/sdk';

ctx.runtime.llm.model = 'openai/gpt-4o';
ctx.runtime.llm.temperature = 0.6;
ctx.runtime.llm.maxOutputTokens = 800;     // not max_tokens
ctx.runtime.llm.topP = 0.9;

// per-prompt overrides: plain objects, LLMRuntimeOptions = { model?, temperature?, maxOutputTokens?, topP? }
ctx.runtime.prompts['draft'] = { model: 'anthropic/claude-haiku-4-5', temperature: 0.7 };
ctx.runtime.prompts['review'] = { model: 'openai/gpt-4o', temperature: 0.3 };
```

`ctx.runtime` is `{ llm: LLMRuntimeOptions, prompts: Record<string, LLMRuntimeOptions> }`. A
prompt-specific entry replaces the global one for that prompt (no field merge). The platform
fills `ctx.runtime` from run metadata (Playground / experiments); code assignments apply to the
current run only.

## Use inside workflows

```typescript
import { fn, LM } from '@agnt5/sdk';
import type { Context } from '@agnt5/sdk';

const lm = LM.openai();

export const draftReply = fn('draft_reply').run(
  async (ctx: Context, input: { customerName: string; topic: string }) => {
    const response = await lm.generate({
      model: 'openai/gpt-4o-mini',
      prompt: { id: 'support_reply', variables: { customer: { name: input.customerName }, topic: input.topic } },
    });
    return response.text;
  },
);

// in the workflow — the step is what makes the reply replay instead of re-generating
const reply = await ctx.step('draft_reply', () => draftReply(ctx, { customerName, topic }), { key: ticketId });
```

The docs page says a `fn(...).run(...)` call checkpoints on its own; in 0.10.5 it does not —
always go through `ctx.step`.

## Compatibility note

`promptRef: 'support_reply'` still works but is deprecated; use `prompt: { id }`.

## Prompt caching

```typescript
import { Agent, LM } from '@agnt5/sdk';

// per agent: true, or a policy object (no PromptCache class)
const agent = new Agent({
  name: 'support', model: LM.anthropic(), modelName: 'anthropic/claude-sonnet-5',
  instructions: LONG_STABLE_INSTRUCTIONS, cache: { ttl: '1h', key: 'support-v3' },
});

// per call
const anthropic = LM.anthropic();
const response = await anthropic.generate({
  model: 'anthropic/claude-sonnet-5',
  systemPrompt: LONG_STABLE_INSTRUCTIONS,
  messages: [{ role: 'user', content: 'Answer the next support question.' }],
  config: { cache: true },
});

// Gemini explicit context cache
const gemini = LM.google();
const cache = await gemini.createCache({ model: 'google/gemini-2.5-pro', contents: [bigDoc], ttlSeconds: 3600 });
const r = await gemini.generate({ model: 'google/gemini-2.5-pro', messages: [...], config: { cache } });
await gemini.deleteCache(cache);
```

`cache` (on `AgentOptions` and `GenerationConfig`) accepts `boolean`, a policy object
`{ enabled?, ttl?, key?, retention?, resource? }`, the object returned by `createCache`, or a
cache resource name string; the union type itself is not exported from the package.
`cacheControl` / `cacheTtl` are deprecated aliases. Metrics on `response.usage` (optional
chaining — `usage` may be undefined): `cachedTokens`, `cacheCreationTokens`, `promptTokens`.
Provider behaviour (Anthropic 5-minute default TTL, OpenAI key/retention, Gemini implicit vs
explicit) is the same as Python.

**Getting hits**: keep `instructions`/`systemPrompt` byte-identical, put dynamic content in the
user message, keep the tool list stable, use the same `modelName` string.

**Silent invalidators in TypeScript**:
- `new Date()` / `Date.now()` interpolated into the system prompt
- `crypto.randomUUID()` or `Math.random()` at module load, appended to instructions
- `JSON.stringify(obj)` embedded in the prefix — key order follows insertion order; sort keys
  first (`JSON.stringify(obj, Object.keys(obj).sort())`)
- Template literals with per-request conditionals in the system prompt

## Not available in TypeScript

- Module-level `lm.generate(...)` / `lm.create_cache(...)` — construct a provider with `LM.<provider>()` and call its methods
- `Prompt` / `PromptRef` classes for the LM (plain object; the exported `Prompt` class is for `MCPServer`)
- `lm.PromptCache(...)` class (plain object policy)
- `LLMRuntimeOptions(...)` constructor (plain object; the name is only a type)
- `max_tokens` naming — it is `maxOutputTokens` everywhere (`GenerationConfig`, `LMPrompt`, `LLMRuntimeOptions`)

## TypeScript pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| `new Prompt({...})` has no `variables` field | wrong `Prompt` (MCP class) | use a plain `{ id, variables }` object |
| Prompt re-generates on every HITL resume | `draftReply(ctx, ...)` called without `ctx.step` | wrap in `ctx.step` |
| OpenAI 400 on `gpt-6-*` | `temperature` sent by default (0.7 on agents) | `temperature: 1` on the agent, `config.temperature: 1` on `generate`, or `temperature: 1` in the prompt front matter |
| `reasoningEffort: 'low'` does not type-check | `ReasoningEffort` is `'minimal' \| 'medium' \| 'high'`; `'minimal'` 400s on gpt-6-luna | `reasoningEffort: 'low' as ReasoningEffort` (works at runtime) |
| `response.usage.cachedTokens` — "possibly undefined" | `usage` is optional | `response.usage?.cachedTokens` |
| Prompt file not found in the deployed worker | resolved from `process.cwd()` | keep `prompts/` at the project root next to `app.ts` |

## Source

https://agnt5.com/docs/build/prompts · https://agnt5.com/docs/build/prompt-caching (TypeScript tabs)
