# Scorers in TypeScript

Verified against `@agnt5/sdk` **0.10.5**. Same section order as the Python SKILL.md. Built-in
scorer names and the `agnt5 experiments` / `agnt5 scores` CLI are language-independent; only
the SDK surface differs.

## Built-in deterministic and LLM-as-judge scorers

Same names, same `--builtin-scorer <name|json>` usage. The TypeScript SDK also exports the
built-ins as plain functions you can call locally against a `ScorerRequest`: `exactMatch`,
`contains`, `regexMatch`, `jsonValid`, `jsonSchema`, `numericRange`, `levenshtein`,
`structuredAssertions`, and async `llmJudge`, `correctness`, `faithfulness`, `goalSuccess`,
`agentJudge` (config through `request.config`, e.g. `{ pattern }`, `{ schema }`, `{ criteria, model }`).

### SDK evaluator presets (for `client.eval()` / `client.batchEval()`)

```typescript
import { Correctness, Helpfulness, Faithfulness, LLMJudge } from '@agnt5/sdk';

const scorers = [
  new Correctness(),
  new Helpfulness({ model: 'openai/gpt-4o' }),
  new Faithfulness({ contextFields: ['retrieved_chunks'] }),
  new LLMJudge({ criteria: 'Is the response under 50 words?', model: 'openai/gpt-4o-mini' }),
];
```

Presets take one config object (`EvaluatorPresetConfig`): `model` (default
`openai/gpt-4o-mini`), `temperature` (default `0`), `threshold` (default `0.7`), `includeInput`
(**default `false` for every preset in TypeScript**, unlike Python), plus field mappings
`answerField`, `referenceField`, `outputField`, `expectedField`, `inputField`, `contextFields`,
`sessionFields`, `journalEventFields`, `metadata`. Full list: `Correctness`, `Faithfulness`,
`Helpfulness`, `Coherence`, `Conciseness`, `ResponseRelevance`, `InstructionFollowing`,
`GoalSuccess`, `Refusal`, `Harmfulness`, `Stereotyping`. `LLMJudge` takes
`{ criteria, model, systemPrompt?, temperature?, includeInput?, promptTemplate?, choiceScores? }`.

## Custom scorers

`scorer(name?, description?, scope = 'item')` returns a decorator you apply to an
`(ctx: ScorerContext, request: ScorerRequest) => ScorerResult | Promise<ScorerResult>` handler.
Import everything from the root `@agnt5/sdk`.

```typescript
import { scorer, ScorerResult } from '@agnt5/sdk';
import type { ScorerContext, ScorerRequest } from '@agnt5/sdk';

export const citesOrderId = scorer('cites_order_id', 'Reply must cite the order ID from the input')(
  async (ctx: ScorerContext, request: ScorerRequest): Promise<ScorerResult> => {
    const orderId = String((request.input as { order_id?: string } | undefined)?.order_id ?? '');
    const cited = orderId !== '' && String(request.output).includes(orderId);
    ctx.log('checked order id', { orderId, cited });
    return new ScorerResult({
      score: cited ? 1 : 0,
      passed: cited,
      explanation: `Order ID ${orderId} ${cited ? 'found' : 'missing'}`,
    });
  },
);
```

- `scope`: `'item' | 'run' | 'trace' | 'span' | 'session' | 'fleet_run'`.
- `ScorerRequest`: `output`, `expected?`, `input?`, `trace?: TraceEvent[]`, `config?`,
  `peer_scores?` and `trace_eval_context?` (snake_case, as sent by the platform).
- `ScorerContext`: `runId`, `correlationId`, `parentCorrelationId?`, `attempt`,
  `log(message, extra?)`.
- `ScorerResult` is a class: `new ScorerResult({ score, passed?, label?, explanation?, metadata? })`,
  `ScorerResult.pass(explanation?)`, `ScorerResult.fail(explanation?)` (not `pass_result`).
- Request helpers are free functions, not methods: `getRequestConfig(request, key, default)`,
  `getToolCalls(request)`, `getToolCallNames(request)`, `getTotalTokens(request)`,
  `getTraceEvents(request, eventType)`, `toolTrajectoryMatches(actual, expected, 'exact' | 'in_order' | 'any_order')`.
- **No `depends_on` and no `ctx.peerScores()`**. Earlier results, when the platform ran other
  scorers first, arrive in `request.peer_scores` (an array of records); ordering is not
  something you can declare from TypeScript.
- Calling `scorer(...)` registers the handler in `ScorerRegistry`; the worker publishes it as a
  `scorer` component on `worker.run()`. Import the module from `app.ts`
  (`import './src/scorers.js'`) or it never registers. After deploy attach by ID:
  `agnt5 experiments create ... --scorer-id <scorer-id>`.

Test locally without deploying:

```typescript
import { runScorer } from '@agnt5/sdk';

const result = await runScorer('cites_order_id', { output: 'Refund for order 42 issued', input: { order_id: '42' } });
console.log(result.score, result.passed, result.explanation);
```

## Trace assertions (glassbox testing)

```typescript
import { scorer, ScorerResult, TraceAssertion, traceScorer } from '@agnt5/sdk';

export const efficiencyCheck = scorer('efficiency_check', 'Token, call and latency budget', 'trace')(
  async (_ctx, request) => {
    const result = traceScorer(request.trace ?? [], [
      TraceAssertion.maxTokens(2000),
      TraceAssertion.maxLmCalls(4),
      TraceAssertion.noErrors(),
      TraceAssertion.durationUnder(15000),
    ]);
    return new ScorerResult({ score: result.score, passed: result.passed, label: result.label, explanation: result.explanation });
  },
);
```

`traceScorer(trace: TraceEvent[], assertions)` — no `ScorerInput` wrapper; pass the events
directly. Assertions: `maxTokens(n)`, `maxLmCalls(n)`, `noErrors()`, `durationUnder(ms)`,
`eventSequence([...])`, `stepMemoized(name)`, `eventCount(type, min)`. Returns
`{ score, passed, label, explanation }` (score = proportion passed). `EvalContext` (with
`getToolCalls()`, `getTotalTokens()`, `toolTrajectoryMatches()`) exists for local eval code but
is not the deployable handler signature.

Trace scorers read journal events (`tool_call.*`, `lm.*`, `workflow.step.*`), which TypeScript
workers emit; the missing OpenTelemetry spans (AGNT5-1320) affect `agnt5 inspect trace`, not
these events.

## Inspect scores

Identical CLI (`agnt5 scores list ...`, `agnt5 scores evidence <score-id> ...`).

## Not available in TypeScript

- `@scorer(depends_on=[...])` and `ctx.peer_scores(name)`
- `ScorerResult.pass_result` / `fail_result` (use `ScorerResult.pass` / `.fail`)
- `ScorerInput`; `request.get_config()` and other request *methods* (free functions instead)
- `agnt5.eval.scorer` legacy registry (there is one `scorer` export; it is the deployable one)
- Python's `include_input=True` default on presets (TypeScript defaults to `false`)

## TypeScript pitfalls

| Symptom | Cause | Fix | Ticket |
|---|---|---|---|
| Scorer not listed after deploy | module never imported by `app.ts` | `import './src/scorers.js'` | — |
| `request.peerScores` is `undefined` | field is `peer_scores` | use `request.peer_scores` | — |
| Judge preset ignores the input | `includeInput` defaults to `false` | pass `{ includeInput: true }` | — |
| Handler returns a plain object and Studio shows no label | works structurally, but `label`/`metadata` easy to drop | return `new ScorerResult({...})` | — |
| `trace` empty for an item | dataset item has no `events` | import items from runs (`agnt5 datasets add-run`) | — |
| `llmJudge` fails with "Model must include provider prefix" | bare `model: 'gpt-4o-mini'` in config | use `openai/gpt-4o-mini` | — |

## Source

https://agnt5.com/docs/improve/scorers (TypeScript tabs)
