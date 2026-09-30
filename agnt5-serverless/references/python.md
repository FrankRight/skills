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

`serve()` builds the component list and the manifest **when it is called**: with a list set to
`None` it copies the registry at that moment. Define or import every component above the
`serve()` call. A workflow defined later is registered but missing from `app.manifest()`, and
invoking it returns 404 `WORKERLESS_COMPONENT_NOT_FOUND`.

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
| `await ctx.wait_for_signal(signal_name, name=None)` | Returns the payload found in `metadata["signals"]` (a JSON string keyed `"<signal_name>:<waiting_step>"`), else suspends; `name` is the waiting-step label (defaults to the signal name). The gateway does not send `signals`; see below |
| `await ctx.emit(event_type, data, metadata=..., step_key=..., data_type=...)` | Also accepts a dict or an SDK `Event`; returned with the response |
| `checkpoint_snapshot()`, `set_checkpoint(key, value)`, `events_snapshot()` | Low-level; the adapter uses these to build the response |

Suspensions are `BaseException` subclasses (`WorkerlessSuspension`,
`WorkerlessWaitingForUserInput`) so a bare `except Exception` does not swallow them - do not
catch `BaseException` inside a workflow.

**Signals from the gateway.** `POST /v1/runs/{run_id}/signals/{name}` resumes the run with
`signal_name`, `waiting_step` and `signal_payload` (a JSON string) in the invoke metadata;
0.13.6 only looks at `metadata["signals"]`, so `ctx.wait_for_signal` keeps suspending. Read the
gateway keys first, and wrap the wait in a step so the payload is checkpointed (only the
latest signal stays in the metadata):

```python
import json
from typing import Any

from agnt5 import workflow


async def wait_for_signal(ctx, signal_name: str, name: str | None = None) -> Any:
    """Return a signal delivered by the AGNT5 gateway; otherwise defer to the SDK."""
    step = name or signal_name
    md = ctx.metadata
    if md.get("signal_name") == signal_name and md.get("waiting_step", signal_name) == step \
            and "signal_payload" in md:
        return json.loads(md["signal_payload"])
    return await ctx.wait_for_signal(signal_name, name=name)


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
    payment = await ctx.step(
        "payment", lambda: wait_for_signal(ctx, "payment.settled", name="await-payment"))
    await ctx.step("ship", lambda: ship(order_id, payment["reference"]))
    await ctx.emit("order.shipped", {"order_id": order_id})
    return {"status": "shipped"}
```

**One question per workflow.** Answers are not checkpointed, and 0.13.6 reads a single
`pause_index` / `user_response` pair per invoke (it ignores `step_events`). After a second
`wait_for_user` is answered, the replay finds no answer for the first one and asks it again,
so a Python serverless workflow cannot get past two questions. Ask one question (a
`multiselect` can collect several choices) or split the flow.

## Local run and offline test

The scaffold ships no `pyproject.toml`, so its `uv add` hint fails until you create one:

```bash
uv init --bare && uv add agnt5 fastapi uvicorn
export AGNT5_SERVERLESS_SIGNING_SECRET="$(openssl rand -base64 32)"
uv run uvicorn agnt5_serverless:app --host 127.0.0.1 --port 8787
agnt5 serverless validate http://127.0.0.1:8787
```

`ServerlessApp.handle_http(method=, path=, headers=, body=, url=)` returns
`(status, payload, headers)` and needs no server. Without a secret it accepts unsigned bodies;
the scaffold's resolver reads `AGNT5_SERVERLESS_SIGNING_SECRET`, so unset it in tests or sign
the request. Header names must be lowercase on this path. To replay, send the previous
`checkpoint` back with the resume `metadata` (string values):

| Resume | `metadata` |
|---|---|
| Answer to question N | `{"pause_index": "N", "user_response": "approve"}`; `"__skipped__"` skips |
| Signal, as the SDK reads it | `{"signals": "{\"payment.settled:await-payment\": {\"reference\": \"pay_123\"}}"}` (a JSON string keyed `<signal>:<waiting_step>`) |
| Signal, as the gateway sends it | `{"signal_name": "payment.settled", "waiting_step": "await-payment", "signal_payload": "{\"reference\": \"pay_123\"}"}` (read by the helper above) |

The checkpoint does not hold answers: send an earlier answer again with every later invoke,
or the replay asks the question again.

```python
# test_serverless.py - uv run --with pytest pytest -q
import asyncio
import hashlib
import hmac
import json
import time

from agnt5_serverless import agnt5_workerless as app   # the ServerlessApp returned by serve()


def signed_headers(secret: str, body: bytes, attempt_id: str = "r1:0") -> dict[str, str]:
    timestamp = str(int(time.time() * 1000))            # Unix milliseconds
    signature = hmac.new(secret.encode(), f"{timestamp}.{attempt_id}.".encode() + body,
                         hashlib.sha256).hexdigest()
    return {"x-agnt5-signature": f"sha256={signature}",
            "x-agnt5-signature-version": "workerless-hmac-sha256.v1",
            "x-agnt5-timestamp": timestamp, "x-agnt5-attempt-id": attempt_id}


def invoke(body: dict, secret: str | None = None) -> dict:
    raw = json.dumps(body).encode()
    headers = signed_headers(secret, raw) if secret else {}
    _, payload, _ = asyncio.run(
        app.handle_http(method="POST", path="/agnt5/invoke", headers=headers, body=raw))
    return payload


def test_question_then_signal() -> None:
    base = {"run_id": "r1", "component_type": "workflow", "component_name": "approve_order",
            "input": {"order_id": "o-1"}}
    asked = invoke(base)
    assert asked["reason"] == "user_input_required"
    answer = {"pause_index": "0", "user_response": "approve"}
    waiting = invoke({**base, "checkpoint": asked["checkpoint"], "metadata": answer})
    assert waiting["reason"] == "signal"
    signal = {"signal_name": "payment.settled", "waiting_step": "await-payment",
              "signal_payload": json.dumps({"reference": "pay_123"})}
    done = invoke({**base, "checkpoint": waiting["checkpoint"], "metadata": {**answer, **signal}})
    assert done["status"] == "completed"
```

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
