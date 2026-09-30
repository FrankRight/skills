# Experiments, datasets and inline evals in TypeScript

Verified against `@agnt5/sdk` **0.10.5**. Sections 1–3, CI gating, regression datasets,
rescoring and annotations in the SKILL.md are CLI/API only and apply unchanged. This file
covers the SDK half: `client.eval()` / `client.batchEval()`. For the rest of `Client` see
`agnt5-client`; for scorer objects see `agnt5-scorers`.

## Client setup

```typescript
import { Client } from '@agnt5/sdk';

const client = new Client({
  gatewayUrl: process.env.AGNT5_GATEWAY_URL,   // default: http://localhost:34181 (NOT gw.agnt5.com)
  apiKey: process.env.AGNT5_API_KEY,           // also read from the env automatically
});
```

The TypeScript default gateway is `http://localhost:34181` (the local `agnt5 dev` gateway).
Against a deployed worker set `AGNT5_GATEWAY_URL=https://gw.agnt5.com`, `AGNT5_API_KEY`, and
pass `deploymentId` in the eval options (the ambient `AGNT5_DEPLOYMENT_ID` is not used for
execution). `Client` is async-only; there is no `AsyncClient`.

## Inline evals from code: `client.eval()`

```typescript
import { Client, Correctness } from '@agnt5/sdk';

const client = new Client();
const result = await client.eval(
  'support_agent',
  { message: 'Where is my order #1234?' },
  {
    componentType: 'agent',                  // default 'function'
    expected: 'Your order #1234 is in transit',
    scorers: [new Correctness()],            // strings, presets, LLMJudge, or raw spec objects
    deploymentId: process.env.CANDIDATE_DEPLOYMENT_ID,
    timeout: 60_000,                         // milliseconds
  },
);

console.log(result.passed, result.output);
for (const s of result.scores) console.log(s.scorer, s.score, s.passed, s.explanation);
result.getScore('exact_match');              // ScorerResultSummary | undefined, by scorer name
result.raiseForStatus();                     // throws RunError when the run itself failed
```

`eval(component, inputData?, options?)` — positional component and input; options:
`expected`, `scorers`, `componentType`, `deploymentId`, `sessionId`, `userId`, `timeout` (ms).
Returns `EvalResponse`: `output`, `scores: ScorerResultSummary[]`, `passed`, `runId`,
`traceId`, `durationMs`, `error`, getters `isSuccess` / `isError` / `elapsed`. With `expected`
and no `scorers`, `exact_match` is used.

## `client.batchEval()`

```typescript
import { Client } from '@agnt5/sdk';
import type { BatchEvalItem } from '@agnt5/sdk';

const items: BatchEvalItem[] = [                          // plain objects, not a class
  { input: { message: 'Where is my order #1234?' }, expected: '...', itemId: 'order-status' },
  { input: { message: 'Cancel order #5678' }, expected: '...', itemId: 'order-cancel' },
];

const result = await client.batchEval('support_agent', items, {
  componentType: 'agent',
  scorers: ['exact_match'],
  maxConcurrency: 10,                         // default 10; start at 3–5 in development
  timeout: 60_000,                            // per item, milliseconds
});

console.log(`Pass rate: ${(result.passRate * 100).toFixed(0)}%`);
for (const item of result.results) {
  console.log(item.itemId, item.passed ? 'PASS' : 'FAIL', item.durationMs);
}
```

Item forms (mixable): `{ input, expected?, itemId?, index? }`; `{ input, expected }` records;
or plain input objects with a separate `expected: [...]` list in the options (the docs page
says this last form is Python-only; `normalizeBatchEvalItems(items, expected)` accepts it).
`batchEval` fans out client-side (one `/v1/eval` call per item under the concurrency cap).

`BatchEvalResult`: `batchId`, `status` (`'completed' | 'partial_failure' | 'failed'`),
`results: BatchEvalItemResult[]`, `stats` (`totalItems`, `completedItems`, `failedItems`,
`passedItems`, `avgDurationMs`, `durationMs`), getters `passRate`, `isSuccess`,
`isPartialFailure`, `outputs`; methods `passingItems()`, `failingItems()` (scoring failures),
`failedItems()` (evaluation errors — check both).

`BatchEvalItemResult`: `index`, `runId`, `output`, `scores`, `passed`, `durationMs`,
`itemId`, `traceId`, `error`, `getScore(name)`, `isSuccess` / `isFailed`.

```typescript
if (result.status === 'partial_failure') {
  for (const item of result.failedItems()) console.log('eval error', item.itemId, item.error);
}
for (const item of result.failingItems()) console.log('score fail', item.itemId);
```

## Gate a CI job from code

```typescript
const result = await client.batchEval('support_agent', items, { componentType: 'agent', scorers: [new Correctness()] });
if (result.passRate < 0.9 || result.failedItems().length > 0) {
  console.error(`pass rate ${result.passRate}, ${result.failedItems().length} errors`);
  process.exit(2);
}
```

For platform-tracked gates keep using `agnt5 experiments run ... --wait --fail-on-gate`.

## Datasets and experiments from the SDK

There is no dataset/experiment API on the TypeScript `Client` (no `datasets`, `experiments`
or `reports` methods). Use the CLI commands in the SKILL.md, the REST endpoints, or the AGNT5
MCP tools from `agnt5 mcp` (`create_eval_dataset`, `add_run_to_dataset_draft`, `run_experiment`,
...; setup in the SKILL.md).
`client.getEvents(runId)` returns the journal events of a run (`{ events: [{ eventType, data,
sequence, correlationId }], runId }`) if you need to build dataset `events` yourself.

## Not available in TypeScript

- `AsyncClient` (the `Client` is already async)
- `BatchEvalItem(...)` class and `component=` / `input_data=` keyword arguments (positional)
- Dataset / experiment / report methods on `Client`
- `https://gw.agnt5.com` as the default gateway (TypeScript defaults to localhost)
- Seconds-based `timeout` (TypeScript uses milliseconds everywhere)

## TypeScript pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| `ECONNREFUSED 127.0.0.1:34181` in CI | default gateway is localhost | set `AGNT5_GATEWAY_URL` / `gatewayUrl` |
| Eval hits the wrong deployment | ambient `AGNT5_DEPLOYMENT_ID` ignored for execution | pass `deploymentId` in options |
| `timeout: 60` fails everything instantly | milliseconds, not seconds | `timeout: 60_000` |
| Judge preset calls OpenAI although you set a Claude model | a bare preset `model` means provider `openai` | `model: 'anthropic/<model>'` |
| `config_error` from a built-in such as `contains` | bare name sends no config | `{ name: 'contains', config: { pattern: 'in transit' } }` (`agnt5-scorers`) |
| Run status shows `EXECUTION_ERROR` for every failure | worker collapses error codes | read `item.error` text / worker logs |
| `agnt5 run <function>` exits 1 during a smoke test although the run later passes | CLI prints first failed attempt | check `agnt5 inspect runs describe` |

## Source

https://agnt5.com/docs/improve/batch-eval · /docs/improve/experiments · /docs/improve/datasets (TypeScript tabs)
