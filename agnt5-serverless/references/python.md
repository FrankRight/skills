# Python serverless reference

Verified against agnt5 0.13.6 (`src/agnt5/serverless.py`) and the September 2026 CLI.

## `serve()` / `ServerlessApp`

```python
from agnt5.serverless import serve, ServerlessApp, WorkerlessContext

app = serve(
    service_name="orders-api",            # manifest service_name
    service_version=os.getenv("GIT_SHA"),  # immutable provider version (Vercel scaffold uses VERCEL_DEPLOYMENT_ID)
    functions=None, workflows=None, tools=None, agents=None,   # None = everything registered; [] = none
    signing_secret=lambda: os.getenv("AGNT5_SERVERLESS_SIGNING_SECRET"),  # str | () -> str | (headers) -> str, may be async
    enabled=True,                          # bool | () -> bool | (headers) -> bool; False -> 503 WORKERLESS_DISABLED
)
```

`serve()` returns a `ServerlessApp`, which is itself an ASGI application. Mount options:

| Framework | Call | Adds |
|---|---|---|
| FastAPI | `app.mount_fastapi(fastapi_app)` | `GET /.well-known/agnt5`, `POST /agnt5/invoke` (hidden from OpenAPI) |
| Starlette | `app.mount_starlette(starlette_app)` | same two routes |
| Flask | `app.mount_flask(flask_app)` | two URL rules; async workflows run through the WSGI bridge |
| Django | `urlpatterns = [..., *app.django_urlpatterns()]` | async views for both routes |
| Raw ASGI | `application = app` | run with `uvicorn module:application` |
| Raw WSGI | `application = app.wsgi_app` | WSGI callable |

Manifest at `app.manifest()`. Constants: `agnt5.serverless.DEFAULT_MANIFEST_PATH`
(`/.well-known/agnt5`), `INVOKE_PATH` (`/agnt5/invoke`), `PROTOCOL_VERSION` (`workerless.v1`).
Third-party call capture (`agnt5.integrations.auto_enable`) is switched on when the app is
built, like the worker path.

## How components are invoked

- `@workflow` handlers receive a `WorkerlessContext` as `ctx` and the JSON input as keyword
  arguments (`handler(ctx, **input)`); a non-object input is passed positionally. Handlers
  whose first parameter is not named `ctx` are called without a context.
- `@function` handlers receive a `FunctionContext` subclass bound to the invoke (events go into
  the response). Retries declared with `@function(retries=...)` are emitted as the manifest
  `flow_control.retry` and enforced by AGNT5, not locally.
- Tools are called as `config.invoke(ctx, **arguments)`.
- Agents run through `agent.stream(message, context=..., history=...)`; history comes from the
  checkpoint (`agent_sessions[session_id]`) or the input's `history`, and the session id from
  `input["session_id"]`, then invoke metadata, then the run id.

## `WorkerlessContext` API

| Member | Notes |
|---|---|
| `run_id`, `attempt`, `component_name`, `invocation_id`, `metadata`, `logger` | Read-only invoke facts |
| `await ctx.step(name, func_or_awaitable)` | Checkpoint key `step:<name>`; replays return the stored JSON result. `func` may be sync, async, or an awaitable. No `key=`, no `*args` |
| `await ctx.get(key, default)`, `await ctx.set(key, value)`, `await ctx.delete(key)` | In-memory for this invoke only - **not** in the checkpoint |
| `await ctx.sleep(seconds, name=None)` | Records start time in a step, then raises a timer suspension until `ready_at_ms` |
| `await ctx.yield_if_needed(reason="budget")` | Raises a budget suspension when `now + yield_before_timeout_ms >= deadline_ms` |
| `await ctx.wait_for_user(question, *, input_type="text", options=None, allow_custom=False, skippable=False)` | Returns the answer string (`None` when skipped); pause index tracked per call |
| `await ctx.wait_for_signal(signal_name, name=None)` | Returns the signal payload once delivered; `name` is the waiting-step label (defaults to the signal name) |
| `await ctx.emit(event_type, data, metadata=..., step_key=..., data_type=...)` | Also accepts a dict or an SDK `Event`; returned with the response |
| `checkpoint_snapshot()`, `set_checkpoint(key, value)`, `events_snapshot()` | Low-level; the adapter uses these to build the response |

Suspensions are `BaseException` subclasses (`WorkerlessSuspension`,
`WorkerlessWaitingForUserInput`) so a bare `except Exception` does not swallow them - do not
catch `BaseException` inside a workflow.

```python
@workflow
async def approve_order(ctx, order_id: str) -> dict:
    order = await ctx.step("load-order", lambda: load_order(order_id))
    await ctx.yield_if_needed()
    decision = await ctx.wait_for_user(
        f"Ship order {order_id} for {order['total']}?", input_type="approval",
        options=[{"id": "approve", "label": "Approve"}, {"id": "reject", "label": "Reject"}],
    )
    if decision != "approve":
        return {"status": "rejected"}
    payment = await ctx.wait_for_signal("payment.settled", name="await-payment")
    await ctx.step("ship", lambda: ship(order_id, payment["reference"]))
    await ctx.emit("order.shipped", {"order_id": order_id})
    return {"status": "shipped"}
```

## Local run and offline test

```bash
export AGNT5_SERVERLESS_SIGNING_SECRET="$(openssl rand -base64 32)"
uv run uvicorn agnt5_serverless:app --host 127.0.0.1 --port 8787
agnt5 serverless validate http://127.0.0.1:8787
```

`ServerlessApp.handle_http(method=, path=, headers=, body=, url=)` returns
`(status, payload, headers)` and needs no server; with no `signing_secret` it accepts unsigned
bodies, so a unit test can post
`{"protocol_version": "workerless.v1", "run_id": "r1", "component_type": "workflow",
"component_name": "hello", "input": {"name": "Ada"}}` and assert on `payload["output"]`. A
second call with `"checkpoint": payload["checkpoint"]` exercises replay.

## Hosts

- **Cloud Run**: `agnt5 serverless init --provider cloud-run --runtime python`; the service reads
  `PORT` and uses `K_REVISION` as the version; sync `--immutable-ref <revision-name>`.
- **Vercel** (Python runtime beta): `--provider vercel --runtime python` generates a FastAPI
  `app.py`; add a supported Python version to `pyproject.toml` or `.python-version`;
  `--immutable-ref` defaults to `VERCEL_DEPLOYMENT_ID`.
- **AWS Lambda Web Adapter** (preview): `--provider aws-lambda --runtime python`; Function URL
  with `AuthType NONE` (AGNT5 does not sign with SigV4; HMAC is the auth); the manifest is public.
- **Generic ASGI/WSGI host**: `--provider http`, sync `--immutable-ref <git-sha>`;
  `--provider fastapi`/`python` remain aliases for `http --runtime python`.
