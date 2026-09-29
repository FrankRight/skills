# Go testing reference

Verified against github.com/agnt5dev/sdk-go v0.10.3 (`agnt5/context.go`, `lm.go`,
`sandbox.go`, `state.go`, `scorer.go`, `scorer_judge.go`, `eval.go`, `registry.go`,
`agent.go`, `serverless/*.go`, and the package's own `*_test.go`).

## What you cannot construct

`*agnt5.Context` has no exported constructor. The SDK's own tests call the unexported
`newContext(context.Background(), agnt5.Invocation{ID: "run-1", RunID: "run-1",
ComponentType: agnt5.ComponentTypeAgent}, nil, "", agnt5.NewInMemoryStateStore())`, which is
unavailable outside the package. Consequences:

- Handlers registered with `agnt5.RegisterFunction/RegisterWorkflow(w, name, func(*agnt5.Context, In) (Out, error))`
  cannot be invoked from your tests. Keep them one line thick and test the plain function
  they call (`func priceOrder(ctx context.Context, in Order) (Quote, error)`).
- `(*agnt5.Agent).Run(ctx *agnt5.Context, input)` cannot run offline; test agents at level 3
  (`agnt5 dev` + `agnt5 run`) or move the prompt-building logic into plain functions and test
  those with a scripted model.
- `agnt5.Step`/`StepWithKey` need the context too; keep step bodies as separate functions.

## Model doubles

```go
static := agnt5.StaticModel{Content: `{"label":"refund"}`}                 // fixed reply; empty Content echoes the last message
scripted := &agnt5.ScriptedModel{Responses: []agnt5.GenerateResponse{      // replies in order, then falls back to StaticModel{}
    {Content: "", ToolCalls: []agnt5.ToolCall{{ID: "c1", Name: "lookup", Arguments: map[string]any{"order_id": "42"}}}},
    {Content: "Order 42 is in transit."},
}}
resp, err := scripted.Generate(context.Background(), agnt5.GenerateRequest{Messages: msgs})
```

Both satisfy `agnt5.LanguageModel`, so they slot into `agnt5.WithAgentModel(...)` for a
level-3 fake deployment and into any code you write against the interface. Neither
implements `StreamingLanguageModel`.

## Sandbox and state

```go
sb := agnt5.NewInMemorySandbox()
_, _ = sb.WriteFile(ctx, "notes.txt", []byte("hello"))
rf, _ := sb.ReadFile(ctx, "notes.txt")
out, _ := sb.ExecuteCode(ctx, "python", "print(1)")      // echo: Stdout == "[python] print(1)", ExitCode 0
cmd, _ := sb.RunCommand(ctx, []string{"ls", "-la"})      // Stdout == "ls -la"
// also ListFiles(ctx, path), DeleteFile(ctx, path, recursive)

store := agnt5.NewInMemoryStateStore()                   // StateStore: Get(ctx, scope, namespace, key) (any, bool, error), Set, ...
```

`InMemorySandbox` emits the same `sandbox.*` events as a real provider, so code that inspects
`ctx.Events()` in a worker behaves the same.

## Scorers

```go
reg := agnt5.NewScorerRegistry()                          // built-in names resolve here too (Get checks builtins first)
_ = reg.Register(agnt5.ScorerConfig{
    Name: "cites_order", Description: "Reply cites the order id",
    Handler: func(ctx context.Context, req agnt5.ScorerRequest) (agnt5.ScorerResult, error) {
        id, _ := req.Input.(map[string]any)["order_id"].(string)
        ok := id != "" && strings.Contains(fmt.Sprint(req.Output), id)
        return agnt5.ScorerResult{Score: b2f(ok), Passed: ok, Explanation: "order id check"}, nil
    },
})
res, err := reg.Run(ctx, "cites_order", agnt5.ScorerRequest{Output: "Refund for order 42", Input: map[string]any{"order_id": "42"}})
res, err  = reg.Run(ctx, "exact_match", agnt5.ScorerRequest{Output: "a", Expected: "a"})

// built-in judge scorers without a provider
ctx = agnt5.WithLLMJudgeModel(ctx, agnt5.StaticModel{Content: `{"score":1.0,"passed":true,"explanation":"ok"}`})
res, err = reg.Run(ctx, "correctness", agnt5.ScorerRequest{Output: "42", Expected: "42", Input: "6*7"})

// trace assertions
tr := agnt5.TraceScorer(events, []agnt5.TraceAssertion{
    agnt5.MaxTokens(2000), agnt5.MaxLMCalls(4), agnt5.NoErrors(), agnt5.DurationUnder(15*time.Second),
    agnt5.EventSequence([]string{"agent.started", "tool.completed", "agent.completed"}),
    agnt5.StepMemoized("load-order"), agnt5.EventCount("tool.completed", 1),
})
```

`agnt5.RegisterScorer(w, config)` puts a scorer on the worker and in
`agnt5.DefaultScorerRegistry()`; `reg.Clear()` resets custom entries between tests.
`ScorerRequest` fields: `Input`, `Output`, `Expected`, `Trace []TraceEvent`, `Config`,
`PeerScores`, `TraceEvalContext`, `State`, `States`, `StateSnapshots`, `Metadata` (`Events`
is a legacy alias of `Trace`).

## Serverless handler offline

```go
h := serverless.New(serverless.Options{})                 // no SigningSecret -> unsigned accepted
_ = serverless.RegisterWorkflow(h, "hello", hello)
rec := httptest.NewRecorder()
h.ServeHTTP(rec, httptest.NewRequest(http.MethodPost, serverless.InvokePath,
    strings.NewReader(`{"component_type":"workflow","component_name":"hello","run_id":"r1","input":{"name":"Ada"}}`)))
if rec.Code != 200 || !strings.Contains(rec.Body.String(), `"output":{"message":"hello Ada"}`) { t.Fatal(rec.Body.String()) }
```

Resume a suspended workflow by echoing the previous `checkpoint` and adding `metadata`
(`signal_name`/`waiting_step`/`signal_payload`, or `pause_index`/`user_response`) - see
`serverless/parity_test.go` and `agnt5-serverless`.

## Against a running gateway

```go
client, _ := agnt5.NewClient("", agnt5.WithAPIKey(os.Getenv("AGNT5_API_KEY")))   // AGNT5_GATEWAY_URL: localhost:34181 for agnt5 dev up
res, err := client.Run(ctx, "greet", map[string]any{"name": "Ada"}, agnt5.WithWaitTimeout(time.Minute))
if err == nil && res.IsPending() { res, err = client.WaitForResult(ctx, res.RunID, 2*time.Minute, time.Second) }
if err := res.RaiseForStatus(); err != nil { t.Fatal(err) }

ev, err := client.Eval(ctx, agnt5.EvalRequest{Component: "support_agent", ComponentType: agnt5.ComponentTypeAgent,
    Input: map[string]any{"message": "Where is order 42?"}, Expected: "in transit",
    Scorers: agnt5.NormalizeEvalScorers("contains", agnt5.Correctness{})})
```

Guard these with a build tag or `testing.Short()` so `go test ./...` stays offline.
