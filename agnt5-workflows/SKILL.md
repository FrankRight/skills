---
name: agnt5-workflows
description: Define AGNT5 functions and workflows - @function with retries, backoff, and timeouts (retries=N is N attempts in total; not applied inside ctx.step in 0.13.6); durable workflow steps via ctx.step() with stable keys, checkpointing arbitrary awaitables, parallel/gather/batch/map fan-out, durable sleep, cron-scheduled and chat workflows, run/session/user state and how session_id/user_id reach a run, progress/output events (ctx.emit), streaming async-generator functions, and idempotency keys for side effects. Use when wrapping a unit of work in a retryable/timeout-bound function, adding or restructuring workflow orchestration, composing functions and agents into a pipeline, scheduling a workflow on a cron, making a flow durable so it resumes after a crash without repeating side effects, or debugging a step that re-runs, a retry that never happens, or a TypeError about FunctionContext.
---

# AGNT5 Workflows

> **TypeScript or Go?** This file shows the Python API. Read [references/typescript.md](references/typescript.md) or [references/go.md](references/go.md) first: same sections, the exact signatures for that SDK, and what it does not support. Known SDK bugs with workarounds are in the `agnt5-sdk-pitfalls` skill.

A **workflow** is a durable orchestrator: if it crashes mid-run, it restarts and resumes from
the last completed step instead of starting over. A **function** is the stateless unit of
work a workflow checkpoints around.

## Defining a workflow

```python
from agnt5 import workflow, WorkflowContext

@workflow
async def onboarding_workflow(ctx: WorkflowContext, user_email: str) -> dict:
    account = await ctx.step(create_account, user_email)
    await ctx.step(send_welcome_email, account["id"])
    return {"status": "done", "account_id": account["id"]}
```

First parameter must be `ctx: WorkflowContext`; everything after is the input the caller
passes. `@workflow` options:

| Option | Meaning |
|---|---|
| `name=` | Explicit name (defaults to the function name) |
| `cron="0 9 * * *"` | Run on a schedule |
| `chat=True` | Multi-turn conversation workflow (one session, many runs) |
| `triggers=[event(...), webhook(...)]` | Event/webhook triggers — see `agnt5-webhooks-integrations` |

## Defining a function

```python
from agnt5 import function, FunctionContext

@function(name="send_email", retries=3, backoff="exponential", timeout_ms=10000)
async def send_email(ctx: FunctionContext, to: str, subject: str, body: str) -> str:
    ctx.logger.info("Sending email", to=to, attempt=ctx.attempt)
    return f"Sent to {to}"
```

`@function` parameters: `name`, `retries` (`int | RetryPolicy`), `backoff`
(`"constant" | "linear" | "exponential"`), `timeout_ms`. `FunctionContext` gives you
`ctx.run_id`, `ctx.attempt` (0 = first try), `ctx.logger`, `ctx.sleep(seconds)` (a plain,
**non-durable** sleep — only `WorkflowContext.sleep` survives restarts). Sync functions are
auto-wrapped in a thread pool. `@function` dispatches on the parameter **name** `ctx` (the
annotation is optional) — name it anything else and your first real argument is treated as
the context. `@workflow` always passes the context first, but only a parameter named `ctx`
is left out of the generated input schema, so use `ctx` everywhere.

Retry semantics (0.13.6):

- `retries=N` means **N attempts in total** (`RetryPolicy(max_attempts=N)`); the platform,
  not the SDK, re-runs the function. Every exception is retried — there is no
  "non-retryable" exception type, so validate inputs before the side effect.
- Retries apply when the function runs on its own (`agnt5 run`, `Client.run`), **not** when a
  workflow calls it through `ctx.step()` — the step gets the first attempt's error
  (AGNT5-1372). Loop inside the function body if a step needs retries today.
- `agnt5 run <function>` prints the first failed attempt and exits 1 while the platform keeps
  retrying — check the stored run (`agnt5 inspect runs describe <run-id>`) for the outcome.

## Steps: the unit of durable work

Use `ctx.step()` to call a `@function` from inside a workflow. The result is checkpointed
after the first successful run — on replay, the cached result returns immediately and the
function does **not** run again.

| Call style | Checkpointed | Use when |
|---|---|---|
| `await ctx.step(fn, *args, key="...")` | Yes | Always, inside a workflow |
| `await ctx.run(fn, *args)` | Yes | Alias for `ctx.step` |
| `await ctx.step("name", awaitable_or_callable, *args)` | Yes | Checkpoint an arbitrary async call (API request, plain coroutine) that is not a `@function` |
| `await fn(ctx, *args)` | No | Only with a `FunctionContext` (a function calling another function). With a `WorkflowContext` it raises `TypeError: Function 'fn' requires FunctionContext as first argument` |

Pass a stable `key=` whenever steps run concurrently, repeat in a loop, or may be reordered by
a code change — without one, keys come from call order, and replay can match the wrong
checkpoint. `key=` is consumed by `ctx.step` itself, so a function parameter literally named
`key` cannot be passed by keyword through a step (`ctx.step(fn, key="x")` fails with
`TypeError: fn() missing 1 required positional argument: 'key'`) — rename it or pass it
positionally.

> `ctx.task()` is **deprecated** — if you see it in older example code, replace it with
> `ctx.step()`.

```python
@workflow
async def order_workflow(ctx: WorkflowContext, order_id: str) -> dict:
    order = await ctx.step(validate_order, order_id)
    if not order["valid"]:
        return {"status": "invalid", "order_id": order_id}

    # Checkpointed: never charged twice even if the workflow restarts here
    charge = await ctx.step(charge_customer, order_id, order["total"])
    await ctx.step(send_confirmation, order_id, charge["charge_id"])
    return {"status": "complete", "order_id": order_id, "charge_id": charge["charge_id"]}
```

## Running in parallel

| Method | Use when… |
|---|---|
| `ctx.parallel(*tasks)` | Small fixed number of independent stages (2-5); results come back in call order |
| `ctx.gather(**tasks)` | Same as `parallel()` but you want results addressable by name |
| `ctx.batch(func, items, max_concurrency=10)` | Many items (10+), need concurrency control and per-item error handling |
| `ctx.map(func, items, max_concurrency=10)` | Like `batch()` but you only need the outputs and want to fail-fast |

```python
# parallel — positional, ordered
sales, inventory, customers = await ctx.parallel(
    ctx.step(fetch_sales, report_id, key="sales"),
    ctx.step(fetch_inventory, report_id, key="inventory"),
    ctx.step(fetch_customers, report_id, key="customers"),
)

# gather — named (wrap each call in ctx.step so it is checkpointed)
data = await ctx.gather(
    revenue=ctx.step(fetch_revenue, key="revenue"),
    users=ctx.step(fetch_active_users, key="users"),
)
# data["revenue"], data["users"]

# batch — many items, controlled concurrency, partial failure tolerant
result = await ctx.batch(
    process_document,
    [{"doc_id": d} for d in doc_ids],
    max_concurrency=20,
    continue_on_failure=True,
    timeout_per_item=30.0,
)
# result.stats.completed_items, result.stats.failed_items, result.outputs

# map — simpler wrapper, raises on any failure
outputs = await ctx.map(process_document, [{"doc_id": d} for d in doc_ids])
```

Items that are not dicts are wrapped as `{"value": item}`, so the function must accept
`value=`. `timeout_per_item` defaults to 30 s; `map` is `batch(..., continue_on_failure=False)`.
`BatchResult` exposes `.outputs` (`None` for failed items), `.successful_outputs`,
`.failed_items`, and `.stats.completed_items` / `.stats.failed_items`.

## Durable sleep

`ctx.sleep(seconds, name=...)` pauses the workflow and survives restarts — if the worker
crashes mid-wait, it resumes and sleeps only for the remaining time (unlike `asyncio.sleep`).

```python
await ctx.step(send_confirmation, user_id)
await ctx.sleep(24 * 60 * 60, name="wait_24h")
await ctx.step(send_follow_up, user_id)
```

## State

| Scope | Access | Persists |
|---|---|---|
| Run | `ctx.state` (sync `get`/`set`; `await set_async` in async code) | Current run only |
| Session | `ctx.session.state` (async) | Across runs sharing the same `session_id` |
| User | `ctx.user.state` (async) | Across all runs for the same `user_id` |

```python
await ctx.state.set_async("phase", "started")   # ctx.state.set(...) is sync — don't await it
phase = ctx.state.get("phase")
count = await ctx.session.state.get("visit_count", 0)
await ctx.session.state.set("visit_count", count + 1)
if ctx.user:                                     # None when the run has no user_id
    await ctx.user.state.set("last_seen", "today")
```

`ctx.user` is `None` when the run was started without a `user_id`, so `ctx.user.state` raises
`AttributeError` — guard it. Without a `session_id` the session scope is the run id:
`ctx.session.state` works, but nothing carries over to the next run. The ids come from the
caller: `Client.run(..., session_id=..., user_id=...)` sends them as `X-Session-ID` /
`X-User-ID` headers (see `agnt5-client`); `session_id` / `user_id` keys inside the input
payload are honored too, but the worker leaves them in the kwargs, so the handler must accept
them. `agnt5 run` has no session/user flag.

For agent memory (`ctx.memory`, `ctx.conversation`), see `agnt5-agents-tools`.

## Idempotent side effects

Steps are checkpointed, but a step can still run twice if the worker dies after the side effect
and before the checkpoint. Pass the activation's idempotency key to downstream APIs that
support one:

```python
@function(retries=3)
async def charge_customer(ctx: FunctionContext, order_id: str, total: int) -> dict:
    key = ctx.activation.idempotency_key if ctx.activation else f"charge:{order_id}"
    return await stripe_charge(order_id, total, idempotency_key=key)
```

## Triggers

```python
@workflow(name="daily_report", cron="0 9 * * *")
async def daily_report(ctx: WorkflowContext) -> dict: ...
```

For event and webhook triggers (`triggers=[event(...)]`, `webhook(...)`, payload envelope,
signature verification), use `agnt5-webhooks-integrations`. For pausing a workflow on
`ctx.wait_for_user()`, use `agnt5-human-in-the-loop`.

## Progress events and streaming output

Emit typed events from any context; they appear on the run's live event stream (Studio,
`agnt5 run`, `Client.stream_events`). Every event needs a `name` and the two correlation ids:

```python
from agnt5 import ProgressUpdate

ctx.emit(ProgressUpdate(name="ingest", correlation_id=ctx.correlation_id,
                        parent_correlation_id=ctx.parent_correlation_id,
                        message="Parsed 40 of 120 pages", percent=33.0, current=40, total=120))
```

Also `OutputDelta(content=..., index=0)` for incremental output and
`StateChanged(key=..., value=..., operation="set")`. A `@function` written as an async
generator streams: each yielded `str` / `bytes` / `dict` chunk becomes an `output.delta`
event (yield `Event` objects to emit typed events instead). Deltas are **transient** — not
stored, not replayed on reconnect — so return the final result as well; only lifecycle,
step and tool/model boundary events are durable.

## Suspensions are `BaseException`s

`ctx.wait_for_user()` and `ctx.sleep()` pause the workflow by raising
`WaitingForUserInputException` / `DurableSleepSuspension`, which subclass `BaseException`. A
bare `except:` or `except BaseException:` around the pause — or around a tool or agent that
pauses — swallows it and the workflow carries on with no answer. Catch `Exception` only.

## Testing a workflow without a worker

Calling a decorated workflow directly (`await my_workflow(name="x")`) auto-creates an
in-memory context, but only **keyword** arguments are forwarded — positional input fails with
`TypeError: missing 1 required positional argument`. See `agnt5-testing` for the full local
setup.

## Serverless workflows

Workflows served by `ServerlessApp` (no worker) get a different context: `ctx.step("name", fn)`,
`ctx.get` / `set` / `delete`, `sleep`, `wait_for_user`, `wait_for_signal`, `emit` — no
`parallel` / `gather` / `batch` / `map`. See `agnt5-serverless`.

## Common mistake to avoid

Calling a `@function` directly (`await fn(ctx, ...)`) inside a workflow raises
`TypeError: Function 'fn' requires FunctionContext as first argument` — go through
`ctx.step()`. Plain coroutines that are *not* `@function`s do run when awaited directly, but
without a checkpoint they repeat on every replay: wrap anything that must not repeat
(charges, emails, external side effects) in `ctx.step("name", coro)`.

## Source

https://agnt5.com/docs/build/workflows · https://agnt5.com/docs/build/functions

Related skills: `agnt5-models` (direct `lm.generate` / `lm.stream`), `agnt5-client` (calling
runs from your app), `agnt5-testing`, `agnt5-sdk-pitfalls`.
