# Go serverless reference

Verified against github.com/agnt5dev/sdk-go v0.10.3 (`serverless/serverless.go`,
`components.go`, `suspension.go`, `payload.go`) and the September 2026 CLI.

The package is `github.com/agnt5dev/sdk-go/serverless`, separate from `agnt5`. It has no
framework dependencies and no worker connection; `*serverless.Handler` implements
`net/http.Handler`.

## Handler and registration

```go
handler := serverless.New(serverless.Options{
    ServiceName:    "orders-api",
    ServiceVersion: os.Getenv("K_REVISION"),                         // or GIT_SHA
    SigningSecret:  func(*http.Request) string { return os.Getenv("AGNT5_SERVERLESS_SIGNING_SECRET") },
    Enabled:        func(*http.Request) bool { return true },        // false -> 503 WORKERLESS_DISABLED
    HTTPClient:     nil,                                              // used for input_ref / output_upload transfers
})

err := serverless.RegisterWorkflow(handler, "hello", func(ctx *serverless.Context, in helloInput) (helloOutput, error) { ... })
err  = serverless.RegisterFunction(handler, "normalize", func(ctx *serverless.Context, in Order) (Order, error) { ... })
err  = serverless.RegisterTool(handler, serverless.Tool{
    Name: "order-status", Description: "Look up an order", Schema: map[string]any{"type": "object"},
    Handler: func(ctx context.Context, args map[string]any) (any, error) { return lookup(ctx, args["order_id"]) },
})
err  = serverless.RegisterAgent(handler, serverless.Agent{
    Name: "support", Description: "Support agent",
    Run: func(ctx *serverless.Context, in serverless.AgentInput) (serverless.AgentResult, error) {
        // in.Message (also accepts prompt/input keys), in.SessionID, in.History []serverless.Message{Role, Content}
        return serverless.AgentResult{Output: "..."}, nil   // Messages optional; history is checkpointed for you
    },
})
```

Registration returns an error for empty names, nil handlers, and duplicate `kind:name`.
Inputs are decoded with `encoding/json` into `In`; outputs are JSON-encoded. The manifest is
built from registrations (sorted by type then name) and served at `serverless.ManifestPath`;
invokes go to `serverless.InvokePath`. Constants: `ProtocolVersion`, `SignatureVersion`.
Body limit 8 MiB; referenced payloads up to 64 MiB.

Routers: `mux.Handle("/.well-known/agnt5", handler)` + `mux.Handle("/agnt5/invoke", handler)`;
Chi `router.Method(http.MethodGet, ...)` / `Method(http.MethodPost, ...)`; Gin `gin.WrapH(handler)`;
Echo `echo.WrapHandler(handler)`. Any other path returns `404 WORKERLESS_NOT_FOUND`.

## `*serverless.Context`

Embeds `context.Context` (the request context), so pass `ctx` straight into HTTP and model
calls. Exported fields: `RunID`, `Attempt`, `ComponentName`, `Metadata map[string]string`.

| Call | Behavior |
|---|---|
| `serverless.Step[T](ctx, name, func(context.Context) (T, error))` | Replays `step:<name>` from the checkpoint (JSON into `T`); otherwise runs and stores it |
| `ctx.Sleep(d time.Duration, name string)` | Stores the start in a step; returns `*Suspension{Reason: "timer"}` until ready. Empty name becomes `sleep_<ms>ms` |
| `ctx.YieldIfNeeded() error` | Returns `*Suspension{Reason: "budget"}` when within the yield margin of the deadline |
| `serverless.WaitForSignal[T](ctx, signalName, waitingStep string) (T, error)` | Returns the delivered payload decoded into `T` (string fallback) or `*Suspension{Reason: "signal"}` |
| `ctx.WaitForUser(serverless.UserInput{Question, Type, Options []UserInputOption{ID, Label, Description}, AllowCustom, Skippable}) (string, error)` | Returns the answer (`""` when skipped; `__custom__:` prefix stripped) or `*Suspension{Reason: "user_input_required"}` |
| `ctx.Emit(serverless.Event{EventType, Data, Metadata})` | Appended to the response `events` |

A `*Suspension` is an `error` - **return it unchanged**. Wrapping it with `fmt.Errorf("%w")`
still works (`errors.As`), but swallowing it turns a suspension into a failed run. There is
no state map, no `Generate`, and no `StepWithKey`; build unique step names yourself when
looping (`fmt.Sprintf("item-%s", id)`).

```go
_ = serverless.RegisterWorkflow(handler, "approve", func(ctx *serverless.Context, in Order) (string, error) {
    total, err := serverless.Step(ctx, "price", func(context.Context) (int, error) { return price(in) })
    if err != nil { return "", err }
    if err := ctx.YieldIfNeeded(); err != nil { return "", err }
    answer, err := ctx.WaitForUser(serverless.UserInput{Question: fmt.Sprintf("Ship for %d?", total), Type: "approval",
        Options: []serverless.UserInputOption{{ID: "approve", Label: "Approve"}, {ID: "reject", Label: "Reject"}}})
    if err != nil || answer != "approve" { return "rejected", err }
    payment, err := serverless.WaitForSignal[Payment](ctx, "payment.settled", "await-payment")
    if err != nil { return "", err }
    _, err = serverless.Step(ctx, "ship", func(c context.Context) (bool, error) { return ship(c, in.ID, payment.Ref) })
    ctx.Emit(serverless.Event{EventType: "order.shipped", Data: map[string]any{"order_id": in.ID}})
    return "shipped", err
})
```

The worker package's `agnt5.Step` has the same shape, so step bodies can be shared; the
handler signatures differ (`*agnt5.Context` vs `*serverless.Context`).

## Run, validate, deploy

```bash
export AGNT5_SERVERLESS_SIGNING_SECRET="$(openssl rand -base64 32)"
PORT=8787 go run ./cmd/agnt5-serverless          # scaffold listens on PORT, default 8080
agnt5 serverless validate http://127.0.0.1:8787   # second terminal
agnt5 serverless sync https://<go-host> --provider http --immutable-ref <git-sha> \
  --signing-secret-env AGNT5_SERVERLESS_SIGNING_SECRET --activate=false
```

Cloud Run: `agnt5 serverless init --provider cloud-run --runtime go`, deploy from source, sync
`--provider cloud-run --immutable-ref <revision-name>` (the scaffold reads `K_REVISION`).

## Offline test (`httptest`)

```go
h := serverless.New(serverless.Options{})            // no SigningSecret -> unsigned invokes accepted
_ = serverless.RegisterWorkflow(h, "hello", hello)
rec := httptest.NewRecorder()
h.ServeHTTP(rec, httptest.NewRequest(http.MethodPost, serverless.InvokePath,
    strings.NewReader(`{"component_type":"workflow","component_name":"hello","run_id":"r1","input":{"name":"Ada"}}`)))
// rec.Body: {"status":"completed","output":{...},"checkpoint":{...},"events":[...]}
```

To exercise a resume, send the suspension's data back in `metadata`: a signal as
`{"signal_name": "...", "waiting_step": "...", "signal_payload": "<json string>"}`, a user
answer as `{"pause_index": "0", "user_response": "ok"}`, and the previous `checkpoint` object
as `"checkpoint"` (this is what `serverless/parity_test.go` does).
