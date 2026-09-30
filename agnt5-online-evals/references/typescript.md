# Online evals with a TypeScript worker

Verified against `@agnt5/sdk` **0.10.5**. The live-experiment API calls, auth, rules and results
in the SKILL.md are the same for every SDK. Only the scorer code differs.

## 1. Write and deploy the scorer

```typescript
import { scorer, ScorerResult } from '@agnt5/sdk';
import type { ScorerContext, ScorerRequest } from '@agnt5/sdk';

// Online, request.input is the run's input and request.output its output.
// request.expected and request.trace are empty.
export const citesOrderId = scorer('cites_order_id', 'Reply must cite the order ID from the input')(
  async (ctx: ScorerContext, request: ScorerRequest): Promise<ScorerResult> => {
    const orderId = String((request.input as { order_id?: string } | undefined)?.order_id ?? '');
    const cited = orderId !== '' && JSON.stringify(request.output ?? '').includes(orderId);
    ctx.log('checked order id', { orderId, cited });
    return new ScorerResult({
      score: cited ? 1 : 0,
      passed: cited,
      explanation: `Order ID ${orderId} ${cited ? 'found' : 'missing'}`,
    });
  },
);
```

- `scorer(name, description?, scope = 'item')`: keep the default `'item'` scope.
- Import the module from `app.ts` (`import './src/scorers.js'`) so it registers; the worker
  publishes it as a `scorer` component on `worker.run()`.
- Do not reuse a built-in name (`exact_match`, `contains`, ...): registration throws.
- Check it offline with `runScorer('cites_order_id', { input: { order_id: '42' }, output: { reply: 'Refund for order 42 issued' } })`
  (`agnt5-testing`).

Deploy (`agnt5-deploy`), then continue with step 2 of the SKILL.md: create the project scorer with
`component_name: "cites_order_id"` and publish its first version with the input requirements.

## Built-ins

`json_valid` and `structured_assertions` need no scorer code. The live experiment still needs a
`scorer_deployment_id`.

## TypeScript pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Scorer missing from the deployment's components | module never imported | `import './src/scorers.js'` in `app.ts` |
| Every sampled run fails the scorer | it compares against `request.expected`, which is empty online | score from `request.input` and `request.output` only |
