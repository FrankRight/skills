---
name: agnt5-client
description: Call deployed AGNT5 components from your own backend, scripts, or CI with the SDK clients - Python Client/AsyncClient, TypeScript Client, Go agnt5.NewClient - service keys (agnt5_sk_), the gateway URL (https://gw.agnt5.com; the TypeScript default is localhost:34181), run vs submit, the 202 pending receipt after wait_timeout and wait_for_result, streaming text and typed events, sessions/users/tenants, idempotency keys, batch runs, inline eval, large outputs, cancelling, and answering a paused human-in-the-loop run or sending a signal over REST (POST /v1/workflows/resume/{run_id}, /v1/runs/{run_id}/signals/{name}). Use when integrating AGNT5 into an application, triggering a run from a script or webhook handler, polling run status, or resuming a paused workflow from outside Studio.
---

# AGNT5 Client

The client talks to the **gateway** over HTTPS. It never runs your code; it starts runs on
whatever deployment serves the target environment and reads their status, output, and
events. For webhook-driven starts see `agnt5-webhooks-integrations`; for `agnt5 run` from a
shell see `agnt5-project-init`.

## Setup

```bash
agnt5 service-keys create --name backend --project <project-id> [--environment production]   # prints agnt5_sk_... once
export AGNT5_API_KEY=agnt5_sk_...
export AGNT5_GATEWAY_URL=https://gw.agnt5.com     # required for TypeScript; the Python/Go default
```

| | Python | TypeScript | Go |
|---|---|---|---|
| Construct | `Client(gateway_url=None, timeout=45.0, api_key=None, tenant_id=None, deployment_id=None)`; `AsyncClient(...)` same args | `new Client({ gatewayUrl, apiKey, tenantId, deploymentId, timeout: 45000, maxRetries: 0, retryDelayMs })` | `agnt5.NewClient("", agnt5.WithAPIKey(key), agnt5.WithTenantID(t), agnt5.WithClientDeploymentID(d), agnt5.WithClientTimeout(45*time.Second), agnt5.WithHTTPClient(c))` |
| Gateway default | `AGNT5_GATEWAY_URL`, else `https://gw.agnt5.com` | `AGNT5_GATEWAY_URL`, else **`http://localhost:34181`** | `AGNT5_GATEWAY_URL`, else `https://gw.agnt5.com` |
| Key | `AGNT5_API_KEY`; `ValueError` unless it starts with `agnt5_sk_` | `AGNT5_API_KEY` | `AGNT5_API_KEY`; error unless `agnt5_sk_` |
| Ambient | - | `AGNT5_TENANT_ID`, `AGNT5_DEPLOYMENT_ID` | `AGNT5_TENANT_ID`, `AGNT5_DEPLOYMENT_ID` |

Without `deployment_id`/`X-DEPLOYMENT-ID` a project key targets the environment it was
created for (production by default). A deployment-pinned key rejects a conflicting
`X-DEPLOYMENT-ID`. Never put the key in code; the clients send it as `X-API-KEY`.

## Run, submit, wait

`run` blocks until the run finishes **or** the wait budget elapses (default 300 s, Python max
86400), then returns a **202 pending receipt** instead of raising. Check it every time:

```python
from agnt5 import Client, RunError

client = Client()
res = client.run("onboarding", {"email": "ada@example.com"}, component_type="workflow",
                 idempotency_key=f"onboard:{user_id}", wait_timeout=120)
if res.is_pending:                                   # status_code == 202
    res = client.wait_for_result(res.run_id, timeout=600, poll_interval=2)
res.raise_for_status()                               # RunError(message, run_id, error_code, attempts, max_attempts)
print(res.output, res.trace_id, res.duration_ms)
```

```typescript
import { Client } from '@agnt5/sdk';
const client = new Client({ gatewayUrl: process.env.AGNT5_GATEWAY_URL });
let res = await client.run('onboarding', { email }, { componentType: 'workflow', idempotencyKey: `onboard:${userId}`, waitTimeoutMs: 120_000 });
if (res.isPending) res = await client.waitForResult(res.runId, 600_000, 2_000);
res.raiseForStatus();                                // throws RunError
const output = await client.resolveOutput(res);      // dereferences output_ref for large (serverless) outputs
```

```go
client, err := agnt5.NewClient("", agnt5.WithAPIKey(os.Getenv("AGNT5_API_KEY")))
res, err := client.Run(ctx, "onboarding", map[string]any{"email": email},
    agnt5.WithRunComponentType(agnt5.ComponentTypeWorkflow), agnt5.WithIdempotencyKey("onboard:"+userID),
    agnt5.WithWaitTimeout(2*time.Minute))
if err == nil && res.IsPending() {
    res, err = client.WaitForResult(ctx, res.RunID, 10*time.Minute, 2*time.Second)
}
if err := res.RaiseForStatus(); err != nil { var runErr *agnt5.RunError; errors.As(err, &runErr) }
var out Onboarding
_ = res.DecodeOutput(&out)
```

`submit` returns immediately with `run_id` (+ `status_url`); poll with `get_status` /
`getStatus` / `GetStatus` (`is_complete`, `is_running`) and fetch with `get_result`.
`component_type` defaults to `"function"` everywhere - pass `workflow`, `agent`, or `tool`.
Agent input must contain `"message"`.

Statuses: `pending`, `enqueued`, `queued`, `started`, `running`, `completed`, `failed`,
`cancelled`, `paused`, `awaiting_input`, `awaiting_user_input`, `timeout`, `unknown`.
`is_success` = 200 + `completed`; `is_error` = 500 or failed/cancelled/timeout; a paused HITL
run is neither - see below. `RunError.was_retried` / `exhausted_retries` tell you whether the
platform already retried.

## Streaming

| | Text chunks | Typed events |
|---|---|---|
| Python | `for chunk in client.stream("summarize", {...}): ...` (str) | `for ev in client.stream_events("agent", {...}, component_type="agent"): ev.event_type, ev.data` (`ReceivedEvent`; also on `AsyncClient`) |
| TypeScript | `for await (const chunk of client.stream(...))` | `for await (const ev of client.events(...))` |
| Go | `client.Stream(ctx, name, input, func(chunk string) error {...}, opts...)` | `client.StreamEvents(ctx, name, input, func(ev agnt5.ReceivedEvent) error {...})` |

Streaming needs a streaming-capable component (`agnt5-workflows` async generators, agents).
Past events of any run: `get_events(run_id)` / `getEvents` / `GetEvents` (journal records
with `event_type`, `data`, `step_key`, `sequence`).

## Sessions, users, tenants, idempotency

- `session_id` / `sessionId` / `WithRunSessionID` scopes chat history and `ctx.session.state`;
  `user_id` scopes `ctx.user.state` (`agnt5-workflows`). Fluent forms:
  Python `client.workflow("support").chat("hi", session_id="s1")`, TypeScript
  `client.workflow('support').chat('hi', 's1')`, Go
  `client.Session("s1").WithUser("u1").Workflow("support").Run(ctx, input)` or
  `client.Chat(ctx, "support_agent", agnt5.ChatMessage{Content: "hi", SessionID: "s1"})`.
- Tenant (`tenant_id`/`tenantId`/`WithTenantID`, per-call `tenant`/`WithRunTenant`) is an
  opaque `[A-Za-z0-9_-]{1,64}` string sent as `X-TENANT-ID` for per-customer metrics and
  fairness.
- Idempotency (`idempotency_key=` / `idempotencyKey` / `WithIdempotencyKey`,
  `WithSubmitIdempotencyKey`) is sent as `Idempotency-Key`; use a stable business id so a
  retried HTTP call admits one run. TypeScript `maxRetries` defaults to 0 - enable it only
  with an idempotency key.

## Batch and inline eval

`client.batch(component, items, max_concurrency=10, continue_on_failure=True, ...)` /
`client.batch(...)` / `client.Batch(ctx, name, items, agnt5.WithBatchMaxConcurrency(5), ...)`
runs many inputs as one batch (`get_batch_status`, `cancel_batch`). `client.eval` /
`batch_eval` (Go `Eval`, `BatchEval` with `agnt5.NormalizeEvalScorers(agnt5.Correctness{},
"exact_match")`) score outputs inline - see `agnt5-experiments` and `agnt5-scorers`.

## Human-in-the-loop, signals, cancel

A workflow paused on `wait_for_user` reports `awaiting_user_input` (Python
`RunStatus.AWAITING_USER_INPUT`); a serverless workflow waiting on a signal reports its
suspension. Only Go has client methods (`client.ResumeWorkflow(ctx, runID, "approve")`,
`client.CancelRun(ctx, runID, reason)`); Python and TypeScript call the gateway directly:

```bash
# answer a wait_for_user pause; user_response arrives in the workflow as a string
curl -X POST "$AGNT5_GATEWAY_URL/v1/workflows/resume/<run_id>" \
  -H "X-API-KEY: $AGNT5_API_KEY" -H "Content-Type: application/json" \
  -d '{"user_response": "approve"}'
# deliver a signal to a serverless workflow waiting on ctx.wait_for_signal("payment.settled")
curl -X POST "$AGNT5_GATEWAY_URL/v1/runs/<run_id>/signals/payment.settled" \
  -H "X-API-KEY: $AGNT5_API_KEY" -H "Content-Type: application/json" -d '{"payload": {"reference": "pay_123"}}'
curl -X POST "$AGNT5_GATEWAY_URL/v1/runs/<run_id>/cancel" -H "X-API-KEY: $AGNT5_API_KEY"
```

Auth is `X-API-KEY` (`agnt5_sk_` service key or `agnt5_uk_` user key), or
`Authorization: Bearer <token>` plus `X-WORKSPACE-ID` and `X-PROJECT-ID`. Resume returns
404 for an unknown run and 409 unless the run is `paused`. Answer formats (live-verified
29 Sep 2026 for TypeScript/Go): text/approval/select send the plain string or option id;
multiselect sends a JSON array string (`"[\"a\",\"b\"]"`); skip sends `"__skipped__"`; any
non-string JSON is stringified, so `null` arrives as the string `"null"`, not a skip. Details
and replay rules: `agnt5-human-in-the-loop`. Agent approvals use
`POST /v1/runs/{run_id}/approvals/{approval_id}/approve|reject` with optional
`{"decided_by", "reason", "payload"}`.

## Raw REST (what the clients call)

`POST /v1/{functions|workflows|agents|tools}/{name}/run` (JSON body = input; header
`X-AGNT5-Wait-Timeout-Ms`), `.../submit`, `.../stream` (SSE), `.../batch`,
`GET /v1/status/{run_id}`, `GET /v1/result/{run_id}`, `GET /v1/runs/{run_id}/events`,
`POST /v1/eval`, `GET|DELETE /v1/batches/{batch_id}`, `POST /v1/agents/{name}/chat`. Studio's
**copy curl** on a component pre-fills `X-API-KEY`, `X-WORKSPACE-ID`, and `X-DEPLOYMENT-ID`.

## Local targets

- `agnt5 dev up` (container stack) exposes the gateway on `http://localhost:34181` - set
  `AGNT5_GATEWAY_URL` to it.
- `agnt5 dev` (cloud-connected worker) is reached by `agnt5 run` without `--env`, which adds
  `X-Dev-Mode: true`. Python `run(headers={"X-Dev-Mode": "true"})` and Go
  `agnt5.WithRunHeader("X-Dev-Mode", "true")` can send the same header; whether the gateway
  honours it for API-key clients is **unverified** - prefer `agnt5 run` or a preview
  deployment for SDK-client tests (`agnt5-testing`).

Per-language method lists: `references/python.md`, `references/typescript.md`, `references/go.md`.

## Source

https://agnt5.com/docs/build/local-development · https://agnt5.com/docs/improve/batch-eval · https://agnt5.com/docs/build/human-in-the-loop
