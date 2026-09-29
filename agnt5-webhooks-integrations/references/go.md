# Go webhooks, triggers, chat bots and the client

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3**; the trigger payload shape comes from a
live webhook run on 29 Sep 2026 (the product docs' Go example reads `event["body"]` at the top
level, which is wrong — the envelope is nested).

## Concept map

| Python (SKILL.md) | Go |
|---|---|
| `@workflow(triggers=[webhook("sentry", event="issue.created")])` | `agnt5.RegisterWorkflow(worker, name, handler, agnt5.WithTriggers(agnt5.WebhookTrigger("sentry", "issue.created")))` |
| `event("user.signed_up")` | `agnt5.EventTrigger("user.signed_up")` |
| `trigger_id=` | `TriggerID` field on the returned `agnt5.TriggerSpec` |
| `filter_expression=`, `input_mapping=`, `batch_window_ms=`, `delay_expression=` | `FilterExpression`, `InputMapping`, `BatchWindowMS`, `DelayExpression` exist on `TriggerSpec` but **leave them zero**: the gateway skips any trigger that sets one, so the workflow never starts |
| `ChatBot(agent, adapters=[SlackConfig(...)])` | none — `agnt5.NewChatBot` gives session memory only; no Slack/Discord/Teams/Telegram adapters |
| `Client().run(...)` / `.submit(...)` | `client.Run(...)` / `client.Submit(...)` (full client: `agnt5-client`) |

## Declare a trigger

```go
issues := agnt5.WebhookTrigger("sentry", "issue.created") // dispatches as "sentry.issue.created"
// Do not set issues.FilterExpression / InputMapping / BatchWindowMS / DelayExpression:
// the gateway skips a trigger that has any of them. Filter inside the handler.

must(agnt5.RegisterWorkflow(worker, "triage_issue", TriageIssue, agnt5.WithTriggers(issues)))
must(agnt5.RegisterWorkflow(worker, "welcome", Welcome, agnt5.WithTriggers(agnt5.EventTrigger("user.signed_up"))))
```

Sources: `standard`, `sentry`, `stripe`, `github`, `slack`; event names per source as in the
SKILL.md. Integration setup, signing secrets and signature verification are platform-side and
identical.

## What the workflow receives

The trigger run's input is an outer event record; the webhook envelope from the SKILL.md sits at
`event.data`, and `body` is still the raw request body string:

```json
{
  "event": {
    "id": "...", "name": "sentry.issue.created", "source": "...", "timestamp_ns": 0,
    "data": { "_webhook": true, "source": "sentry", "integration_id": "int_abc123",
              "event_type": "sentry.issue.created", "idempotency_key": "req_9f3c…",
              "timestamp": 1733337600, "headers": {"sentry-hook-resource": "issue"},
              "body": "{\"action\":\"created\",...}" }
  },
  "deployment_id": "...", "target_kind": "...", "target_ref": "..."
}
```

Typed handler (all fields optional so the inferred input schema stays permissive):

```go
type WebhookEnvelope struct {
    Webhook        bool              `json:"_webhook,omitempty"`
    Source         string            `json:"source,omitempty"`
    IntegrationID  string            `json:"integration_id,omitempty"`
    EventType      string            `json:"event_type,omitempty"`
    IdempotencyKey string            `json:"idempotency_key,omitempty"`
    Timestamp      any               `json:"timestamp,omitempty"` // numeric in the sample; type not verified
    Headers        map[string]string `json:"headers,omitempty"`   // keys lowercased
    Body           string            `json:"body,omitempty"`      // raw, signature-verified bytes
}

type TriggerInput struct {
    Event struct {
        ID          string          `json:"id,omitempty"`
        Name        string          `json:"name,omitempty"`
        Source      string          `json:"source,omitempty"`
        TimestampNS int64           `json:"timestamp_ns,omitempty"`
        Data        WebhookEnvelope `json:"data,omitempty"`
    } `json:"event,omitempty"`
    DeploymentID string `json:"deployment_id,omitempty"`
    TargetKind   string `json:"target_kind,omitempty"`
    TargetRef    string `json:"target_ref,omitempty"`
}

func TriageIssue(ctx *agnt5.Context, in TriggerInput) (TriageOutput, error) {
    var payload struct {
        Data struct{ Issue map[string]any `json:"issue"` } `json:"data"`
    }
    if err := json.Unmarshal([]byte(in.Event.Data.Body), &payload); err != nil {
        return TriageOutput{}, err
    }
    key := in.Event.Data.IdempotencyKey // key side effects off this + EventType
    ...
}
```

Untyped: `func(ctx *agnt5.Context, event map[string]any)` then
`event["event"].(map[string]any)["data"].(map[string]any)["body"].(string)` — check each
assertion.
Internal `EventTrigger` runs use the same outer record with the event's own payload at
`event.data` (only the webhook case was live-tested).

## Idempotent delivery

Delivery is at-least-once exactly as in the SKILL.md. Put every side effect in `agnt5.Task`/
`agnt5.Step` and key external calls off `EventType` + `IdempotencyKey` (or an ID in the body).
Slack retries are never deduped, so a Slack-triggered workflow must dedupe itself (e.g. a
`ctx.State().Scope(agnt5.StateScopeGlobal, "slack-events")` check inside a step).

## Chat bots

No Slack/Discord/Teams/Telegram adapters and no `@bot.on_*` handlers in Go. What exists:

```go
bot, err := agnt5.NewChatBot("support-bot", agent) // wraps an Agent with session conversation memory
must(agnt5.RegisterChatBot(worker, bot))            // registers as an agent component named "support-bot"
```

Invoke it with `{"message": "...", "session_id": "...", "user_id": "..."}` (`content` is accepted
as an alias for `message`); it appends the turn to `ctx.Memory().Conversation()` and replies
with `{"session_id", "message": {"role": "assistant", "content": ...}}`. From code:
`client.Chat(ctx, "support-bot", agnt5.ChatMessage{SessionID: sid, Role: agnt5.MessageRoleUser, Content: text})`.

To build a Slack bot: `agnt5.WebhookTrigger("slack", "app_mention")` on a workflow that parses
`Body`, runs the agent (or the `ChatBot.Handle(ctx, msg)` method) inside a `Step`, and posts the
reply with Slack's Web API from another `Step`. Signature verification is still done by AGNT5.

## Calling a deployed workflow from your app

```go
client, err := agnt5.NewClient("", agnt5.WithAPIKey(os.Getenv("AGNT5_API_KEY"))) // env: AGNT5_GATEWAY_URL, AGNT5_API_KEY

res, err := client.Run(ctx, "onboarding_workflow", map[string]any{"user_email": "ada@example.com"},
    agnt5.WithRunComponentType(agnt5.ComponentTypeWorkflow), // default is function
    agnt5.WithIdempotencyKey("onboard:"+userID),
    agnt5.WithWaitTimeout(2*time.Minute)) // gateway wait, default 5 min; 0 = return the accepted receipt
if err != nil { ... }                     // transport error or *agnt5.RunError (errors.As)
if res.IsPending() { res, err = client.WaitForResult(ctx, res.RunID, 10*time.Minute, 2*time.Second) }
var out OnboardingOutput
err = res.DecodeOutput(&out)              // res.Status, res.RunID, res.TraceID, res.Error

sub, err := client.Submit(ctx, "onboarding_workflow", input, // fire and forget
    agnt5.WithSubmitComponentType(agnt5.ComponentTypeWorkflow),
    agnt5.WithSubmitIdempotencyKey("onboard:"+userID))
// sub.RunID; later client.GetStatus / GetResult / GetEvents / CancelRun(ctx, runID, reason)

res, err = client.Workflow("onboarding_workflow").Run(ctx, input)          // fluent form
res, err = client.Session("sess-1").WithUser("ada").Run(ctx, "support_bot", input) // X-Session-ID / X-User-ID
```

`WithRunSessionID`/`WithRunUserID`, `WithRunTimeout` (HTTP deadline), `WithRunHeader`, `Batch`,
`Stream`/`StreamEvents`, `ResumeWorkflow` and `RunError` details: `agnt5-client`.

## Not available in Go

Chat adapters (`SlackConfig`, `DiscordConfig`, `TeamsConfig`, `TelegramConfig`), `@bot.on_*`
routing, `AsyncClient`, a typed webhook-envelope type in the SDK (define your own as above).

## Go pitfalls for this skill

- Reading `event["body"]` at the top level (docs example) gets `nil`; the envelope is at
  `event.data`.
- A typed input with required fields may reject a trigger payload; use `omitempty` everywhere or
  `map[string]any`.
- `client.Run` without `WithRunComponentType` hits `/v1/functions/<name>/run` — a workflow
  name there fails.
- Long-running workflows exceed the 5-minute default wait: pass `WithWaitTimeout(0)` and poll,
  or `Submit`.
