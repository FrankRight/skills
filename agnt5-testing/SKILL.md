---
name: agnt5-testing
description: Test AGNT5 components at every level - unit-test @function/@workflow/agent/tool/scorer code in-process without a worker (Python direct calls with FunctionContext, a LanguageModel fake, InMemorySandbox, run_scorer; TypeScript ContextImpl, the fn().run() no-context fallback, a LanguageModel fake for Agent, InMemorySandbox, MemoryStateAdapter, runScorer, registry clear(); Go StaticModel/ScriptedModel, InMemorySandbox, InMemoryStateStore, ScorerRegistry, WithLLMJudgeModel, httptest for serverless handlers), then run end to end against agnt5 dev, a deployed environment (agnt5 run --env, Client.eval/batch_eval) and regression datasets with a CI gate. Use when writing pytest/vitest/go tests for an AGNT5 project, mocking a model, checking a deployed component before promotion, or when agnt5 run reports a failure that later succeeded.
---

# AGNT5 Testing

Test the logic in-process first, then the component against a real runtime, then the
deployed build. Each level catches a different failure: wrong code, wrong registration or
schema, wrong environment.

| Level | What runs | Tool |
|---|---|---|
| 1. Pure logic | Your functions, no SDK context | pytest / vitest / `go test` |
| 2. Component in-process | Handler with a local context, fake model, in-memory sandbox/state; scorers by name; serverless handlers over HTTP | SDK test helpers (below) |
| 3. Local end to end | Real registration, schema, retries, tracing | `agnt5 dev` + `agnt5 run`; `agnt5 dev up` + `Client` |
| 4. Deployed smoke | The exact build that will serve traffic | `agnt5 run --env`, `Client`, `client.eval` |
| 5. Regression gate | Curated dataset, scored, pass-rate threshold | `agnt5 experiments run --wait --fail-on-gate` (`agnt5-experiments`) |
| 6. Production quality | Sampled live runs scored by a live experiment | `agnt5-online-evals` |

## Level 2: components without a worker

What is **not** testable offline: platform retries/backoff, durable step replay, HITL pauses
on worker contexts, cron/webhook triggers, and Go agent runs (see Go notes). Test those at
level 3.

### Python (`references/python.md`)

```python
import pytest
from agnt5 import FunctionContext, ScorerRequest, run_scorer
from app import greet, onboarding_workflow            # @function / @workflow

async def test_greet():                               # asyncio_mode = auto in pytest.ini
    assert await greet(name="Ada") == "Hello, Ada!"   # no ctx param: a local FunctionContext is created

async def test_greet_with_ctx():                      # handler declared as (ctx, name)
    ctx = FunctionContext(run_id="test", correlation_id="corr", parent_correlation_id="parent")
    assert await greet(ctx, name="Ada") == "Hello, Ada!"

async def test_workflow():
    result = await onboarding_workflow(user_email="ada@example.com")   # keyword args ONLY; positional args are dropped
    assert result["status"] == "done"                 # steps run inline, state is in-memory, no replay

async def test_scorer():
    r = await run_scorer("cites_order_id", ScorerRequest(output="Refund for order 42", input={"order_id": "42"}))
    assert r.passed
```

Fake a model by subclassing `agnt5.lm.LanguageModel` and passing the instance as
`Agent(model=...)`. Implement both methods: `Agent.run()` calls `stream()` when the agent has no
tools (the final text comes from the `LMCompleted` event) and `generate()` when it has tools. If
`stream()` does not yield an `LMCompleted` with the text, `agent.run()` returns `''`. Working
fake: `references/python.md`.
`InMemorySandbox()` backs sandbox-using tools with a file map and echo execution. `timeout_ms`
is enforced locally; `retries` are not.

### TypeScript (`references/typescript.md`)

```typescript
import { beforeEach, expect, it } from 'vitest';
import { ContextImpl, FunctionRegistry, WorkflowRegistry, runScorer } from '@agnt5/sdk';
import { greet, onboarding } from '../src/app.js';     // fn('greet').run(...) / workflow('onboarding', ...)

beforeEach(() => { FunctionRegistry.clear(); WorkflowRegistry.clear(); });   // names are global

it('runs handlers with a local context', async () => {
  const ctx = new ContextImpl('inv-1', 'run-1', 0, 'test-service', { storage: 'memory' });
  expect(await greet(ctx, 'Ada')).toBe('Hello, Ada!');            // fn wrapper calls the raw handler
  expect(await onboarding(ctx, { email: 'ada@example.com' })).toMatchObject({ status: 'done' });
});

it('scores', async () => {
  expect((await runScorer('exact_match', { output: 'a', expected: 'a' })).passed).toBe(true);
});
```

`new Agent({ model: fakeModel, ... })` accepts any `LanguageModel`: an object with
`generate(request): Promise<GenerateResponse>` that returns `{ text, finishReason?, toolCalls? }`
(the root `GenerateResponse` has no `id` or `model`; adding them fails `tsc` with TS2353).
Working fake: `references/typescript.md`. `InMemorySandbox`, `MemoryStateAdapter` +
`StateManager` cover sandbox and state. Type-check with `npx tsc --noEmit`; run `npx vitest run`.

### Go (`references/go.md`)

```go
func TestPriceOrder(t *testing.T) {                     // keep logic in plain funcs: no exported *agnt5.Context constructor
    got, err := priceOrder(context.Background(), order)  // the handler is a thin wrapper around this
    ...
}
func TestPrompt(t *testing.T) {
    model := &agnt5.ScriptedModel{Responses: []agnt5.GenerateResponse{{Content: `{"label":"refund"}`}}}
    resp, err := model.Generate(context.Background(), agnt5.GenerateRequest{Messages: msgs})
    ...
}
func TestScorer(t *testing.T) {
    reg := agnt5.NewScorerRegistry()
    _ = reg.Register(agnt5.ScorerConfig{Name: "cites_order", Handler: citesOrder})
    res, err := reg.Run(context.Background(), "cites_order", agnt5.ScorerRequest{Output: "order 42", Input: map[string]any{"order_id": "42"}})
    ...
}
```

`agnt5.Agent.Run` needs a `*agnt5.Context`, which only the worker (and the package's
unexported `newContext`) can build - agents run at level 3. Serverless workflows are the
exception: drive `serverless.Handler` with `httptest` (`agnt5-serverless`).

### Serverless handlers offline (all three SDKs)

No signing secret configured means unsigned invokes are accepted: Python
`await app.handle_http(method="POST", path="/agnt5/invoke", headers={}, body=...)`,
TypeScript `await handler.fetch(new Request('https://x/agnt5/invoke', { method: 'POST', body }))`,
Go `handler.ServeHTTP(rec, httptest.NewRequest(http.MethodPost, serverless.InvokePath, body))`.
Feed the returned `checkpoint` back to test replay and resume (`agnt5-serverless` references).

## Level 3: local end to end

```bash
agnt5 dev                                   # worker connected to your AGNT5 environment, hot reload
agnt5 components                            # what registered, by type
agnt5 run greet --input '{"name":"Ada"}'                       # function (streams) - routes to the dev worker
agnt5 run onboarding --type workflow --input '{"email":"ada@example.com"}'
agnt5 run support_agent --type agent --input '{"message":"hi"}'
agnt5 inspect runs ls                       # then agnt5-observe for traces and logs
```

`agnt5 run` without `--env` targets the `agnt5 dev` worker. For SDK clients use the container
stack: `agnt5 dev up` then `AGNT5_GATEWAY_URL=http://localhost:34181` (`agnt5-client`).
Provider keys come from `.env` (`agnt5-project-init`); Python HITL pauses can be answered in
Studio while the dev worker runs.

## Level 4: smoke-test the deployed build

```bash
agnt5 deployment list                  # ID of the deployment you just shipped
agnt5 run onboarding --type workflow --input '{"email":"ada@example.com"}' --deployment-id <id>
agnt5 run onboarding --type workflow --input '{...}' --deployment-id <id> --timeout 5m   # client-side limit; run continues
```

Target the deployment by ID. `--env preview` currently answers 409 "environment has no active
deployment" even when a preview deployment is running.

From code (`agnt5-client`): `Client(deployment_id=...)` / `new Client({ deploymentId })` /
`agnt5.WithClientDeploymentID`, then `client.run(...)`, or score in one call:

```python
from agnt5 import Client
from agnt5.eval import Correctness
client = Client()
r = client.eval(component="support_agent", component_type="agent", deployment_id=candidate,
                input_data={"message": "Where is order #1234?"}, expected="in transit",
                scorers=[{"name": "contains", "config": {"pattern": "in transit"}},
                         Correctness(model="openai/gpt-4o-mini")])
assert r.passed, [s.explanation for s in r.scores]
```

`client.batch_eval(...)` / `client.batchEval(...)` / Go `client.BatchEval(...)` run a list
with `max_concurrency` (start at 3-5). Deterministic scorers first; judge scorers cost money
and must not run on gpt-6 models. A bare `"contains"` sends no `pattern`, and Python and
TypeScript workers score it as `config_error`; the built-ins that need config are listed in
`agnt5-scorers`.

## Level 5: regression datasets and CI gates

Turn a good run into a test case (`agnt5 datasets add-run <dataset-id> <run-id>`), publish a
version, create an experiment with scorers, and run it per candidate:

```bash
agnt5 experiments run <experiment-id> --deployment-id "$CANDIDATE" --name "ci-$GIT_SHA" --wait --fail-on-gate
agnt5 experiments runs regression-dataset <run-id> --name onboarding-regressions --start-run --wait
```

Exit code 2 = gate failed, 3 = run failed, 4 = timeout. Full flow in `agnt5-experiments`.

## Pitfalls

- **`agnt5 run <function>` can report a failure that is still being retried**:
  the streaming function path prints the first failed attempt as the final result while the
  platform keeps retrying. Confirm the real outcome with `agnt5 inspect runs` / Studio, or
  poll `client.wait_for_result(run_id)`; wrap retried functions in a workflow for CLI smoke
  tests.
- Workflow direct calls in Python take keyword arguments only and skip replay - do not assert
  on checkpoint behaviour offline.
- HITL workflows re-run from the top on resume; anything before the pause must be a step
  (`agnt5-human-in-the-loop`). Do not wrap the pause in `except BaseException`.
- Component names are global per process: clear TypeScript registries between tests and
  avoid re-decorating the same Python name in fixtures.
- Model tests need provider keys: mark them (`@pytest.mark.llm`, a vitest `describe.skipIf`)
  and skip without credentials; keep `temperature` off gpt-6 models (`agnt5-models`).
- `agnt5 run --timeout` and `Client(timeout=...)` bound the wait, not the run; a run that
  outlives them keeps executing and still counts toward retries.

## Source

https://agnt5.com/docs/build/local-development · https://agnt5.com/docs/cli/run · https://agnt5.com/docs/improve/batch-eval · https://agnt5.com/docs/improve/scorers
