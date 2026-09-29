# Go workflows and functions

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3** (SDK source, not the product docs —
the docs understate Go in places). Import path: `github.com/agnt5dev/sdk-go/agnt5`.

## Concept map

| Python (SKILL.md) | Go |
|---|---|
| `@workflow` / `@function` | `agnt5.RegisterWorkflow(worker, name, handler, opts...)` / `agnt5.RegisterFunction(...)` — explicit, each returns `error` |
| `ctx: WorkflowContext` / `FunctionContext` | `ctx *agnt5.Context` (one type for every component; it is also a `context.Context`) |
| `await ctx.step(fn, *args, key=...)` | `agnt5.Task(ctx, name, input, fn)` / `agnt5.TaskWithKey(ctx, name, key, input, fn)`; for closures `agnt5.Step` / `agnt5.StepWithKey` |
| `ctx.run(fn, ...)`, `ctx.task()` | none |
| `ctx.parallel / gather / batch / map` | none — goroutines + `sync.WaitGroup` + keyed steps (recipe below) |
| `await ctx.sleep(seconds, name=...)` | `ctx.Sleep(d, agnt5.WithSleepKey(name)) error` — **return its error** |
| `ctx.state`, `ctx.session.state`, `ctx.user.state` | `ctx.State()`, `ctx.State().Scope(agnt5.StateScopeSession, ns)`, `Scope(agnt5.StateScopeUser, userID)` |
| `ctx.activation.idempotency_key` | `agnt5.ActivationFromContext(c)` / `ctx.Activation()` → `.IdempotencyKey` |
| `retries=3, backoff="exponential"` | `agnt5.WithRetry(3, 500, 10000)`, `agnt5.WithBackoff("exponential", 2.0)` |
| `timeout_ms=` | none — `context.WithTimeout` inside the handler |
| `cron=`, `triggers=[...]` | `agnt5.WithCron("0 9 * * *")`, `agnt5.WithTriggers(agnt5.EventTrigger(...), agnt5.WebhookTrigger(...))` |
| `chat=True` | none — `agnt5.NewChatBot` + `RegisterChatBot`, see `agnt5-webhooks-integrations` |
| `ctx.attempt`, `ctx.run_id`, `ctx.logger` | `ctx.Attempt()`, `ctx.RunID()`, `ctx.Logger().Info(msg, "k", v)` |

## Handlers and registration

```go
type OnboardingInput struct {
    UserEmail string `json:"user_email" description:"Address of the new user"`
    Plan      string `json:"plan,omitempty"` // omitempty => optional in the published schema
}

type OnboardingOutput struct {
    Status    string `json:"status"`
    AccountID string `json:"account_id"`
}

func OnboardingWorkflow(ctx *agnt5.Context, in OnboardingInput) (OnboardingOutput, error) {
    account, err := agnt5.Task(ctx, "create_account", CreateAccountInput{Email: in.UserEmail}, CreateAccount)
    if err != nil {
        return OnboardingOutput{}, err
    }
    if _, err := agnt5.Task(ctx, "send_welcome_email", SendEmailInput{To: in.UserEmail}, SendEmail); err != nil {
        return OnboardingOutput{}, err
    }
    return OnboardingOutput{Status: "done", AccountID: account.ID}, nil
}

// main.go — registration is explicit; there is no auto_register.
must(agnt5.RegisterWorkflow(worker, "onboarding_workflow", OnboardingWorkflow))
must(agnt5.RegisterFunction(worker, "send_email", SendEmail,
    agnt5.WithRetry(3, 500, 10000),          // maxAttempts, initial ms, max ms
    agnt5.WithBackoff("exponential", 2.0),   // "constant" | "linear" | "exponential", multiplier
))
```

Handler shape is always `func(ctx *agnt5.Context, in In) (Out, error)`; `In`/`Out` are decoded
and encoded with `encoding/json`. The JSON Schema is inferred by reflection: exported fields,
`json` tag names, `omitempty`/`omitzero` → not required, `description:"..."` and `format:"..."`
struct tags are copied into the schema, `time.Time` → `date-time`, `map` → `object`, `any` → `{}`.
Override with `agnt5.WithInputSchema(map[string]any{...})` / `WithOutputSchema`. Other options:
`WithComponentMetadata`, `WithComponentConfig`, `WithCron`, `WithTriggers`. Returning an error
fails the run (retries per `WithRetry` when the runtime invokes the component directly).
`agnt5.RegisterRaw(worker, name, agnt5.ComponentTypeFunction, func(*agnt5.Context, []byte) ([]byte, error))`
is the untyped escape hatch.

## Steps: the unit of durable work

```go
// Task: checkpoint + a Function node in Studio. fn has the RegisterFunction handler shape,
// so the same Go function can be registered standalone and used as a step.
ids, err := agnt5.Task(ctx, "fetch_top_ids", FetchTopIDsInput{Limit: 5}, FetchTopIDsFunction)

// Step: checkpoint around a closure. T must survive a JSON round trip.
stories, err := agnt5.Step(ctx, "fetch_stories", func(c context.Context) ([]Story, error) {
    return fetchAllStories(c, ids)
})

// Keyed variants for loops, branches and goroutines: key = stable per-item identity.
summary, err := agnt5.TaskWithKey(ctx, "summarize", strconv.Itoa(story.ID),
    SummarizeInput{Story: story}, SummarizeFunction)
score, err := agnt5.StepWithKey(ctx, "score", item.ID, func(sc *agnt5.Context) (Score, error) {
    return scoreItem(sc, item)
})
```

| Call | Checkpointed | Notes |
|---|---|---|
| `agnt5.Task` / `TaskWithKey` | yes | emits `function.started/completed`, renders as its own Function node |
| `agnt5.Step` / `StepWithKey` | yes | anonymous step |
| `fn(ctx, in)` called directly | no | re-runs on every replay — the same mistake the SKILL.md warns about |

- Without a key, the checkpoint key is the step name plus a call-order ordinal. Any loop,
  branch, goroutine or reorderable call needs `*WithKey` with a key that is unique per item and
  stable across code changes (an ID, not a slice index). Reusing a key with different durable
  semantics returns `agnt5.ErrNondeterministicReplay`.
- Step results are serialized to the checkpoint: use exported fields with `json` tags.
  Unexported fields encode as `{}` and come back zero on replay.
- `Step` hands the closure a `context.Context`; under the durable runtime that value is a
  step-scoped `*agnt5.Context` (assert `c.(*agnt5.Context)` for `Logger`/`State`).
  `StepWithKey` and `Task` hand you `*agnt5.Context` directly. The shipped templates keep
  using the outer `ctx` inside `Step` closures (for agent runs); that works too.
- Signatures: `Step[T](ctx, name string, fn func(context.Context) (T, error))`,
  `StepWithKey[T](ctx, name, key string, fn func(*agnt5.Context) (T, error))`,
  `Task[In, Out](ctx, name string, input In, fn func(*agnt5.Context, In) (Out, error))`,
  `TaskWithKey[In, Out](ctx, name, key string, input In, fn ...)`.

## Running in parallel

There is no `ctx.parallel/gather/batch/map`. Two recipes:

**Per-item durable (preferred for 10+ items or expensive items).** Keyed steps are the SDK's
fan-out primitive (`StepWithKey`/`TaskWithKey` doc comments, CHANGELOG 0.6.0); run them from
goroutines with your own semaphore:

```go
func summarizeAll(ctx *agnt5.Context, stories []Story) ([]SummarizedStory, error) {
    out := make([]SummarizedStory, len(stories))
    errs := make([]error, len(stories))
    sem := make(chan struct{}, 5) // max_concurrency
    var wg sync.WaitGroup
    for i, story := range stories {
        wg.Add(1)
        go func(i int, story Story) {
            defer wg.Done()
            sem <- struct{}{}
            defer func() { <-sem }()
            out[i], errs[i] = agnt5.TaskWithKey(ctx, "summarize", strconv.Itoa(story.ID),
                SummarizeInput{Story: story}, SummarizeFunction)
        }(i, story)
    }
    wg.Wait()
    return out, errors.Join(errs...) // continue_on_failure: filter errs instead of joining
}
```

**Coarse (what the shipped Go templates do).** One `agnt5.Step` around a `sync.WaitGroup`
fan-out. Simpler, but a restart re-runs the whole group, model calls included. Fine for 2–5
cheap branches (the `ctx.parallel` case); avoid for per-item LLM work.

## Durable sleep

```go
if err := ctx.Sleep(24*time.Hour, agnt5.WithSleepKey("wait_24h")); err != nil {
    return Output{}, err // under the durable runtime this is a suspension; the run resumes here later
}
```

Like `AskUser`, `Sleep` returns an error the handler must propagate; on replay a completed timer
returns `nil` immediately. Without a durable runtime (plain local process) it blocks in-process
and honours `ctx.Done()`. `time.Sleep` is never checkpointed.

## State

```go
if err := ctx.State().Set(ctx, "phase", "started"); err != nil { return Output{}, err }
phase, err := ctx.State().GetString(ctx, "phase")          // Get(ctx, key) (any, error) also exists
if errors.Is(err, agnt5.ErrStateNotFound) { /* unset */ }

visits := ctx.State().Scope(agnt5.StateScopeSession, "visits") // Session | User | Global | Run
raw, err := visits.Get(ctx, "count")                            // numbers come back as float64
```

`StateManager` methods: `Get`, `GetString`, `Set`, `Delete`, `List`, `Scope`. Values round-trip
through JSON, so decode numbers as `float64` (or store strings). Memory (`ctx.Memory()` KV /
working / conversation) is covered in `agnt5-agents-tools`.

## Idempotent side effects

```go
func chargeCustomer(c context.Context, orderID string, total int) (Charge, error) {
    key := "charge:" + orderID
    if act, ok := agnt5.ActivationFromContext(c); ok && act.IdempotencyKey != "" {
        key = act.IdempotencyKey // ActivationExecution{ActivationID, Attempt, IdempotencyKey}
    }
    return stripeCharge(c, orderID, total, key)
}
```

`ActivationFromContext` works on the context handed into a step or tool; `ctx.Activation()` is
the method form on `*agnt5.Context`.

## Triggers and streaming

`agnt5.RegisterWorkflow(worker, "daily_report", DailyReport, agnt5.WithCron("0 9 * * *"))` —
give the input struct only optional fields; the scheduled payload is not verified to carry any.
Event/webhook triggers, payload shape and `TriggerSpec` options: `agnt5-webhooks-integrations`.
Pauses: `agnt5-human-in-the-loop`. Streaming: `ctx.Output(delta)` emits `output.delta`
(`ctx.IsStreaming()` tells you whether anyone listens); built-in model providers do not stream,
so agent text is not streamed token by token.

## Not available in Go

`ctx.parallel/gather/batch/map`, `ctx.run`, `timeout_ms`, `chat=True`, sync-function
thread-pool wrapping, `ctx._is_replay`, `ctx.user.state` helper (use `Scope`), per-step retry
options, `Worker(auto_register=True)`.

## Go pitfalls for this skill

- `WithRetry` is registration metadata used when the runtime invokes the component. In Python and
  TypeScript retries are not applied to a function called inside a workflow step;
  the Go step-level behaviour was not tested — retry inside the step body if it matters.
- No timeout option: wrap external calls with `context.WithTimeout(ctx, d)`; the run's own
  deadline still applies through `ctx.Done()`.
- Unkeyed steps inside goroutines or `if` branches match the wrong checkpoint on replay. Use
  `*WithKey`.
- Step and state values are JSON: unexported fields vanish, ints become `float64` on the way
  back from `State().Get`.
- Forgetting to `return` the error from `ctx.Sleep`/`ctx.AskUser` turns a durable pause into a
  no-op.
- Returning `error` from a `Step` closure fails the step (and, if unhandled, the run); there is
  no per-item error collection — build it yourself as in the fan-out recipe.
