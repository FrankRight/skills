# Go client reference

Verified against github.com/agnt5dev/sdk-go v0.10.3 (`agnt5/client.go`, `client_parity.go`,
`batch.go`, `eval.go`, `eval_presets.go`, `chat.go`).

## Construct

```go
client, err := agnt5.NewClient("",                     // "" -> AGNT5_GATEWAY_URL -> https://gw.agnt5.com; "host:port" gets http://
    agnt5.WithAPIKey(os.Getenv("AGNT5_API_KEY")),      // default AGNT5_API_KEY; must start with agnt5_sk_
    agnt5.WithTenantID("acme"),                        // default AGNT5_TENANT_ID
    agnt5.WithClientDeploymentID(""),                  // default AGNT5_DEPLOYMENT_ID
    agnt5.WithClientTimeout(45*time.Second),
    agnt5.WithHTTPClient(&http.Client{}),
)
```

## Methods

```go
Run(ctx, component string, input any, opts ...RunOption) (*RunResponse, error)
Submit(ctx, component, input, opts ...SubmitOption) (*SubmitResponse, error)
GetStatus(ctx, runID) (*StatusResponse, error)          // IsComplete(), IsRunning()
GetResult(ctx, runID) (*RunResponse, error)
WaitForResult(ctx, runID, timeout, pollInterval time.Duration) (*RunResponse, error)
GetEvents(ctx, runID) (*EventsResponse, error)
Stream(ctx, component, input, handle func(string) error, opts ...RunOption) error
StreamEvents(ctx, component, input, handle func(ReceivedEvent) error, opts ...RunOption) error
Workflow(name) *WorkflowProxy                            // Run(ctx, input, opts...), Submit(ctx, input, opts...)
Session(sessionID) *SessionProxy                         // WithUser(userID), Run(ctx, component, input, opts...), Workflow(name).Run(ctx, input, opts...)
Chat(ctx, agent string, msg ChatMessage, opts ...RunOption) (*ChatResponse, error)   // ChatMessage{SessionID, UserID, Role, Content, Metadata}
Batch(ctx, component, items any, opts ...BatchOption) (*BatchResult, error)
Eval(ctx, EvalRequest{Component, ComponentType, Input, Expected, Scorers []EvalScorerSpec, Metadata}, opts ...RunOption) (*EvalResponse, error)
BatchEval(ctx, component, items []BatchEvalItem, BatchEvalOptions{Scorers, Expected, ComponentType, DeploymentID, MaxConcurrency, Timeout}, opts ...RunOption) *BatchEvalResult
ResumeWorkflow(ctx, runID string, userResponse any, opts ...RunOption) (*ResumeWorkflowResponse, error)   // POST /v1/workflows/resume/{id} {"user_response": ...}
CancelRun(ctx, runID, reason string, opts ...RunOption) (*CancelRunResponse, error)
```

Run options: `WithRunComponentType(agnt5.ComponentTypeWorkflow | ComponentTypeAgent |
ComponentTypeTool | ComponentTypeFunction)`, `WithRunSessionID`, `WithRunUserID`,
`WithRunTenant`, `WithWaitTimeout(d)` (gateway wait; default 5 min), `WithRunTimeout(d)`
(HTTP deadline), `WithRunHeader(k, v)`, `WithRunHeaders(map)`, `WithIdempotencyKey(key)`
(alias `WithRunIdempotencyKey`). Submit options: `WithSubmitComponentType`,
`WithSubmitMetadata`, `WithSubmitTenant`, `WithSubmitIdempotencyKey`. Batch options:
`WithBatchComponentType`, `WithBatchMaxConcurrency`, `WithBatchContinueOnFailure`,
`WithBatchTimeoutMS`, `WithBatchDefaultItemTimeoutMS`, `WithBatchMetadata`.

Scorers for `Eval`/`BatchEval`: `[]agnt5.EvalScorerSpec{{Name: "exact_match"}, {Name: "llm_judge",
Config: map[string]any{"criteria": "..."}}}` or `agnt5.NormalizeEvalScorers("exact_match",
agnt5.Correctness{}, agnt5.Helpfulness{EvaluatorPresetConfig: agnt5.EvaluatorPresetConfig{Model: "openai/gpt-4o-mini"}})`.
Presets: `Correctness`, `Faithfulness`, `Helpfulness`, `Coherence`, `Conciseness`,
`ResponseRelevance`, `InstructionFollowing`, `GoalSuccess`, `Refusal`, `Harmfulness`,
`Stereotyping`, and `LLMJudge{...}`. (The public docs say Go lacks batch eval and presets;
the source has both.) `BatchEvalResult`: `PassRate()`, `IsSuccess()`, `IsPartialFailure()`,
`Outputs()`, `PassingItems()`, `FailingItems()`, `FailedItems()`, `Stats`.

## Responses and errors

```go
type RunResponse struct {
    RunID string; StatusCode int; Status RunStatus; Output json.RawMessage; Error *RunErrorDetail
    DurationMS *int64; TraceID, Component, SessionID string; CreatedAt, StartedAt, CompletedAt, FailedAt *time.Time
    Metadata map[string]any; Raw map[string]any
}
// IsSuccess() IsPending() IsError() DecodeOutput(&v) RaiseForStatus() -> *RunError
type RunError struct { Message, RunID, ErrorCode string; Attempts, MaxAttempts int; Metadata map[string]any }   // WasRetried() ExhaustedRetries()
type ClientError struct { Method, URL string; StatusCode int; Body string }                                        // non-run HTTP failures
```

`RunStatus` constants: `agnt5.RunStatusPending ... RunStatusAwaitingUserInput, RunStatusTimeout,
RunStatusUnknown`. `WaitForResult` returns a synthetic `timeout` response when the deadline
passes; the run keeps executing. `DecodeOutput` returns `io.EOF` for a missing/null output.

A human-in-the-loop question and a durable sleep both report `agnt5.RunStatusPaused`
(`RunStatusAwaitingUserInput` is not reported). `Run` returns at the first pause with
`IsPending() == true` and `RaiseForStatus() == nil`; `StatusResponse.IsComplete()` counts
`paused` as finished, so `WaitForResult` returns at once with `Status == RunStatusPaused`.
Branch on `res.Status` before decoding output. `GetEvents` items keep `Data` and `Metadata`.

## Patterns

```go
sub, err := client.Submit(ctx, "generate_report", map[string]any{"report_id": id},
    agnt5.WithSubmitComponentType(agnt5.ComponentTypeWorkflow), agnt5.WithSubmitIdempotencyKey("report:"+id))
res, err := client.WaitForResult(ctx, sub.RunID, 10*time.Minute, 2*time.Second)

// resume a paused workflow with an approval decision (the key needs the `workflow` scope).
// Check first that the newest workflow.paused is a question, not a durable sleep:
// pendingQuestion() in agnt5-human-in-the-loop references/go.md.
_, err = client.ResumeWorkflow(ctx, runID, "approve")
res, err = client.WaitForResult(ctx, runID, 5*time.Minute, time.Second) // returns at the next pause too
_, err = client.CancelRun(ctx, runID, "operator stop")                  // {"reason": ...}; `workflow` scope too

// stream an agent reply
err = client.Stream(ctx, "support_agent", map[string]any{"message": "hi"}, func(chunk string) error {
    fmt.Print(chunk); return nil
}, agnt5.WithRunComponentType(agnt5.ComponentTypeAgent), agnt5.WithRunSessionID("s1"))
```

There is no `SendSignal` method; POST the signal yourself (`workflow`-scoped key):

```go
func sendSignal(ctx context.Context, runID, name string, payload any) error {
    body, err := json.Marshal(map[string]any{"payload": payload})
    if err != nil {
        return err
    }
    req, err := http.NewRequestWithContext(ctx, http.MethodPost,
        os.Getenv("AGNT5_GATEWAY_URL")+"/v1/runs/"+runID+"/signals/"+name, bytes.NewReader(body))
    if err != nil {
        return err
    }
    req.Header.Set("X-API-KEY", os.Getenv("AGNT5_API_KEY"))
    req.Header.Set("Content-Type", "application/json")
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    if resp.StatusCode != http.StatusOK { // 200 {"run_id", "signal_id", "resumed", ...}
        return fmt.Errorf("signal %s: %s", name, resp.Status)
    }
    return nil
}
```

`resumed` in the response is `true` when the signal woke a run paused on it.
