# Go scorers

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3**. The product docs claim Go has no
evaluator presets, no batch eval and no trace assertions — all three exist; write from the API
below. CLI flags (`--builtin-scorer`, `agnt5 scores ...`) are identical for Go workers.

## Concept map

| Python (SKILL.md) | Go |
|---|---|
| `@scorer(name=..., scope=..., depends_on=[...])` | `agnt5.RegisterScorer(worker, agnt5.ScorerConfig{Name, Description, Handler, Scope, DependsOn, InputSchema, Metadata})` |
| `(ctx: ScorerContext, request: ScorerRequest) -> ScorerResult` | `Handler: func(ctx context.Context, req agnt5.ScorerRequest) (agnt5.ScorerResult, error)` (`agnt5.ScorerHandler`) |
| `ScorerResult.pass_result / fail_result` | `agnt5.PassingScorerResult(why)`, `agnt5.FailingScorerResult(why)`, `agnt5.NewScorerResult(score, why)` (clamps, `Passed` at ≥ 0.5) |
| `request.get_tool_calls()` / `get_tool_call_names()` | `agnt5.ExtractToolCallsFromEvents(req.Trace)`, `agnt5.ToolCallNames(calls)` |
| `trace_scorer(ScorerInput(...), [TraceAssertion.max_tokens(n), ...])` | `agnt5.TraceScorer(req.Trace, []agnt5.TraceAssertion{agnt5.MaxTokens(n), ...})` |
| `Correctness()`, `Helpfulness(model=...)`, `LLMJudge(criteria=...)` | `agnt5.Correctness{}`, `agnt5.Helpfulness{EvaluatorPresetConfig: agnt5.EvaluatorPresetConfig{Model: "openai/gpt-4o"}}`, `agnt5.NewLLMJudge(agnt5.LLMJudgeConfig{Criteria: ...})` |
| `run_scorer(name, request)` | `agnt5.DefaultScorerRegistry().Run(ctx, name, req)` |

## Custom scorer

```go
citesOrderID := agnt5.ScorerConfig{
    Name:        "cites_order_id",
    Description: "Reply must cite the order ID from the input",
    Scope:       agnt5.ScorerScopeItem, // Item (default) | Run | Trace | Span | Session | FleetRun
    Handler: func(c context.Context, req agnt5.ScorerRequest) (agnt5.ScorerResult, error) {
        input, _ := req.Input.(map[string]any) // Input/Output/Expected are `any`
        orderID, _ := input["order_id"].(string)
        output := fmt.Sprint(req.Output)
        if m, ok := req.Output.(map[string]any); ok && m["response"] != nil {
            output = fmt.Sprint(m["response"]) // an agent's reply; its messages include the system prompt
        }
        if orderID != "" && strings.Contains(output, orderID) {
            return agnt5.PassingScorerResult("Order ID " + orderID + " found"), nil
        }
        return agnt5.FailingScorerResult("Order ID " + orderID + " missing"), nil
    },
}
must(agnt5.RegisterScorer(worker, citesOrderID))
```

- `ScorerRequest` fields: `Input`, `Output`, `Expected` (all `any`), `Trace []agnt5.TraceEvent`,
  `Config map[string]any`, `PeerScores []map[string]any`, `TraceEvalContext`, `State`,
  `States`, `StateSnapshots`, `Metadata`. `Events` is a deprecated alias for `Trace`.
- `ScorerResult{Score float64, Passed bool, Explanation, Label string, Metadata map[string]any}`.
  Returning a Go `error` marks the scorer as failed (not "scored 0").
- When the worker dispatches the scorer, `c` is the run's `*agnt5.Context`
  (`if ctx, ok := c.(*agnt5.Context); ok { ctx.Logger().Info(...) }`).
- `DependsOn: []string{"other_scorer"}` + read `req.PeerScores` (one map per earlier result,
  with `ScorerResult`'s JSON keys — exact keys were not verified, log one first).
- The scorer deploys with the worker like any component (type `scorer`), but deploying does not
  create a project scorer. Get a scorer ID with MCP `create_scorer` (`type: "deployed"`,
  `deployment_id`, `component_name: "cites_order_id"`) and `publish_scorer_version`, then
  `agnt5 experiments create ... --scorer-id <scorer-id>` (steps in the SKILL.md). A component ID
  is accepted at create and fails at `experiments run` with 404.

Test locally without a worker:

```go
reg := agnt5.NewScorerRegistry() // fresh registry; built-in names resolve here too
must(reg.Register(citesOrderID))
result, err := reg.Run(context.Background(), "cites_order_id", agnt5.ScorerRequest{
    Output: "Refund for order 42 issued", Input: map[string]any{"order_id": "42"},
})
```

## Built-in deterministic scorers

`agnt5.BuiltInDeterministicScorerNames` is the full CLI list (`exact_match`, `contains`,
`regex_match`, `json_valid`, `json_schema`, `numeric_range`, `levenshtein`,
`structured_assertions`, `tool_called`, ..., `state_equals`). Platform experiments run them in
the AGNT5 runtime; `client.Eval` sends them to your worker, and the Go worker implements every
one. Config keys mirror the CLI JSON and the required-config table in the SKILL.md (`contains` →
`pattern`, `json_schema` → `schema`, `duration_under` → `max_ms`, ...). Without `pattern`, the Go
`contains` and `regex_match` fall back to `Expected`, while Python and TypeScript workers return a
config error, so always pass it:
`agnt5.EvalScorerSpec{Name: "contains", Config: map[string]any{"pattern": "in transit"}}`.
Ready-made `ScorerConfig`s you can register or call: `agnt5.ExactMatchScorer()`,
`agnt5.ContainsScorer()`.

`agnt5.StructuredAssertions(req)` runs the shared assertion language directly:

```go
result := agnt5.StructuredAssertions(agnt5.ScorerRequest{
    Output: map[string]any{"status": "ok", "items": []any{1, 2}},
    Config: map[string]any{
        "assertions": []any{
            map[string]any{"name": "ok", "expr": `output.status == "ok"`},
            map[string]any{"name": "has_items", "expr": `size(output.items) >= 1 && all(output.items, is_number)`},
        },
        "score_threshold": 1.0, // default 1.0; 1..64 assertions
    },
})
```

Expression roots: `input`, `output`, `expected` (and `input_json`/`output_json`/`expected_json`
to parse a JSON string first); `.field` selectors, JSON literals, `== != < <= > >= && || !`,
functions `size(x)`, `unique(x)`, `is_string/is_number/is_boolean/is_null/is_array/is_object(x)`,
`all(x, is_number)`, `any(x, is_string)`. Whitespace includes vertical tab (unlike the Rust
implementation). Score = passed/total; `Passed` when score ≥ threshold.

## Trace assertions (glassbox)

```go
efficiency := agnt5.ScorerConfig{
    Name: "efficiency_check", Scope: agnt5.ScorerScopeTrace,
    Handler: func(_ context.Context, req agnt5.ScorerRequest) (agnt5.ScorerResult, error) {
        return agnt5.TraceScorer(req.Trace, []agnt5.TraceAssertion{
            agnt5.MaxTokens(2000),
            agnt5.MaxLMCalls(4),
            agnt5.NoErrors(),
            agnt5.DurationUnder(15 * time.Second), // time.Duration, not ms
            agnt5.EventSequence([]string{"lm.completed", "tool_call.completed"}),
            agnt5.StepMemoized("fetch_user"),
            agnt5.EventCount("tool_call.completed", 1),
        }), nil
    },
}
```

`TraceScorer` returns score = fraction passed, `Passed` only when all pass, and lists failures
in `Explanation`. `assertion.Check(trace)` gives one `AssertionResult`. Tool trajectories:
`calls := agnt5.ExtractToolCallsFromEvents(req.Trace)`; `agnt5.ToolCallNames(calls)`;
`agnt5.ToolTrajectoryMatches(actual, expected, agnt5.ToolTrajectoryInOrder)` (`Exact`,
`InOrder`, `AnyOrder`). `agnt5.EvalContext{Events: req.Trace}` bundles the same helpers as
methods (`LMCalls()`, `TotalTokens()`, `StepEvents(name)`).

## LLM-as-judge presets and specs (for `client.Eval`/`BatchEval`)

```go
scorers := agnt5.NormalizeEvalScorers(
    "exact_match",                                   // name only
    agnt5.Correctness{},                             // default model openai/gpt-4o-mini, threshold 0.7
    agnt5.Helpfulness{EvaluatorPresetConfig: agnt5.EvaluatorPresetConfig{Model: "openai/gpt-4o"}},
    agnt5.Faithfulness{EvaluatorPresetConfig: agnt5.EvaluatorPresetConfig{ContextFields: []string{"input.retrieved_chunks"}}}, // input., output. or expected.
    agnt5.NewLLMJudge(agnt5.LLMJudgeConfig{Criteria: "Is the response concise and actionable?", Model: "openai/gpt-4o-mini"}),
    agnt5.EvalScorerSpec{Name: "llm_judge", Config: map[string]any{"criteria": "...", "model": "gpt-4.1-mini"}}, // raw: bare name!
)
```

Presets: `Correctness`, `Faithfulness`, `Helpfulness`, `Coherence`, `Conciseness`,
`ResponseRelevance`, `InstructionFollowing`, `GoalSuccess`, `Refusal`, `Harmfulness`,
`Stereotyping` — each embeds `EvaluatorPresetConfig{Model, IncludeInput *bool, Temperature,
Threshold *float64, ContextFields, ...}`. `agnt5.NamedScorer("json_valid")` builds a bare spec.

Model naming rule: typed presets and `NewLLMJudge` split `provider/model` into
`provider` + `model`; a raw `EvalScorerSpec.Config["model"]` is sent to the provider verbatim, so
`"openai/gpt-4.1-mini"` there is a 400 `invalid model ID` — use a bare name (and optionally
`"provider": "anthropic"`). Any gpt-6 judge fails because the judge always sends
`temperature: 0`. The built-in judge runs inside your Go worker and reads its key from
`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GOOGLE_API_KEY`/`GEMINI_API_KEY` (plus Mistral, Groq,
DeepSeek, OpenRouter, Together, Fireworks, Moonshot keys) — set it as a deployment secret.
Tests: `agnt5.WithLLMJudgeModel(ctx, agnt5.StaticModel{Content: `{"score":1,"passed":true,"label":"pass","explanation":"ok"}`})`.

## Not available in Go

`ScorerContext` helpers (`ctx.peer_scores(name)`, `request.get_config/get_total_tokens`),
`agnt5.eval.scorer` legacy registry, `ScorerInput`/`trace_scorer` names (use `TraceScorer`).

## Go pitfalls for this skill

- `req.Input["key"]` does not compile (`Input` is `any`) — the product-docs snippet does this;
  type-assert first.
- Raw judge specs need bare model names; presets accept `provider/model`.
- A scorer registered with a name already taken (including built-in names) fails with
  `ScorerNameCollisionError` at registration.
- Scorers deploy with the worker: after `agnt5 deploy`, wait for the Go build to finish and the
  deployment to be Ready, then create the project scorer against that deployment ID before
  attaching it to an experiment (`agnt5-deploy`).
