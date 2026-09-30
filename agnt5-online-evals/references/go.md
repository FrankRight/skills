# Online evals with a Go worker

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3**. The live-experiment API calls, auth,
rules and results in the SKILL.md are the same for every SDK. Only the scorer code differs.

## 1. Write and deploy the scorer

```go
// Online, req.Input is the run's input and req.Output its output.
// req.Expected and req.Trace are empty.
var citesOrderID = agnt5.ScorerConfig{
    Name:        "cites_order_id",
    Description: "Reply must cite the order ID from the input",
    Scope:       agnt5.ScorerScopeItem,
    Handler: func(_ context.Context, req agnt5.ScorerRequest) (agnt5.ScorerResult, error) {
        input, _ := req.Input.(map[string]any) // Input and Output are `any`
        orderID, _ := input["order_id"].(string)
        output, _ := json.Marshal(req.Output)
        if orderID != "" && strings.Contains(string(output), orderID) {
            return agnt5.PassingScorerResult("Order ID " + orderID + " found"), nil
        }
        return agnt5.FailingScorerResult("Order ID " + orderID + " missing"), nil
    },
}

// in main, next to your other components
if err := agnt5.RegisterScorer(worker, citesOrderID); err != nil {
    log.Fatal(err)
}
```

- Keep `Scope: agnt5.ScorerScopeItem` (the default).
- A built-in name (`exact_match`, `contains`, ...) fails registration with
  `ScorerNameCollisionError`.
- Test it offline with `agnt5.NewScorerRegistry()` and `reg.Run(...)` (`agnt5-testing`).

Deploy (`agnt5-deploy`) and wait until the deployment is Ready: managed Go workers compile when
they start. Then continue with step 2 of the SKILL.md: create the project scorer with
`component_name: "cites_order_id"` and publish its first version with the input requirements.

## Built-ins

`json_valid` and `structured_assertions` need no scorer code. The live experiment still needs a
`scorer_deployment_id`.

## Go pitfalls

- A returned `error` marks the scorer as failed, not as a score of 0. Return
  `FailingScorerResult` for a real failure.
- `req.Expected` is nil online. A scorer that compares against it fails every sampled run.
