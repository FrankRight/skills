# Go human-in-the-loop

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3**; answer shapes from a live resume
test on 29 Sep 2026.

## Concept map

| Python (SKILL.md) | Go |
|---|---|
| `await ctx.wait_for_user(question, input_type, options, allow_custom, skippable)` | `ctx.AskUser(agnt5.UserInputRequest{ID, Prompt, Type, Options, AllowCustom, Skippable, Metadata}) (string, error)` |
| `input_type="text" \| "approval" \| "select" \| "multiselect"` | `agnt5.HITLText` (default) \| `HITLApproval` \| `HITLSelect` \| `HITLMultiSelect` |
| `options=[{"id": "pdf", "label": "PDF"}]` | `[]agnt5.HITLOption{{Label: "PDF", Value: "pdf"}}` — `Value` is what comes back |
| approval shortcut | `ctx.RequestApproval(prompt, metadata) (bool, error)` |
| `None` when skipped | `""` (see answer table) |
| `AskUserTool(ctx)` / `RequestApprovalTool(ctx)` | none — a custom tool that calls `AskUser` (below) |
| resume API | `client.ResumeWorkflow(ctx, runID, answer)` |

## The one rule: return the error

The first `AskUser` call emits the pause events and returns
`("", *agnt5.WaitingForUserInputError)`. The workflow handler **must return that error**
(wrapping with `%w` is fine; test with `agnt5.IsWaitingForUserInput(err)`). If you swallow it,
the run continues with `""` as the answer and never pauses. When the user answers, the whole
handler re-runs from the top and the same call returns the saved answer with a `nil` error.

```go
func ReviewWorkflow(ctx *agnt5.Context, in ReviewInput) (ReviewOutput, error) {
    draft, err := agnt5.Task(ctx, "generate_draft", DraftInput{Topic: in.Topic}, GenerateDraft) // checkpointed
    if err != nil {
        return ReviewOutput{}, err
    }

    decision, err := ctx.AskUser(agnt5.UserInputRequest{
        Prompt: "Approve this draft?\n\n" + draft,
        Type:   agnt5.HITLApproval,
        Options: []agnt5.HITLOption{
            {Label: "Approve", Value: "approve"},
            {Label: "Discard", Value: "discard"},
        },
    })
    if err != nil {
        return ReviewOutput{}, err // first call: pauses the run; never guard this line
    }
    if decision == "discard" {
        return ReviewOutput{Status: "cancelled"}, nil
    }

    name, err := ctx.AskUser(agnt5.UserInputRequest{Prompt: "Title for the report?"}) // Type defaults to text
    if err != nil {
        return ReviewOutput{}, err
    }

    format, err := ctx.AskUser(agnt5.UserInputRequest{
        Prompt: "Which output format?", Type: agnt5.HITLSelect,
        Options: []agnt5.HITLOption{{Label: "PDF", Value: "pdf"}, {Label: "Markdown", Value: "markdown"}},
        AllowCustom: true, Skippable: true,
    })
    if err != nil {
        return ReviewOutput{}, err
    }
    if format == "" || format == "null" {
        format = "markdown" // skipped
    }

    topics, err := ctx.AskUser(agnt5.UserInputRequest{
        Prompt: "Which topics?", Type: agnt5.HITLMultiSelect,
        Options: []agnt5.HITLOption{{Label: "Market", Value: "market"}, {Label: "Tech", Value: "tech"}},
    })
    if err != nil {
        return ReviewOutput{}, err
    }
    return publish(ctx, name, format, selectedValues(topics))
}
```

## What the returned string contains

| Type | Returned string |
|---|---|
| text | what the user typed |
| approval / select | the chosen option's `Value` (`"approve"`, `"pdf"`) |
| multiselect | a JSON array **string**, e.g. `["market","tech"]` — `json.Unmarshal` it; fall back to a comma split for older UIs |
| skipped (`Skippable: true`) | `""` (the SDK maps the runtime's `__skipped__` marker); a JSON `null` sent through the resume API arrived as the literal string `"null"` — treat both as skipped |
| `AllowCustom` free text | the text (the SDK strips a `__custom__:` prefix) |

`RequestApproval` returns `true` only for `approve`, `approved` or `yes` (case-insensitive).

```go
func selectedValues(answer string) []string {
    var values []string
    if err := json.Unmarshal([]byte(answer), &values); err == nil {
        return values
    }
    if answer == "" || answer == "null" {
        return nil
    }
    return strings.Split(answer, ",")
}
```

## Replay safety

Same rule as Python: **checkpoint side effects, never guard the pause.** Everything above the
pause runs again on resume; `agnt5.Task`/`agnt5.Step` results are carried across the pause in
the checkpoint metadata and return immediately, bare statements re-run. There is no
`_is_replay` flag in Go, so a bare `sendEmail(...)` before a pause sends twice — wrap it.
`ctx.Logger()` lines before a pause duplicate on resume; that is noise, not a bug.

## Multiple and conditional pauses

Answers are matched to calls by pause **index** (call order), not by `ID`; calls inside `if`
blocks are fine because only executed branches consume an index. `ID` is optional — every
ID-less call defaults to `<runID>:user-input`, and two such calls in one workflow resolved
correctly in the live test. Set `ID` when you want a recognisable name in the events.

State across pauses: derive later values from step results (they replay) rather than relying on
`ctx.State()` — whether run state survives a pause/resume was not verified.

## Agent-level HITL

There is no `AskUserTool`. A tool whose handler calls `AskUser` does the same job: when a tool
returns a waiting error the agent loop returns it unchanged (`agent.go`), so the workflow
pauses; on resume the agent loop starts again from its first turn and the tool call finds the
saved answer. Keep such agents short (`WithAgentMaxTurns`) and do not wrap the model turn in a
`Step` that hides the error.

```go
askUser, err := agnt5.NewTool("ask_user", func(c context.Context, args map[string]any) (any, error) {
    ctx, ok := c.(*agnt5.Context)
    if !ok {
        return nil, errors.New("ask_user needs an AGNT5 workflow context")
    }
    question, _ := args["question"].(string)
    return ctx.AskUser(agnt5.UserInputRequest{Prompt: question, Type: agnt5.HITLText}) // returns the waiting error on first call
},
    agnt5.WithToolDescription("Ask the human a clarifying question and wait for the answer."),
    agnt5.WithToolSchema(map[string]any{
        "type":       "object",
        "properties": map[string]any{"question": map[string]any{"type": "string"}},
        "required":   []string{"question"},
    }),
)
```

`request_approval` is the same tool with `ctx.RequestApproval(prompt, nil)`. Build the tools
once at startup (they read the context from the call, not from a captured variable) and run the
agent from a workflow handler, not from `RegisterAgent` — pauses need a workflow run.

## Resuming from code

The run reports `agnt5.RunStatusPaused` both while it waits for an answer and during a durable
`ctx.Sleep`; `RunStatusAwaitingUserInput` never appears. Only the newest `workflow.paused` event
tells them apart: a question has `pause_reason: "user_input_required"` in its `Metadata`, a
sleep has `reason: "timer"` in its `Data`. A resume sent during a sleep is accepted and its
answer goes to the next question without that question being shown, so check first.
`ResumeWorkflow` and `CancelRun` need a key with the `workflow` scope (`agnt5 service-keys
create ... --scopes run,workflow`); with a `run`-only key they return a `*agnt5.ClientError`
with `StatusCode` 403 (`INSUFFICIENT_SCOPES`). Deployed workers pause on `ctx.Sleep`; under
local `agnt5 dev` a Go `ctx.Sleep` does not suspend (the run shows `assigned`), so test the
paused-sleep case on a deployment.

```go
// pendingQuestion returns the metadata of the question a paused run waits on,
// or nil while the run sleeps or is still running.
func pendingQuestion(ctx context.Context, client *agnt5.Client, runID string) (map[string]any, error) {
    status, err := client.GetStatus(ctx, runID)
    if err != nil || status.Status != agnt5.RunStatusPaused {
        return nil, err
    }
    events, err := client.GetEvents(ctx, runID)
    if err != nil {
        return nil, err
    }
    var latest map[string]any
    for _, ev := range events.Items {
        if ev.EventType == "workflow.paused" {
            latest = ev.Metadata
        }
    }
    if latest["pause_reason"] != "user_input_required" {
        return nil, nil
    }
    return latest, nil
}
```

```go
client, err := agnt5.NewClient("", agnt5.WithAPIKey(os.Getenv("AGNT5_API_KEY"))) // "" => AGNT5_GATEWAY_URL
res, err := client.Run(ctx, "review_workflow", in, agnt5.WithRunComponentType(agnt5.ComponentTypeWorkflow))
// Run returns at the first pause: res.Status == agnt5.RunStatusPaused, res.IsPending() == true.
if q, err := pendingQuestion(ctx, client, res.RunID); err == nil && q != nil {
    _, err = client.ResumeWorkflow(ctx, res.RunID, "approve") // POST /v1/workflows/resume/{run_id} {"user_response": "approve"}
}
_, err = client.ResumeWorkflow(ctx, res.RunID, []string{"market", "tech"}) // multiselect: arrives as the JSON string above
_, err = client.CancelRun(ctx, res.RunID, "operator stop")                 // POST /v1/runs/{run_id}/cancel {"reason": ...}
```

`WaitForResult` counts `paused` as finished and returns at once with `Status ==
agnt5.RunStatusPaused`. Paused runs are missing from `agnt5 inspect runs ls` and
`agnt5 inspect runs describe` returns 404 until the run finishes, so keep `res.RunID`
(`GET /v1/runs?component_name=<workflow>` on the gateway lists unfinished runs). Studio
and `agnt5 run` handle the pause UI for you; `ResumeWorkflow` is for your own backend.

## Not available in Go

`AskUserTool`/`RequestApprovalTool`, `ctx._is_replay`, `None` return (Go returns `""`),
`options` as `id`/`label` dicts (Go uses `Label`/`Value`).

## Go pitfalls for this skill

- Not returning the `AskUser`/`RequestApproval` error: no pause, empty answer. Same for
  `ctx.Sleep`.
- Multiselect answers are a JSON string, not `"a,b"`; skips can arrive as `"null"`. Use the
  helper above.
- A side effect before the pause outside `Task`/`Step` runs on every resume.
- `RequestApproval` treats anything but approve/approved/yes as rejection — including a custom
  option you added yourself.
