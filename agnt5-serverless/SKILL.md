---
name: agnt5-serverless
description: Run AGNT5 workflows, functions, tools, and agents as a signed HTTP endpoint instead of a persistent worker - Python agnt5.serverless.serve/ServerlessApp (FastAPI, Starlette, Flask, Django, ASGI/WSGI), TypeScript @agnt5/sdk/serverless serve/serveNode/serveCloudflare (Cloudflare Workers, Vercel/Next.js, Express), Go sdk-go/serverless (net/http, Cloud Run), the serverless context (step, get/set, sleep, yield, waitForUser, waitForSignal, emit), flow-control policies, AGNT5_SERVERLESS_SIGNING_SECRET, and the agnt5 serverless init/validate/sync/status/enable/disable lifecycle. Use when deploying to a serverless host, mounting AGNT5 routes into an existing web app, porting a worker workflow to serverless, or when a workflow must wait for an external signal.
---

# AGNT5 Serverless endpoints

A **serverless endpoint** is your own HTTPS service that serves an AGNT5 manifest and accepts
signed invoke requests. AGNT5 owns dispatch, retries, checkpoints, and the run journal; the
provider (Cloudflare, Vercel, Cloud Run, Lambda, any HTTPS host) owns scaling and deploys.
There is no `agnt5 deploy` and no worker process. Beta: serverless dispatch must be enabled
for your workspace.

Pick this over a managed worker (`agnt5-deploy`) when you already run a web app, want
per-request billing, or need `waitForSignal`. Stay on a worker when you need fan-out
(`parallel`/`gather`/`batch`/`map`), run/session/user state, or batch flow-control policies.

## The contract: two routes, one secret

| Route | Purpose |
|---|---|
| `GET /.well-known/agnt5` | Manifest: `protocol_version: "workerless.v1"`, `service_name`, `service_version`, `components[]` |
| `POST /agnt5/invoke` | One signed invoke; returns `completed`, `failed`, or `suspended` plus a checkpoint |

Invokes are HMAC-SHA256 signed: `X-AGNT5-Signature: sha256=<hex>` over
`"{X-AGNT5-Timestamp}.{X-AGNT5-Attempt-ID}." + body`, with
`X-AGNT5-Signature-Version: workerless-hmac-sha256.v1` and a 5-minute skew window.
`X-AGNT5-Timestamp` is Unix time in **milliseconds** in all three SDKs; a test that signs with
seconds gets 401 `WORKERLESS_SIGNATURE_EXPIRED`. AGNT5 sends `{run_id}:{attempt}` as the
attempt ID. The SDK verifies for you, **but only if your resolver returns a secret** - with no
secret configured all three SDKs accept unsigned invokes. Generate one and use the same value
on both sides:

```bash
( umask 077 && openssl rand -base64 32 > .agnt5-serverless-secret )   # keep out of git
export AGNT5_SERVERLESS_SIGNING_SECRET="$(cat .agnt5-serverless-secret)"
```

No SDK reads `AGNT5_SERVERLESS_SIGNING_SECRET` itself; your resolver does (`os.getenv`,
`process.env`, the Worker `env` binding, `os.Getenv`). An optional `enabled` resolver returns
`503 WORKERLESS_DISABLED` to fail closed during an incident.

## Minimal endpoint

`agnt5 serverless init --provider <http|vercel|cloudflare|cloud-run|aws-lambda> --runtime
<typescript|python|go>` scaffolds exactly these (never overwrites without `--force`).

```python
# Python: agnt5_serverless.py (http/python scaffold) - see references/python.md
import os
from fastapi import FastAPI
from agnt5 import workflow
from agnt5.serverless import serve

app = FastAPI()

@workflow
async def hello(ctx, name: str = "world") -> dict[str, str]:
    return {"message": f"hello {name}"}

agnt5_serverless = serve(
    service_name="orders-api",
    service_version=os.getenv("GIT_SHA", "local"),
    signing_secret=lambda: os.getenv("AGNT5_SERVERLESS_SIGNING_SECRET"),
)
agnt5_serverless.mount_fastapi(app)      # also mount_starlette / mount_flask / django_urlpatterns / wsgi_app
```

```typescript
// TypeScript: src/agnt5-workerless.ts (Cloudflare scaffold) - see references/typescript.md
import { serve, workflow } from '@agnt5/sdk/serverless';

interface Env { AGNT5_SERVERLESS_SIGNING_SECRET?: string }

const hello = workflow('hello', async (_ctx, input: { name?: string }) => ({
  message: `hello ${input.name ?? 'world'}`,
}));

export default serve<Env>({
  serviceName: 'orders-api',
  serviceVersion: 'local',
  signingSecret: (_request, env) => env?.AGNT5_SERVERLESS_SIGNING_SECRET,
  workflows: [hello],
});
// Node/Express: serveNode from '@agnt5/sdk/serverless/node'; Vercel: route handlers call handler.fetch(request)
```

```go
// Go: cmd/agnt5-serverless/main.go (http/go scaffold) - see references/go.md
handler := serverless.New(serverless.Options{
    ServiceName: "orders-api", ServiceVersion: os.Getenv("GIT_SHA"),
    SigningSecret: func(*http.Request) string { return os.Getenv("AGNT5_SERVERLESS_SIGNING_SECRET") },
})
_ = serverless.RegisterWorkflow(handler, "hello", func(ctx *serverless.Context, in helloInput) (map[string]string, error) {
    name, err := serverless.Step(ctx, "normalize-name", func(context.Context) (string, error) { return in.Name, nil })
    return map[string]string{"message": "hello " + name}, err
})
log.Fatal(http.ListenAndServe("127.0.0.1:8787", handler))   // handler is an http.Handler; containers listen on ":"+port
```

- **Python `serve()` snapshots the registries when it runs.** Define or import every component
  before calling it: a workflow defined after `serve()` is missing from the manifest, and its
  invokes get 404 `WORKERLESS_COMPONENT_NOT_FOUND`.
- **Go scaffold has no `go.mod`.** Run `go mod init <module>`,
  `go get github.com/agnt5dev/sdk-go@v0.10.3` and `go mod tidy`. sdk-go declares `go 1.26.5`,
  so `go get` raises an older `go` line to that; build with Go 1.26.5 or newer.
- **Go handlers that emit nothing answer `"events": null`** (sdk-go v0.10.3), and AGNT5
  rejects that response with `WORKERLESS_INVALID_RESPONSE`. Wrap the handler so `null` becomes
  `[]` (`references/go.md`).
- **Bind to `127.0.0.1` for local tests.** The Go and Node scaffolds listen on all interfaces;
  `references/go.md` and `references/typescript.md` show a `HOST` switch.

Component lists (`workflows=`, `functions=`, `tools=`, `agents=`) are optional in Python and
TypeScript: omit them to expose everything registered in the process, pass an explicit list to
expose only those (an empty list exposes none of that type). Go registers explicitly with
`RegisterWorkflow/RegisterFunction/RegisterTool/RegisterAgent`.

## The serverless context is not the worker context

Workflows written for a managed worker do **not** port unchanged. Every invoke gets a
request-scoped context that replays the checkpoint and may return a suspension instead of
finishing. Compare before porting:

| Capability | Managed worker | Serverless endpoint |
|---|---|---|
| Durable step | Py `ctx.step(fn, *args, key=)` or `ctx.step("name", fn)`; TS `ctx.step('name', fn, {key})`; Go `agnt5.Step`/`StepWithKey` | Py `await ctx.step("name", fn_or_awaitable)` (name-first only); TS `ctx.step('name', fn)`; Go `serverless.Step(ctx, name, fn)` |
| Fan-out | Py `ctx.parallel/gather/batch/map` | None. Sequential steps only (TS and Go never had fan-out) |
| State | Py `ctx.state`, `ctx.session.state`, `ctx.user.state`; Go `ctx.State()`, `ctx.Memory()` | Py/TS `await ctx.get/set/delete` - a plain map that lives for **one invoke** and is not checkpointed; Go has none. Anything needed after a suspension must be a step result |
| Sleep | `ctx.sleep(seconds, name=)` / `ctx.sleep(ms, name)` / `ctx.Sleep(d, opts...)` | Same names; returns a timer suspension and resumes on reinvoke (Py seconds, TS ms, Go `time.Duration` + name) |
| Human input | Py `ctx.wait_for_user`; TS `ctx.waitForUser`; Go `ctx.AskUser`/`RequestApproval` | Py `ctx.wait_for_user(question, input_type=, options=, allow_custom=, skippable=)`; TS `ctx.waitForUser(q, {inputType, options, allowCustom, skippable})`; Go `ctx.WaitForUser(serverless.UserInput{...})` |
| External signal | Py: no method; TS `ctx.waitForSignal` **throws** `ConfigurationError`; Go: no method | Py `await ctx.wait_for_signal(name, name=step)` (0.13.6 does not see gateway deliveries, see below); TS `await ctx.waitForSignal<T>(name, step?)`; Go `serverless.WaitForSignal[T](ctx, name, step)` |
| Budget | n/a | `await ctx.yield_if_needed()` / `ctx.yieldIfNeeded()` / `ctx.YieldIfNeeded()` |
| Model call helper | Go `ctx.Generate(model, req)` | Go: none - call `model.Generate(ctx, req)` (`*serverless.Context` embeds `context.Context`) |
| Events | `ctx.emit(...)` streamed live | `ctx.emit(...)` / `ctx.Emit(Event{...})` returned in the invoke response, appended to the journal |

The portable step form is name-first: Python `await ctx.step("load", lambda: load(user_id))`,
TypeScript `await ctx.step('load', () => load(userId))`. Keep step names stable across
releases; renaming one re-runs its side effect. Go worker handlers take `*agnt5.Context`,
serverless handlers take `*serverless.Context` - different types, so they do not compile
against each other.

## Suspensions: sleep, budget, signals, human input

Each invoke carries a budget (`deadline_ms`, `yield_before_timeout_ms`, set at sync with
`--request-timeout-ms` and `--yield-before-timeout-ms`). Call the yield method between chunks
of work; when the margin is reached the adapter returns `status: "suspended"` with its
checkpoint and AGNT5 reinvokes the same pinned deployment. Timers, signals, and user input
work the same way - the HTTP request never stays open.

- **Signal**: the workflow suspends with `reason: "signal"` and the run reports `paused`;
  deliver it with `POST {gateway}/v1/runs/{run_id}/signals/{signal_name}` and body
  `{"payload": <json>}` using a `workflow`-scoped key (`agnt5-client`). The gateway passes the
  payload to the next invoke as `signal_name` / `waiting_step` / `signal_payload` metadata,
  which TypeScript and Go read (Go decodes JSON into `T`, falling back to the raw string).
  **Python 0.13.6 looks for a `signals` metadata map instead, which the gateway does not send,
  so its `wait_for_signal` keeps suspending**; read the gateway keys yourself
  (`references/python.md`).
- **User input**: suspends with `reason: "user_input_required"`; answer from Studio or
  `POST /v1/workflows/resume/{run_id}` with `{"user_response": "..."}` (`agnt5-client`).
  Replay semantics are the same as on workers - checkpoint side effects before the pause
  (`agnt5-human-in-the-loop`).
- **Agents**: session history is stored in the invoke checkpoint (`agent_sessions`) and
  restored before the next turn. Go's `serverless.Agent{Name, Run}` is a plain runner, not
  `agnt5.Agent`.

**Replay carries step results, not answers or signals.** A resumed invoke replays the handler
with the checkpoint (step results) plus the metadata of the latest resume:

- An answered question needs its answer on every later invoke. On the platform the latest
  `pause_index` / `user_response` stay in the run's metadata, so a question followed by a
  signal works; an offline test must send them again with the signal.
- Only the latest signal is in the metadata. Put each signal wait inside a step so its payload
  is checkpointed and survives a later signal: Python
  `await ctx.step("payment", lambda: wait_for_signal(ctx, "payment.settled"))`, TypeScript
  `await ctx.step('payment', () => ctx.waitForSignal('payment.settled'))`.
- More than one question: TypeScript returns the earlier answers as `step_events` in each
  suspension, and AGNT5 hands them back. Python 0.13.6 reads one answer per invoke and Go
  v0.10.3 returns no `step_events`, so in those SDKs the second answer makes the replay ask the
  first question again. Keep Python and Go serverless workflows to one question.

Large inputs/outputs arrive and leave as signed object-store references; the SDKs resolve
them. Clients read big outputs with `client.resolveOutput()`/`waitForOutput()` (TypeScript).

## Flow control (declared in the manifest, enforced by the runtime)

TypeScript declares policies on the component: `fn('x').flowControl({...}).run(handler)` or
`workflow('x', handler, { flowControl: {...} })`. Keys (`flow-control.d.ts`):

| Policy | Shape |
|---|---|
| `retries` | `{ maxAttempts, initialIntervalMs, maxIntervalMs, backoff: 'constant'\|'linear'\|'exponential', multiplier }` |
| `concurrency` | `{ limit, scope: 'project'\|'deployment'\|'component'\|'workspace'\|'custom', key, keyExpression }` |
| `throttle`, `rateLimit` | `{ limit, periodMs \| windowMs, key, keyExpression }` (sliding window) |
| `debounce` | `{ windowMs, key, keyExpression }` (latest run wins) |
| `priority` | `'interactive' \| 'normal' \| 'batch'`, `{ level, expression }`, or a number |
| `singleton` | `{ key, keyExpression, mode: 'queue' }` |
| `idempotency` | `{ key, keyExpression, ttlMs }` |
| `batch` | `{ maxSize, windowMs }` - **rejected at sync during beta**; keep batch policies on workers |

Python emits only the retry policy from `@function(retries=...)`; Go emits none. There is no
Python or Go declaration API for the other policies in these versions.

## Lifecycle: init, validate, sync, verify, activate

There is no `agnt5 serverless activate` command. Activation is `sync --activate=true` (the
default); the promotion flow is sync twice with the same immutable ref.

```bash
agnt5 serverless init --provider cloudflare --runtime typescript --name orders-api   # once
# develop and deploy with the provider CLI: wrangler dev / vercel dev, wrangler deploy / vercel deploy --prod
agnt5 serverless validate http://127.0.0.1:8787             # local; read-only, no project needed
agnt5 serverless validate https://<host>                    # deployed manifest, protocol, components, hash
agnt5 serverless sync https://<host> --provider cloudflare --env production \
  --immutable-ref <provider-version> --signing-secret-env AGNT5_SERVERLESS_SIGNING_SECRET \
  --request-timeout-ms 10000 --yield-before-timeout-ms 1000 --activate=false
agnt5 serverless status --deployment-id <agnt5-deployment-id> --verify   # non-zero unless healthy
agnt5 run workflow hello --deployment-id <agnt5-deployment-id> --input '{"name":"Ada"}'   # test before routing
agnt5 serverless sync https://<host> --provider cloudflare --env production \
  --immutable-ref <provider-version> --signing-secret-env AGNT5_SERVERLESS_SIGNING_SECRET --activate=true
```

`sync` is idempotent per provider + immutable ref; a new ref creates a new AGNT5 deployment
version. `--verify` checks `manifest_health`, `dispatch`, `signing`, `protocol`, and
`manifest_compatibility`; use `--output json` in CI. Known issue: `manifest_compatibility` can
fail falsely - if `manifest_hash_changed` is `false` in the raw `status` output and the other
four pass, proceed. Other subcommands: `register` (attach an endpoint to the latest existing
deployment; no env/activation flags), `disable`/`enable` (aliases `pause`/`resume`;
`--reason`, `--allow-incompatible`), `provider-link create|list|delete` (deployment-complete
webhooks; exactly one `--webhook-secret*` flag required). Protected previews: pass
`--invoke-header-env x-vercel-protection-bypass=VERCEL_AUTOMATION_BYPASS_SECRET`.

## Deploy targets

| Host | Entrypoint | `init --provider` | `--immutable-ref` |
|---|---|---|---|
| Cloudflare Workers | `@agnt5/sdk/serverless` (`serve` or `serveCloudflare`) | `cloudflare` | Worker version ID (`wrangler versions list`) |
| Vercel / Next.js | `@agnt5/sdk/serverless` + `app/**/route.ts`, or `agnt5.serverless` (FastAPI, Python runtime beta) | `vercel` | deployment ID (defaults to `VERCEL_DEPLOYMENT_ID`) |
| Cloud Run | `agnt5.serverless` or Go `serverless` | `cloud-run` | revision name (`K_REVISION`) |
| AWS Lambda Web Adapter | `agnt5.serverless` (preview; Function URL `AuthType NONE`, HMAC is the auth) | `aws-lambda` | function version or image digest |
| Any Node.js server | `@agnt5/sdk/serverless/node` (`serveNode`) | `http` | git SHA / release ID |
| Any Python ASGI/WSGI | `agnt5.serverless` | `http` | git SHA / release ID |
| Any Go `net/http` | `github.com/agnt5dev/sdk-go/serverless` | `http` | git SHA / release ID |

Keep `--request-timeout-ms` below the provider's duration cap and size Next.js
`export const maxDuration` for the full agent loop. The TypeScript SDK needs the Node runtime
(no Edge), `serverExternalPackages: ['@agnt5/sdk', 'better-sqlite3']`, and `next build
--webpack` on Next 16.

## Operate

- **Route conflicts**: one active owner per `workflow:<name>` per environment. Test with
  `--activate=false` + `--deployment-id`, rename the component, or take over with
  `--replace-route-owners`. Runs already pinned to a deployment stay pinned.
- **Incident**: `agnt5 serverless disable --deployment-id <id> --reason "..."` stops new
  dispatch and keeps state; `validate`, then `enable`, then `status --verify`.
- **Rotate the secret** as a new release: new secret in the provider, deploy a new immutable
  version, `validate`, `sync --activate=false`, `status --verify`, activate; keep the old
  deployment disabled until its runs drain. Prefer `--signing-secret-env` or
  `--signing-secret-ref secret:<id>` over `--signing-secret <value>`.
- **Roll back**: disable the bad deployment, roll the provider back (`wrangler rollback`),
  `validate`, `sync` the previous ref with `--activate=false`, verify, activate.
- **Invoke timeouts**: lower `request_timeout_ms` below the provider cap and reserve
  `yield_before_timeout_ms`; add yields inside long loops.

Per-language detail: `references/python.md`, `references/typescript.md`, `references/go.md`.

## Source

https://agnt5.com/docs/run/serverless-endpoints · https://agnt5.com/docs/run/serverless-sdk-reference · https://agnt5.com/docs/run/operate-serverless-endpoints · https://agnt5.com/docs/cli/serverless
