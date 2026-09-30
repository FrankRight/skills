# TypeScript testing reference

Verified against @agnt5/sdk 0.10.5 (`dist/context.d.ts`, `function.js`, `workflow.d.ts`,
`agent.d.ts`, `sandbox.d.ts`, `state.d.ts`, `scorer.d.ts`, registries) and the SDK's own
`vitest.config.ts` / `src/__tests__`.

## vitest setup

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
export default defineConfig({ test: { include: ['src/**/*.test.ts', 'tests/**/*.test.ts'], environment: 'node', globals: true } });
```

`npx vitest run` for tests, `npx tsc --noEmit` for types (the SDK's optional native loader is
fine under Node; there is no browser/Edge test target). Load `.env` yourself for `llm` tests
and skip with `describe.skipIf(!process.env.OPENAI_API_KEY)`.

## Local context

```typescript
import { ContextImpl } from '@agnt5/sdk';
const ctx = new ContextImpl('inv-1', 'run-1', 0, 'my-service', {
  storage: 'memory',                 // 'memory' | 'sqlite' (dbPath)
  metadata: { session_id: 's1' },    // optional runtime metadata
});
```

`ContextImpl` implements the full `Context` interface: `get/set/delete` (async, in-memory),
`step(name, fn, { key })` (runs `fn` once per name, no journal), `sleep(ms)`, `waitForUser`
(no responder offline - avoid), `waitForSignal` (throws `ConfigurationError` - only serverless
contexts implement it), `logger`, `emit` (no-op without an emitter), `signal` (never aborts).

## Calling components

```typescript
import { fn, workflow, tool, Agent } from '@agnt5/sdk';

export const greet = fn('greet').run(async (_ctx, name: string) => `Hello, ${name}!`);
export const onboarding = workflow('onboarding', async (ctx, input: { email: string }) => {
  const account = await ctx.step('create-account', () => createAccount(input.email));
  return { status: 'done', accountId: account.id };
});

// tests
await greet(ctx, 'Ada');                   // fn().run() returns a wrapper; without a platform ctx (emit + pushCorrelation) it calls the raw handler
await onboarding(ctx, { email: 'ada@example.com' });   // workflow() returns the handler itself
```

`fn().run()` and `workflow()` also register the name globally (`FunctionRegistry`,
`WorkflowRegistry`, `ToolRegistry`, `AgentRegistry`, `ScorerRegistry`); call the static
`clear()` in `beforeEach` when tests define components. Input schemas are not inferred from
TypeScript types - assert on the `inputSchema`/`outputSchema` you declared if the shape matters.

## Fake model for `Agent`

The root `GenerateRequest`, `GenerateResponse` and `LanguageModel` types are the agent's model
contract: a reply is `{ text, usage?, finishReason?, toolCalls? }`. (`LM` responses are
`LMGenerateResponse` and carry `id` and `model`; returning those fields here fails `tsc` with
TS2353.) Without a `stream()` method, the agent calls `generate()` for every turn.

```typescript
import { beforeEach, expect, it } from 'vitest';
import { Agent, ToolRegistry, tool } from '@agnt5/sdk';
import type { Context, GenerateRequest, GenerateResponse, LanguageModel } from '@agnt5/sdk';

class FakeModel implements LanguageModel {
  readonly requests: GenerateRequest[] = [];
  constructor(private readonly replies: GenerateResponse[]) {}

  async generate(request: GenerateRequest): Promise<GenerateResponse> {
    this.requests.push(request);
    const reply = this.replies.shift();
    if (!reply) throw new Error('FakeModel: no replies left');
    return reply;
  }
}

beforeEach(() => ToolRegistry.clear());

it('answers with the canned reply', async () => {
  const model = new FakeModel([{ text: 'The order is in transit.', finishReason: 'stop' }]);
  const agent = new Agent({ name: 'support', model, modelName: 'fake-model', instructions: 'Be brief.' });
  const result = await agent.run('Where is my order?');
  expect(result.output).toBe('The order is in transit.');
  expect(model.requests[0].systemPrompt).toContain('Be brief.');
});

it('drives a tool round', async () => {
  const lookupOrder = tool(
    'lookup_order',
    {
      description: 'Look up an order.',
      inputSchema: { type: 'object', properties: { order_id: { type: 'string' } }, required: ['order_id'] },
    },
    async (_ctx: Context, args: { order_id: string }) => `Order ${args.order_id} is in transit.`,
  );
  const model = new FakeModel([
    { text: '', toolCalls: [{ id: 'call_1', name: 'lookup_order', arguments: JSON.stringify({ order_id: '42' }) }] },
    { text: 'Order 42 is in transit.' },
  ]);
  const agent = new Agent({ name: 'support', model, modelName: 'fake-model', instructions: 'Use tools.', tools: [lookupOrder] });
  const result = await agent.run('Where is order 42?');
  expect(result.output).toBe('Order 42 is in transit.');
  expect(result.toolCalls.map((c) => c.name)).toEqual(['lookup_order']);
  expect(JSON.stringify(model.requests[1].messages)).toContain('Order 42 is in transit.');   // tool result went back
});
```

Keep `modelName` free of `/` (or use a real prefix such as `openai/gpt-4o-mini`): a name
with a slash is validated against the supported provider list even when the model is a fake,
and `fake/model` throws `ConfigurationError`.

## Sandbox, state, scorers

```typescript
import { InMemorySandbox, MemoryStateAdapter, StateManager, runScorer } from '@agnt5/sdk';

const sb = await new InMemorySandbox().start();           // sandboxId 'memory'
await sb.writeFile('notes.txt', 'hello');
expect((await sb.readFile('notes.txt')).content.toString()).toBe('hello');
expect((await sb.executeCode('print(1)', 'python')).stdout).toBe('[python] print(1)');   // echo backend

const state = new StateManager(new MemoryStateAdapter(), 'run', 'run-1');
await state.set('phase', 'started'); expect(await state.get('phase')).toBe('started');

const r = await runScorer('cites_order', { output: 'order 42', input: { order_id: '42' } });   // custom scorer() or built-in name
expect(r.passed).toBe(true);
```

Built-in deterministic scorers are also exported as functions (`exactMatch`, `contains`,
`jsonValid`, `jsonSchema`, `numericRange`, `regexMatch`, `levenshtein`,
`structuredAssertions`) plus trace helpers (`getToolCalls`, `toolTrajectoryExact`, ...);
judge helpers (`correctness`, `faithfulness`, `goalSuccess`, `agentJudge`, `llmJudge`) call a
provider (`agnt5-scorers`).

## Serverless handler offline

```typescript
import { serve } from '@agnt5/sdk/serverless';
const handler = serve({ serviceName: 'test', workflows: [hello] });   // no signingSecret -> unsigned accepted
const res = await handler.fetch(new Request('https://x/agnt5/invoke', {
  method: 'POST',
  body: JSON.stringify({ protocol_version: 'workerless.v1', run_id: 'r1', component_type: 'workflow', component_name: 'hello', input: { name: 'Ada' } }),
}));
expect(await res.json()).toEqual({ status: 'completed', output: { message: 'hello Ada' } });
```

## Against a running gateway

```typescript
import { Client } from '@agnt5/sdk';

const client = new Client({ gatewayUrl: process.env.AGNT5_GATEWAY_URL ?? 'http://localhost:34181' });   // agnt5 dev up
let res = await client.run('greet', { name: 'Ada' }, { waitTimeoutMs: 60_000 });
if (res.isPending) res = await client.waitForResult(res.runId, 120_000);
res.raiseForStatus();
```

`client.eval` / `client.batchEval` for scored checks (`agnt5-experiments`); gate them behind
an env flag so unit runs stay offline. Built-ins that need config take the object form, e.g.
`scorers: [{ name: 'contains', config: { pattern: 'in transit' } }]` (`agnt5-scorers`).
