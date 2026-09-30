# Go observability notes

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3**. The CLI/MCP/Studio parts of the
SKILL.md (`agnt5 inspect runs|trace|logs`, metrics) are the same for Go runs; this file covers
what a Go worker emits and how to control it.

## What lands in the trace

| Go call | Trace / journal |
|---|---|
| `agnt5.Step` / `StepWithKey` | `workflow.step.<name>` span (a durable STEP activation on the current runtime) |
| `agnt5.Task` / `TaskWithKey` | same, plus `function.started/completed` → its own Function node in Studio |
| `agent.Run` | `agent.*` iteration events, one lm span per model call (nested under the iteration since 0.10.3), `tool_call.*` per tool |
| `ctx.Generate` | one lm span / MODEL activation (`lm.completed` / `lm.failed` are the event names trace scorers match) |
| `ctx.AskUser` / `ctx.Sleep` | `workflow.step.paused`, `approval.requested`, `workflow.paused` / timer activation |
| `load_skill` | `skill.loaded` (`skill_name`, `instructions_length`, `resources_materialized`) |
| `ctx.Logger().Info(...)` | run-scoped log record (MCP `get_run_logs`, the run in Studio) plus a `log.info` journal event |
| `ctx.Output(delta)` | `output.delta` (streaming) |
| `ctx.Emit(agnt5.Event{Type: "order.validated", Data: map[string]any{...}})` | your own event in the journal |

Read a run's journal from code: `client.GetEvents(ctx, runID)` returns
`(*agnt5.EventsResponse, error)`; the events are in `resp.Items` (`[]agnt5.RunEvent{EventType,
Data, StepKey, CorrelationID, ...}`) and `resp.Count` is their number:

```go
resp, err := client.GetEvents(ctx, runID)
if err != nil {
    return err
}
for _, ev := range resp.Items {
    fmt.Println(ev.EventType, string(ev.Data))
}
```

`agnt5.ExtractToolCallsFromEvents` works on `[]agnt5.TraceEvent` (the scorer-side shape,
`agnt5-scorers`).

## Logging

```go
ctx.Logger().Info("Fetched top IDs", "count", len(ids), "limit", in.Limit) // keyvals, run-scoped
ctx.Logger().Warn(...) / Error(...) / Debug(...)
```

Application-wide `slog` that stays correlated with runs:

```go
slog.SetDefault(slog.New(agnt5.NewSlogHandler(slog.NewJSONHandler(os.Stdout, nil)))) // before worker.Run
// in handlers/tools: pass the invocation context
slog.InfoContext(ctx, "cache miss", "key", key) // forwarded to AGNT5 with run correlation
slog.Info("starting")                          // no context => local only
```

`NewSlogHandler` keeps your handler's level filtering and groups; records without an invocation
context (or a context derived from one) are not forwarded.

Where each kind of output can be read:

| Output | Local `agnt5 dev` | Deployed worker |
|---|---|---|
| `ctx.Logger()` | run's logs only (`agnt5 inspect logs -r`, MCP `get_run_logs`, Studio); not printed in the terminal | run's logs |
| `slog.InfoContext(ctx, ...)` via `NewSlogHandler` | terminal and run's logs | run's logs |
| `log.Printf`, `fmt.Println`, `slog.Info` without context | terminal (`agnt5 dev logs` when detached) | not shown anywhere |

`agnt5 inspect logs -r <runId>` shows the run's logs (CLIs older than `20260930-a31e8d` answer
403); `agnt5 logs <deployment-id>` is the platform's lifecycle log for the deployment, not your
worker's output. A deployed worker that crashes shows its last output line in
`agnt5 deploy debug <deployment-id> --logs`. In the run's logs each keyval becomes a string
attribute named `field.<key>` (`"n", 3` → `field.n: "3"`).

## Export (OTLP) and metrics

- Logs and lifecycle records export over OTLP gRPC. With no endpoint configured the SDK uses
  its built-in default; set `OTEL_EXPORTER_OTLP_ENDPOINT` (or `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`)
  to send them to your collector. Every record carries the canonical
  workspace/project/deployment and `agnt5.app_name` attributes.
- Trace spans (invocation, step, model) are exported **only** when
  `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` or `OTEL_EXPORTER_OTLP_ENDPOINT` is set (0.9.0+). W3C
  parents and runtime trace IDs are preserved; handler errors and panics are recorded.
- `AGNT5_CORE_METRICS_LOGS=1` prints `AGNT5_CORE_METRIC` JSON lines (pull-slot occupancy,
  activation RPC timings) to stderr for local benchmarking.

## Model-call retries and timeouts (0.10.2+)

OpenAI, Anthropic and Google calls get a 10-minute request timeout (bounded by the run's
deadline) and retry up to twice with jittered backoff on timeouts and 408/429/500/502/503/504/529,
honouring `Retry-After`. Overrides: `AGNT5_LM_MAX_RETRIES`, `AGNT5_LM_INITIAL_DELAY_MS`,
`AGNT5_LM_MAX_DELAY_MS`. A call that still fails returns `*agnt5.ModelRequestError{Provider,
Attempts, Timeout, Err}`:

```go
var mre *agnt5.ModelRequestError
if errors.As(err, &mre) && mre.Timeout { ctx.Logger().Warn("provider timed out", "provider", mre.Provider, "attempts", mre.Attempts) }
```

## Automatic capture: Python/TypeScript only

`AGNT5_CAPTURE*` and the `agnt5[openai]`/`[openai-agents]`/`[google-adk]` extras have no Go
counterpart. In Go a model call is traced only when it goes through an `agnt5.LanguageModel`
(`agent.Run`, `ctx.Generate`). Calls made with a vendor Go SDK directly are invisible — wrap them
in a type implementing `Generate(ctx, agnt5.GenerateRequest) (agnt5.GenerateResponse, error)`
and call it through `ctx.Generate`, or at least run them inside a `Step` so the span and its
duration exist. Token usage for direct calls: `resp.Usage` (`InputTokens`, `OutputTokens`,
`CachedTokens`, `CacheCreationTokens`).

## Streaming a run's output

`ctx.Output(delta)` in the handler; `ctx.IsStreaming()` reports whether a listener is
attached. Consumers: `client.Stream(ctx, component, input, func(chunk string) error {...})`
or `client.StreamEvents(...)` for typed `agnt5.ReceivedEvent`s (`stream.wait_expired`,
`stream.detached` when the gateway wait ends; the run continues). Built-in providers do not
implement `agnt5.StreamingLanguageModel`, so agent tokens are not streamed — emit
`ctx.Output` yourself after `agent.Run` returns, or plug in a custom streaming model.

## Not available in Go

`AGNT5_CAPTURE*` auto-capture, `capture_mode=observed` spans, structured `ctx.logger`
keyword form (`ctx.logger.info("msg", to=to)` → `ctx.Logger().Info("msg", "to", to)`).

## Go pitfalls for this skill

- Using `log.Printf` for run diagnostics: it never reaches the run's logs and is invisible
  once deployed — use `ctx.Logger()` or `slog.InfoContext(ctx, ...)` through `NewSlogHandler`.
- Direct vendor SDK calls leave gaps in the trace and in LLM cost metrics.
- Trace export silently stays off until an OTLP traces endpoint is set; only logs are exported by
  default.
- Reading `ctx.Events()` mid-run returns the events buffered so far, not the runtime's journal;
  use `client.GetEvents` for the full record.
