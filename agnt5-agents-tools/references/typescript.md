# Agents and tools in TypeScript

Verified against `@agnt5/sdk` **0.10.5**. Same section order as the Python SKILL.md.

## Creating an agent

```typescript
import { Agent, LM } from '@agnt5/sdk';

export const researcher = new Agent({
  name: 'researcher',
  model: LM.openai({ apiKey: process.env.OPENAI_API_KEY }),   // provider client
  modelName: 'openai/gpt-4o-mini',                             // provider/model string
  instructions: 'You are a research assistant. Answer questions with cited sources.',
});
```

| Option | Required | Notes |
|---|---|---|
| `name` | yes | Component name; also the tool name when used as an agent-as-tool |
| `model` | yes | `LM.openai()`, `LM.anthropic()`, `LM.google()`, `LM.groq()`, `LM.openrouter()`, ... (`LM` reads the provider's `*_API_KEY` env var when `apiKey` is omitted) |
| `modelName` | no | Defaults to `openai/gpt-4o-mini`. Prefix must match the `model` provider or the constructor throws `ConfigurationError` |
| `instructions` | yes | System prompt |
| `tools` | no | `tool()` results, `Tool` instances, MCP tools, or other `Agent`s |
| `builtInTools` | no | `['web_search' \| 'code_interpreter' \| 'file_search' \| 'web_fetch']` (string union, not an enum) |
| `handoffs` | no | `Agent` or `handoff(...)` entries |
| `sandbox` | no | `new Sandbox({...})` |
| `maxIterations` | no | Default `10` |
| `temperature` | no | Default `0.7`, sent explicitly. Omitted automatically only for `openai/gpt-5*`, `o1*`, `o3*`, `o4*`. For `gpt-6*` pass `temperature: 1` (AGNT5-1302) |
| `cache` | no | `true` or `{ ttl, key, retention, resource }` — see `agnt5-prompts` |
| `callbacks` | no | `{ beforeAgent, afterAgent, beforeModel, afterModel, beforeTool, afterTool }` |
| `skills` / `skillsDir` / `agentsMd` | no | See `agnt5-agent-skills` |

`maxTokens`, `topP` and `modelConfig` are not `AgentOptions`; base URL / org / key go on the `LM`
constructor (`LM.openai({ apiKey, baseUrl, organizationId })`).

Run: `const result = await agent.run('...')` → `AgentResult`:
`{ output, toolCalls: [{ name, arguments, iteration, id?, builtIn? }], context, handoffTo, handoffMetadata }`.
Multi-turn: `const [reply, messages] = await agent.chat('Hello', messages)`.

Stream: `for await (const item of agent.stream('...'))`. The generator yields `AgentEvent`s **and,
last, the `AgentResult` itself** (no `eventType`), so narrow first:

```typescript
for await (const item of agent.stream(task, ctx)) {
  if (!('eventType' in item)) { console.log(item.output); continue; }   // final AgentResult
  switch (item.eventType) {
    case 'lm.message.delta': process.stdout.write(item.content ?? ''); break;
    case 'tool_call.started': console.log('tool', item.toolName); break;
  }
}
```

| Event type | When |
|---|---|
| `agent.started` / `agent.completed` / `agent.failed` | Loop begins / final answer / error |
| `agent.iteration.started` / `agent.iteration.completed` | One reasoning loop |
| `lm.message.start` / `.delta` / `.stop`, `lm.thinking.*`, `lm.tool_call.start` / `.delta` / `.stop` | Token streaming (not `lm.content_block.*`) |
| `tool_call.started` / `tool_call.completed` / `tool_call.failed` | A tool call |
| `skill.loaded` | A SKILL.md was loaded |

Inside a workflow, pass `ctx` as the second positional argument: `agent.run(task, ctx)`. Wrap
the call in `ctx.step('agent', () => agent.run(task, ctx))` so the finished answer replays
instead of re-running the model (AGNT5-1373). Do not put an agent that carries
`AskUserTool`/`RequestApprovalTool` inside a step: its pause must propagate to the workflow.
Any `beforeModel` or `afterModel` callback turns token streaming off (`lm.message.delta` stops).

## Custom tools

```typescript
import { tool } from '@agnt5/sdk';
import type { Context } from '@agnt5/sdk';

export const getWeather = tool(
  'get_weather',
  {
    description: 'Get the current weather for a city.',
    inputSchema: {
      type: 'object',
      properties: { city: { type: 'string', description: 'Name of the city to check.' } },
      required: ['city'],
    },
    recoveryPolicy: 'idempotent_retry',   // default 'unknown_outcome'
  },
  async (ctx: Context, args: { city: string }): Promise<string> => `Sunny, 22°C in ${args.city}`,
);
```

- Signature: `tool(name, options, handler)`; the handler is `(ctx, args)` with one args object.
- No type-hint inference. **Without `inputSchema` the model sees an empty parameter list**
  (`{ type: 'object', properties: {} }`). Generate it from a Zod or TypeBox schema with
  `zodToJsonSchema(schema)` / `typeBoxToJsonSchema(schema)` if you already have one.
- Options: `description` (defaults to the name), `inputSchema`, `recoveryPolicy`
  (`'idempotent_retry' | 'durable_steps' | 'unknown_outcome' | 'compensate' | 'fail'`),
  `durable` (default `true`), `confirmation` (typed, not enforced — use `agnt5-human-in-the-loop`).
- `tool()` registers globally at call time, so the tool is a worker component
  (`agnt5 run get_weather --type tool`). Import the module from `app.ts`.
- The return value is callable (`await getWeather(ctx, { city })`) and exposes the `Tool`
  instance as `getWeather._tool` — that is what `MCPServer` and `addTool` want.

## Built-in tools

```typescript
const agent = new Agent({ ..., builtInTools: ['web_search'] });
```

Same provider matrix as Python. There is no `agnt5.tools.web_fetch()` / `web_search()`
provider-agnostic factory in TypeScript: write a `tool()` around `fetch` or use an MCP server.

## MCP tools

```typescript
import { MCPClient } from '@agnt5/sdk';

const mcp = new MCPClient('repo-tools');
mcp.addStreamableHttpServer('deepwiki', 'https://mcp.deepwiki.com/mcp');   // (name, url, headers?, apiKey?, timeout?)
mcp.addSseServer('internal', 'https://example.com/sse', { Authorization: 'Bearer ...' });
mcp.addStdioServer('local', 'npx', ['-y', 'my-mcp-server']);              // (name, command, args?, env?, cwd?)
await mcp.connect();                                                      // before getTools()
const agent = new Agent({ ..., tools: mcp.getTools() });
```

Methods match Python in camelCase: `connect()`, `disconnect()`, `getTools()`, `listTools()`,
`listServerTools(server)`, `callTool(server, name, args)`, `callToolAuto(name, args)`,
`isConnected(server)`, `connectedServers()`. `callTool` returns `{ content: [{ type, text? }], isError }`
(no `getText()`). No `async with`: use `try/finally` or `await using mcp` (`Symbol.asyncDispose`).

Expose components: `new MCPServer({ id, name, version, tools: { greet: greet._tool }, agents: { assistant } })`
then `await server.runStdio()` or `server.runHTTP({ host, port, path })`.

## Sandboxed code execution

```typescript
import { Agent, LM, Sandbox } from '@agnt5/sdk';

const agent = new Agent({ ..., sandbox: new Sandbox({ provider: 'e2b' }) });
```

`SandboxOptions`: `provider`, `template`, `env`, `cpuCores`, `memoryMib`, `timeoutSecs`,
`autoDestroy` (default `true`). `new Sandbox()` picks the first provider with credentials in
the environment (`E2B_API_KEY`, `DAYTONA_API_KEY`, ...). **`AGNT5_SANDBOX_PROVIDER` is not read
by the TypeScript SDK** — pass `provider` explicitly. Custom tools reach the workspace through
`ctx.sandbox` (`Sandbox | undefined`): `executeCode(code, language?, timeoutMs?)`, `writeFile`,
`readFile` (returns `{ content: Buffer }`), `listFiles`, `deleteFile`. There is no
`runCommand`. `sandboxTools({ sandbox })` returns the same tools for manual composition;
`InMemorySandbox` is the no-network test double. Credentials per provider:
[references/sandbox-providers.md](sandbox-providers.md).

## Multi-agent patterns

**Agents as tools** — pass the `Agent` in `tools`. The tool is named **`<agent.name>`**
(not `ask_<name>`) and takes one `message` argument; tell the coordinator that name.

```typescript
const coordinator = new Agent({
  ..., tools: [researchAgent, analystAgent],   // tools "researcher" and "analyst"
});
```

**Handoffs** — `handoff()` is positional: `handoff(agent, description?, toolName?, passFullHistory?, joinPolicy?)`.

```typescript
import { Agent, LM, handoff, ChildJoinPolicy } from '@agnt5/sdk';

const triage = new Agent({
  ..., handoffs: [
    handoff(billingAgent, 'Transfer for billing/payment questions'),
    handoff(technicalAgent, 'Transfer for technical issues', 'transfer_to_tech', true, ChildJoinPolicy.Detached),
  ],
});
const result = await triage.run('My payment failed but I was still charged.');
result.handoffTo;   // 'billing'
```

Defaults: description = target's `instructions`, `toolName` = `transfer_to_<name>`,
`passFullHistory` = `true`, `joinPolicy` = `ChildJoinPolicy.Required`. Handoffs can only be set
in the constructor and there is no depth limit — two agents that hand off to each other loop
until `maxIterations` on each side (AGNT5-1357). Passing agents directly
(`handoffs: [billingAgent]`) uses the defaults.

## Callbacks (guardrails, caching, redaction)

```typescript
import { Agent, callbackOverride } from '@agnt5/sdk';
import type { ToolCallbackContext } from '@agnt5/sdk';

const BLOCKED = new Set(['delete_account']);

const agent = new Agent({
  ...,
  callbacks: {
    beforeTool: (cb: ToolCallbackContext) =>
      BLOCKED.has(cb.toolName) ? { error: `${cb.toolName} is not allowed` } : undefined,
    afterModel: (cb, request, response) => undefined,
  },
});
```

Return `undefined` to continue, a value to short-circuit (skip the tool/model), or
`callbackOverride(value)` when the replacement itself is `undefined`/falsy. Contexts:
`AgentCallbackContext { agent, context, userMessage, history? }`,
`ModelCallbackContext { agent, context, iteration, messages, toolDefs }`,
`ToolCallbackContext { agent, context, iteration, toolName, toolCallId, toolCall, args, tool? }`
(`args`, not `arguments`). There are no `beforeToolCallback`-style top-level options; only the
`callbacks` object.

## Memory

There is no `ctx.memory` or `ctx.conversation`. The classes are standalone:

```typescript
import { ConversationMemory, SemanticMemory } from '@agnt5/sdk';

const convo = new ConversationMemory(sessionId);           // MemoryStateAdapter by default
await convo.add('user', message);
const history = await convo.getAsLmMessages(20);            // [{ role, content }]

const mem = new SemanticMemory('user', userId);            // in-memory word-overlap search
await mem.store('Prefers concise answers', { extra: { kind: 'preference' } });
const hits = await mem.search('answer style', 5);
```

Both live in the worker process unless you pass your own `StateAdapter` /
`SemanticMemoryAdapter`; `SemanticMemory.fromEnv(scope, id)` uses OpenAI embeddings plus a
configured vector DB when the native binding is available. When an agent component is invoked
with a `session_id`, the worker loads and saves its chat history for you; nothing else persists
across runs.

## Not available in TypeScript

- `ctx.memory.*`, `ctx.conversation.*`, `ctx.session`, `ctx.user`
- `agnt5.tools.web_search()` / `web_fetch()` factories
- `model_config` / `ModelConfig`, `max_tokens`, `top_p` on the agent
- `before_*_callback` keyword options and the `AgentCallbacks(...)` class (use the `callbacks` object)
- `Sandbox.run_command` / `run_command_stream`, `AGNT5_SANDBOX_PROVIDER`
- `async with mcp:` (use `await using` or `try/finally`)
- Tool schema inference from types or docstrings
- Trace spans for tool/model calls (AGNT5-1320) — use `ctx.logger` and the event stream

## TypeScript pitfalls

| Symptom | Cause | Fix | Ticket |
|---|---|---|---|
| OpenAI 400 "temperature not supported" on gpt-6 | agent sends `0.7` by default | `temperature: 1` on `Agent` (or `config.temperature` on `LM.generate`) | AGNT5-1302 / AGNT5-1285 |
| `result.output` is a tool's JSON after a long run | `maxIterations` hit: output = last message content | raise `maxIterations`, or detect `toolCalls.length` and re-prompt | AGNT5-1353 |
| Model never fills tool arguments | no `inputSchema` | always pass `inputSchema` | — |
| Coordinator calls `ask_researcher` and fails | agent-as-tool is named `researcher` | use `<agent.name>` in instructions | — |
| No `lm.message.delta` events | `beforeModel`/`afterModel` disables streaming | drop the callback or consume `agent.completed` | — |
| `ConfigurationError: Provider ... does not match model prefix` | `modelName` prefix vs `LM` provider | match them (`LM.anthropic()` + `anthropic/...`) | — |
| Agent missing from `agnt5 components` | agents are not auto-registered | `worker.registerAgents([agent])` | — |
| Two agents hand off forever | no handoff depth limit | avoid mutual handoffs; lower `maxIterations` | AGNT5-1357 |
| `agent.run` re-runs on HITL resume | not inside `ctx.step` | wrap in `ctx.step` (unless it holds HITL tools) | AGNT5-1373 |

## Source

https://agnt5.com/docs/build/agents · /docs/build/tools · /docs/build/mcp · /docs/build/sandboxes (TypeScript tabs)
