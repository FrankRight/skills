# Go experiments and inline evals

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3**. Datasets, experiments, reports,
rescoring and annotations are CLI/API operations and work the same for Go workers; this file
covers the SDK half (`client.Eval` / `client.BatchEval`, which the product docs wrongly say
Go lacks) and the Go-specific deployment timing.

## Concept map

| Python (SKILL.md) | Go |
|---|---|
| `Client()` | `client, err := agnt5.NewClient("", agnt5.WithAPIKey(os.Getenv("AGNT5_API_KEY")))` — `""` means `AGNT5_GATEWAY_URL`, then `https://gw.agnt5.com` |
| `client.eval(component=..., component_type=..., input_data=..., expected=..., scorers=[...])` | `client.Eval(ctx, agnt5.EvalRequest{Component, ComponentType, Input, Expected, Scorers}) (*agnt5.EvalResponse, error)` |
| `client.batch_eval(component, items=[BatchEvalItem(...)], scorers, max_concurrency, timeout)` | `client.BatchEval(ctx, component, []agnt5.BatchEvalItem{...}, agnt5.BatchEvalOptions{...}) *agnt5.BatchEvalResult` |
| `BatchEvalItem(input, expected, item_id)` | `agnt5.BatchEvalItem{Input map[string]any, Expected any, ItemID string, Index *int}` |
| `deployment_id=` | `BatchEvalOptions.DeploymentID` / `agnt5.WithClientDeploymentID(id)` |
| scorer strings, presets, `LLMJudge(...)` mixed | `agnt5.NormalizeEvalScorers("exact_match", agnt5.Correctness{}, agnt5.NewLLMJudge(...))` → `[]agnt5.EvalScorerSpec` |

## Single eval

```go
client, err := agnt5.NewClient("", agnt5.WithAPIKey(os.Getenv("AGNT5_API_KEY")))
if err != nil { log.Fatal(err) }

res, err := client.Eval(ctx, agnt5.EvalRequest{
    Component:     "support_agent",
    ComponentType: agnt5.ComponentTypeAgent, // default "" routes as a function
    Input:         map[string]any{"message": "Where is my order #1234?"},
    Expected:      "Your order #1234 is in transit",
    Scorers:       agnt5.NormalizeEvalScorers(agnt5.Correctness{}),
})
if err != nil { log.Fatal(err) }                 // transport / HTTP error
if err := res.RaiseForStatus(); err != nil { … } // res.Error (component or scorer failure)
fmt.Println(res.Passed, string(res.Output))     // Output is json.RawMessage; res.DecodeOutput(&v)
if s, ok := res.GetScore("correctness"); ok {
    fmt.Println(s.Score, s.Passed, s.Explanation)
}
```

`EvalResponse`: `Output`, `Scores []agnt5.EvalScore{Scorer, Score, Passed, Explanation, Label,
Metadata}`, `Passed`, `RunID`, `TraceID`, `DurationMS`, `Error *agnt5.EvalError`, `Raw`.
Run options apply: `agnt5.WithRunTimeout(d)`, `agnt5.WithRunHeader(...)`.

## Batch eval

```go
items := []agnt5.BatchEvalItem{
    {Input: map[string]any{"message": "Where is my order #1234?"}, Expected: "...", ItemID: "order-status"},
    {Input: map[string]any{"message": "Cancel order #5678"}, Expected: "...", ItemID: "order-cancel"},
}
result := client.BatchEval(ctx, "support_agent", items, agnt5.BatchEvalOptions{
    ComponentType:  agnt5.ComponentTypeAgent,
    Scorers:        agnt5.NormalizeEvalScorers("exact_match", agnt5.Correctness{}),
    MaxConcurrency: 3,                 // default 10; start at 3–5 in development
    Timeout:        60 * time.Second,  // per item
    // DeploymentID: "<id>", Expected: []any{...} (positional fallback when item.Expected is nil)
})

fmt.Printf("Pass rate: %.0f%%\n", result.PassRate()*100)
for _, item := range result.Results {
    fmt.Println(item.ItemID, item.Passed, item.DurationMS, item.Error)
}
for _, item := range result.FailedItems() {  // evaluation errors (item.Error != "")
    fmt.Println("error:", item.ItemID, item.Error)
}
for _, item := range result.FailingItems() { // ran fine, scored below threshold
    fmt.Println("fail:", item.ItemID)
}
```

- `BatchEval` returns `*BatchEvalResult` and **no error**: per-item transport errors land in
  `Results[i].Error`. Check `result.Status` (`completed` / `partial_failure` / `failed`) or
  `IsSuccess()` / `IsPartialFailure()`.
- `BatchEvalResult`: `BatchID`, `Status`, `Results` (sorted by `Index`), `Stats{TotalItems,
  CompletedItems, FailedItems, PassedItems, AvgDurationMS, DurationMS}`, methods `PassRate()`,
  `PassingItems()`, `FailingItems()`, `FailedItems()`, `Outputs()`.
- `agnt5.NormalizeBatchEvalItems(inputs []map[string]any, expected []any)` builds items from
  parallel slices (the Python "plain dicts + expected list" form).
- Scorer/judge naming rules and the built-in judge's env keys: `agnt5-scorers`.

## Platform experiments with a Go worker

Everything in the SKILL.md (`agnt5 datasets ...`, `agnt5 experiments create/run`, CI gate exit
codes, regression datasets, rescoring) is unchanged. Go-specific timing:

1. Managed Go workers compile at pod start (`go mod download` + `go build ./...`), so a fresh
   deployment — and the worker an experiment run spins up — is slow to become Ready. Run
   `go mod tidy && go build ./...` locally first; a stale `go.sum` fails the deploy.
2. Wait for `agnt5 deployment status --watch` to show Ready before `agnt5 experiments run` or
   `agnt5 run --env`, otherwise items fail with connection errors rather than bad scores.
3. Custom scorers (`RegisterScorer`) ship in the same build; attach them by ID after the deploy
   is Ready. Built-in judge scorers run in the Go worker and need the provider key set as a
   secret (`agnt5 secrets set --name OPENAI_API_KEY --type api_key`).
4. Trace-level scorers need `events` on dataset items — capture them with
   `agnt5 datasets add-run` from a run produced by the Go worker.

## Not available in Go

Dataset/experiment management from the SDK (CLI or MCP only), item input as bare dicts with
`input`/`expected` keys (build `BatchEvalItem`s), `AsyncClient` (the Go client is already
context-based).

## Go pitfalls for this skill

- `Expected` is positional when passed through `BatchEvalOptions.Expected`: keep it aligned
  with `items` or set `item.Expected`.
- `ComponentType` left empty routes the eval to `/v1/functions/...`; set it for agents and
  workflows.
- Raw `EvalScorerSpec` judge configs need bare model names; presets accept `provider/model`
 .
- Run/stream calls wait up to 5 minutes by default; for long-running components set
  `Timeout` (it becomes `WithRunTimeout`, the HTTP deadline for that item) and lower
  concurrency.
